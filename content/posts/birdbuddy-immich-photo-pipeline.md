---
title: "Automating a Smart Bird Feeder's Photo Archive with Kubernetes and Immich"
date: 2026-07-16
draft: false
tags: ["kubernetes", "immich", "birdbuddy", "python", "homelab", "cronjob", "photography"]
---

My [Bird Buddy](https://mybirdbuddy.com/) smart feeder takes a photo every time a bird visits. Over a few months, that adds up to thousands of images — all sitting in Bird Buddy's cloud app with no good way to search, organize, or back them up. I wanted them in [Immich](https://immich.app/), my self-hosted photo library, organized by species and properly timestamped.

The Bird Buddy app doesn't have an export feature. But the `pybirdbuddy` Python library can authenticate to their API and walk the feed. So I built a two-stage pipeline: a CronJob that downloads new photos daily, and a second CronJob that syncs them into Immich. Both run in Kubernetes, and the whole thing is about 300 lines of Python.

## The timestamp problem

Bird Buddy images arrive as plain JPEGs with no EXIF metadata. If you drop them into Immich as-is, every image gets dated to when *the CronJob ran*, not when *the bird visited*. A cardinal at sunrise and a chickadee at sunset both show up as "3:00 AM" because that's when `download.py` ran.

The Bird Buddy API does return a `createdAt` timestamp for each postcard. The fix is to write that timestamp into the JPEG's EXIF data before Immich ever sees the file:

```python
def apply_timestamp(filepath, dt, is_video):
    """Embed capture time so Immich dates the asset correctly."""
    local = dt.astimezone(LOCAL_TZ)

    if not is_video:
        stamp = local.strftime("%Y:%m:%d %H:%M:%S").encode()
        raw_off = local.strftime("%z")
        offset = (raw_off[:3] + ":" + raw_off[3:]).encode() if raw_off else None
        exif = piexif.load(str(filepath))
        exif["Exif"][piexif.ExifIFD.DateTimeOriginal] = stamp
        exif["Exif"][piexif.ExifIFD.DateTimeDigitized] = stamp
        if offset:
            exif["Exif"][piexif.ExifIFD.OffsetTimeOriginal] = offset
        piexif.insert(piexif.dump(exif), str(filepath))

    # Also set file mtime — Immich's fallback when EXIF is absent (videos)
    ts = dt.timestamp()
    os.utime(filepath, (ts, ts))
```

Immich reads `DateTimeOriginal` first, which is the standard EXIF field for "when was this photo actually taken." For videos (MP4s from the feeder's motion clips), EXIF doesn't apply, so we fall back to setting the file's modification time — Immich uses that as a last resort.

The timezone offset matters too. Without it, Immich has to guess whether "2026-07-04 08:32:15" is UTC, local time, or something else. Writing the explicit offset (`-05:00` for Central) removes the ambiguity.

## Stage 1: Download

The downloader runs daily at 3 AM as a Kubernetes CronJob. It authenticates to Bird Buddy, walks the feed for the last 24 hours, and downloads any new media into species-organized directories on an NFS share:

```
/photos/birdbuddy/
├── American_Robin/
│   ├── abc123.jpg
│   └── def456.mp4
├── Black-capped_Chickadee/
│   ├── ghi789.jpg
│   └── ...
├── Northern_Cardinal/
│   └── ...
└── .downloaded.json    ← state file for deduplication
```

Deduplication is handled by a simple JSON state file that tracks every media ID we've already downloaded. The state file lives on the same NFS share as the photos, so it survives pod restarts.

The CronJob itself uses a stock `python:3.12-slim` image with pip-installed dependencies — no custom Docker image to maintain:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: birdbuddy-downloader
spec:
  schedule: "0 3 * * *"
  timeZone: "America/Chicago"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: downloader
              image: python:3.12-slim
              command:
                - /bin/sh
                - -c
                - |
                  pip install --quiet pybirdbuddy aiohttp piexif tzdata && \
                  python /scripts/download.py
              envFrom:
                - secretRef:
                    name: birdbuddy-credentials
              env:
                - name: OUTPUT_DIR
                  value: /data/birdbuddy
              volumeMounts:
                - name: scripts
                  mountPath: /scripts
                - name: data
                  mountPath: /data/birdbuddy
          volumes:
            - name: scripts
              configMap:
                name: birdbuddy-downloader-scripts
            - name: data
              nfs:
                server: 192.168.1.10
                path: /mnt/storage/media/photos/birdbuddy
```

The Python script is delivered as a Kustomize-generated ConfigMap — the same pattern I use for [AppDaemon automations](/posts/unit-tested-appdaemon-automations-kubernetes/). No Dockerfile, no container registry, no build pipeline. Change the Python, push to Git, ArgoCD syncs the ConfigMap.

The downloader supports two modes: an incremental daily sync (last 24 hours of the feed) and a full backfill that walks all collections. The initial backfill pulled about 2,000 images. Daily runs typically download 5-20 new photos.

## Stage 2: Sync to Immich

Thirty minutes after the downloader finishes, a second CronJob reconciles the NFS directory with Immich:

1. **Trigger a library scan.** Immich has an "external library" feature that watches a filesystem path. The sync script pokes the API to start a scan and waits 60 seconds for it to index new files.

2. **Ensure a "Bird Buddy" album exists.** All bird photos get collected into a single album for easy browsing.

3. **Add missing assets to the album.** Pages through every asset Immich imported from the BirdBuddy path and adds any that aren't already in the album.

4. **Archive everything.** Sets each asset's visibility to `archive`, which removes it from the main Photos timeline while keeping it visible in the album and Archive views.

The archiving step is the key UX decision. Without it, my family's photo timeline would be 80% bird pictures. Archiving keeps them organized and searchable without drowning out the actual family photos.

```python
def archive_assets(asset_ids):
    for batch in chunked(asset_ids):
        api("PUT", "/api/assets", {"ids": batch, "visibility": "archive"})
```

The sync script uses only Python's standard library — no `requests`, no third-party HTTP client. `urllib.request` is ugly but it means the CronJob doesn't need a `pip install` step at all.

## Secrets management

Bird Buddy credentials and the Immich API key are stored as Sealed Secrets:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: birdbuddy-credentials
  namespace: birdbuddy-downloader
spec:
  encryptedData:
    BIRDBUDDY_EMAIL: AgBTcv5LFLpb610j...
    BIRDBUDDY_PASSWORD: AgBipbrCt5fI3I8V...
```

The download script reads them as environment variables via `envFrom`. Nothing sensitive touches Git in plaintext.

## What I learned

**EXIF timestamps are non-negotiable for photo imports.** Without them, bulk imports are useless — every photo shows up on the same date. This applies to any pipeline that downloads photos from an API and imports them into a photo manager. The few lines of `piexif` code saved hours of manual re-dating.

**Two CronJobs are better than one.** Separating download from sync means I can re-run either independently. If Immich has a hiccup, I re-run the sync without re-downloading everything. If Bird Buddy's API is down, the next day's sync picks up whatever was already downloaded.

**NFS as the integration layer.** The downloader writes to NFS. Immich's external library reads from NFS. The sync script talks to Immich's API. No service-to-service coupling, no message queues, no shared databases. The filesystem is the contract.

The pipeline has been running unattended for several months now. Every morning, any new bird visitors appear in the Bird Buddy album, correctly dated and organized by species. The total infrastructure cost is two CronJobs and an NFS share that was already there.
