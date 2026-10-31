---
title: "My Home Assistant Recorder Wasn't Killing My SSDs, Though I Committed That It Was"
date: 2026-12-15
draft: false
categories: ["home-automation", "homelab", "debugging"]
tags: ["home-assistant", "longhorn", "unpoller", "smart"]
---

A node in my home Kubernetes cluster died, I went looking for a reason, and I found a very convincing one. Home Assistant was generating a huge share of the cluster's disk writes. I wrote the fix, put the diagnosis in the commit message, and restarted Home Assistant.

Nearly everything in that commit was wrong except the fix, and the fix hadn't loaded.

## The convincing number

The cluster had already lost one node, put down at the time to a failed SSD, when a second one went unreachable. The boot drives are 256 GB SanDisk SATA SSDs rated for 80 TBW, and they aren't young. Per-process write counters on one node gave me an obvious suspect:

| Measure | Value |
|---|---|
| Writes to Longhorn over 23 days | About 990 GB |
| Share from the Home Assistant config volume | 323 GB |
| Rate | About 14 GB/day |
| After Longhorn's 3x replication | About 40 GB/day across the cluster |

So I opened the recorder database, 3.4 GB with 15.7 million rows, and grouped by entity:

| Entity pattern | Rows | Share |
|---|---|---|
| `sensor.*_rx` / `sensor.*_tx` | 10,903,246 | 69.4% |
| `*utilization*` | 762,880 | 4.9% |
| `device_tracker.*` | 566,847 | 3.6% |

Those first two rows come from unpoller, the UniFi exporter. It publishes a bandwidth counter for every client on the network, and the value never repeats, so every scrape writes a new row. Home Assistant was storing nearly 11 million rows of data I never looked at.

It's easy to measure on your own database. Use `GLOB` for the entity patterns, because `_` is a wildcard in `LIKE`:

```sql
SELECT m.entity_id, COUNT(*) AS n
FROM states s JOIN states_meta m USING (metadata_id)
WHERE m.entity_id GLOB 'sensor.*_[rt]x'
GROUP BY 1 ORDER BY n DESC LIMIT 20;
```

Excluding them costs nothing here, because the InfluxDB integration has no filter and already stores every series for the long term. The recorder only backs the History and Logbook views, plus short-term automations.

So I committed "stop the recorder wearing out the cluster's SSDs". I said the writes had worn out the boot drives and caused both node failures. I added the exclusions and raised `commit_interval` to 30, saying it was currently one second.

## Mistake one: I never asked the drives

The same day, I read SMART on the two surviving nodes:

| Node | Life left | NAND written | Reallocated | Unexpected power loss |
|---|---|---|---|---|
| A | 87% | 23.4 TB | 0 | 142 |
| B | 81% | 33.8 TB | 0 | 65 |

These drives had years of life left. My lifetime wear estimate came from `/sys/block/sda/stat`, using a 23-day window and an assumed nine years of service. Power-on hours said about 5.3 years, and the real NAND writes were 4-6 TB a year rather than the 11-29 I'd extrapolated. That's off by about four times.

The column I hadn't been looking for was the last one. Node A had 142 unexpected power losses. Boot logs showed the two nodes restarting within a second or two of each other on six occasions, all during service-call hours and separate from the staggered 03:00 reboots from kured.

The cause was Powerwall maintenance dropping the backed-up panel the cluster sits on. The drives weren't wearing out, the power was just going away.

If you want to judge SSD wear, read the drive's own counters for lifetime NAND writes and erase count, rather than the block layer's.

## Mistake two: the fix never loaded

While I was correcting that, the restart logged this:

```
Setup of package 'recorder' at packages/recorder.yaml, line 33 failed:
integration 'recorder' has duplicate key 'commit_interval'
```

I'd added `commit_interval` to a recorder package managed in Git. `configuration.yaml` on the volume already had a `recorder:` block with that key. Home Assistant's package merge treats a duplicate scalar as fatal, and it drops the *whole package* rather than the one key. So not one of the exclusions had applied.

Lists merge cleanly across the two files, so removing that one line made the exclusions work:

```yaml
recorder:
  exclude:
    entity_globs:
      - sensor.*_rx
      - sensor.*_tx
```

Home Assistant keeps running after this kind of failure, so unless you read the log, a dropped package looks just like a working one. Recorder settings also can't be reloaded, so every attempt costs a full restart.

## Mistake three: the value wasn't what I said

That duplicate key also showed what the existing value was: 5 seconds. I'd written that it was 1 second, and that raising it would cut writes by about 30 times. It would have been closer to six, and in the end I never made that change. It's still 5.

## What the fix was worth

| Metric | Before | Now |
|---|---|---|
| Recorder database | 3.4 GB | 1.6 GB |
| Retention | 10 days | 10 days |

The recorder was never why anything died. It was still worth fixing, because most of its rows carried no value and all of them were being replicated three times.

The commit that corrected the record is the one I'm happiest with. It's titled "correct the recorder rationale - SSD wear was wrong", and the config file now has a `CORRECTION` block saying the same. Otherwise the next person to read that file, probably me, would inherit the wrong story.

## Caveats

- **I only read SMART on two nodes.** Two others weren't reachable with passwordless sudo at the time, and I didn't read the two that failed. The power explanation fits the logs on the survivors, but it isn't proof of what killed those two.
- **The database comparison isn't controlled.** The size was measured about two months apart, with different devices and integrations in between. I don't have a clean write-rate figure from after the change.
- **The write figures are from one node.** The 990 GB comes from a single node's per-process counters over 23 days. The fleet-wide 40 GB/day is that figure scaled up by replication, not something I measured directly.
