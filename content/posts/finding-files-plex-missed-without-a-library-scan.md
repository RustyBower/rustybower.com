---
title: "Finding the Files Plex Missed Without Scanning the Library"
date: 2026-11-17
draft: false
categories: ["homelab", "python", "kubernetes", "debugging"]
tags: ["plex", "radarr", "sonarr", "cronjob"]
---

Every so often, Radarr would say a movie was on disk and Plex wouldn't show it. The fix was always the same: paste a folder-scoped refresh URL into a browser, and the movie would appear.

```
https://<plex>/library/sections/<id>/refresh?path=/data/movies/Some%20Movie%20(2021)/&X-Plex-Token=<token>
```

That works fine for the one movie you happened to notice. It tells you nothing about the ones you didn't. So I built a nightly job to find them all, and the funny part is that the movie that started it wasn't one of them.

## The false alarm

Plex had that movie all along. It was matched, the file was readable, and it had been in the library for nearly two years. A search on the server found it straight away, so whatever I saw was a client or profile problem rather than a missing scan.

The check I ran to prove that found the real problem:

| Library | Files Plex was missing |
|---|---|
| Movies | 14 of 7,703 |
| TV | 217 episodes across 8 shows |

Radarr and Sonarr already call a script hook that asks Plex to scan each import, so most files arrive normally. Some of them just don't, and nothing reports it.

Most of the misses were upgrades. When Sonarr replaces a file with a better copy, the new one has a different name, and if that scan gets lost, Plex is left pointing at a filename that no longer exists. That's why this job compares paths rather than titles: a title match would have said everything was fine.

## Why not just scan?

A full library scan is the obvious fix, and on a library this size it's the wrong one. It walks over 100,000 files on network storage to find a handful of misses. I've also had Plex wedge itself on a busy database before, so I'd rather not hand it extra work every night.

All the answers are already in three databases, so the job never needs to touch the disks.

## The delta

It's a set difference on file paths:

1. **What should exist.** Radarr's `/api/v3/movie`, plus Sonarr's `/api/v3/episodefile` for each series that has files.
2. **What Plex has.** Every `Part` file under `/library/sections/<id>/all` for movies and `allLeaves` for TV.
3. **The gap.** Anything in the first set that isn't in the second.

Radarr, Sonarr and Plex all see the same `/data` mount, so the paths compare directly with no remapping.

```python
missing = [p for p in expected
           if p not in indexed
           and os.path.splitext(p)[1].lower() not in SKIP_EXTENSIONS]
gaps = {}
for p in missing:
    gaps.setdefault(expected[p], []).append(p)
```

Missing files are grouped by movie or series folder, and each folder gets one scoped refresh, with a five-second pause between them.

It's a single stdlib Python script in a ConfigMap, run by a CronJob on `python:3.15-slim` at 04:00, an hour after the backup window. A typical run:

| Metric | Value |
|---|---|
| Arr files checked | 104,490 |
| Files in Plex | 112,513 |
| Runtime | About 4 minutes |
| Peak memory | About 335 MB |

Plex has more files than the arrs because not everything in it came from them. That's fine, because only one direction of the difference matters here.

## The details that matter

**The TV listing is too big to parse in one go.** `allLeaves` for this library is hundreds of megabytes of XML, so the script streams it and throws elements away as it goes:

```python
for _, el in ET.iterparse(resp):
    if el.tag == "Part" and el.get("file"):
        files.add(norm(el.get("file")))
    elif el.tag in ("Video", "Directory"):
        el.clear()
```

**Paths need Unicode normalising.** The same accented filename can be stored as different byte sequences, and a set difference treats those as different files. Everything goes through NFC first.

**A big gap means something else is wrong.** If a mount moves or the Plex database gets swapped, thousands of files look missing at once, and the worst response is to fire thousands of refreshes at the disks. Past 25 folders, the job reports and refreshes nothing:

```python
if len(gaps) > MAX_REFRESH:
    lines.append(f"Gap exceeds MAX_REFRESH={MAX_REFRESH}; refreshing nothing. "
                 "Check mounts / Plex before raising it.")
```

**Some gaps can't be fixed by refreshing.** Formats Plex won't index, like `.mk3d`, are skipped so they don't repeat every night. 0-byte files are truncated downloads that the arrs still count as present, so those get listed as re-download candidates instead. The first run found four.

Every run posts a summary to Discord, including when nothing is wrong, so a quiet channel means the job broke rather than the library being healthy.

## The bug that waited for a real gap

Four days after I built it, the job crashed:

```
File "/scripts/plex_gap_refresh.py", line 137, in main
    plex(f"/library/sections/{key}/refresh", path=folder + "/").read()
TypeError: plex() got multiple values for argument 'path'
```

The helper looked like this:

```python
def plex(path, **params):
```

The URL path is an argument called `path`, and Plex's query parameter is also called `path`. The very first time the job had a folder to refresh, it passed both.

The test runs had both missed it, for different reasons. The dry run skipped the refresh line on purpose. The first live run had nothing to refresh, because I'd already cleared the backlog by hand. Everything around that line had been exercised, and that one line had never run.

The fix is one character, which makes `path` positional-only so `**params` can hold a key of the same name:

```python
def plex(path, /, **params):
```

A manual run then found four missing files across four folders and refreshed them. A dry run afterwards reported:

```
Plex gap check: 104,490 arr files, 112,513 in Plex, 0 missing across 0 folders
```

## Caveats

- **It treats the symptom.** I don't know why Plex occasionally drops a scan, only that it does. The job finds the misses after the fact, up to a day later.
- **Only arr-managed files are covered.** Anything in Plex that didn't come from Radarr or Sonarr is outside the check.
- **One direction only.** The job won't notice files Plex lists that have since been deleted.
- **The numbers are early.** This has been running for days, not months, so I can't yet say how many files it catches in a normal week.
