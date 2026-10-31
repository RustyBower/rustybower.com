---
title: "The Backup That Spent Four Days Uploading My Media Library"
date: 2026-12-01
draft: false
categories: ["kubernetes", "homelab", "gitops", "debugging"]
tags: ["velero", "argocd", "kopia", "prometheus"]
---

My Kubernetes backups failed in four different ways over about a month, and each fix uncovered the next one. The most expensive failure uploaded 8.2 TB over my home internet connection. The most worrying one reported success while leaving apps out of the backup.

The setup is Velero on a single-node cluster, running a nightly schedule at 03:00. It backs up to MinIO on my home NAS, which sits on the far side of a WAN link from the cluster. Everything Velero saves has to cross the internet.

## Failure one: no backup had ever existed

This command came back empty:

```
kubectl get backup -A
```

The schedule had a 720-hour TTL, so there should have been about 30 backups. There were none, along with no DataUploads and no PodVolumeBackups. The logs showed each night's run being cancelled about 30 seconds in:

```
03:00:50  Schedule creates velero-daily-critical-20260818030050
03:00:52  ArgoCD syncs the velero app with prune:true
03:01:19  "Cancel PVB under phase Accepted because it is being deleted"
03:01:24  "Async fs backup data path canceled"
```

ArgoCD was deleting every backup two seconds after it was created.

Velero copies a Schedule's labels onto each Backup it generates. My Schedule came from a Helm chart, so it carried `app.kubernetes.io/instance: velero`. That also happens to be how ArgoCD decides what it owns by default. ArgoCD saw a resource with its label that wasn't in Git, and pruning that kind of resource is its job.

The quick fix was a `resource.exclusions` entry in `argocd-cm` for the objects Velero creates at runtime, followed by a restart of the application controller. A scoped test backup then finished with `items=52/52 errors=0`.

The proper fix is `application.resourceTrackingMethod: annotation`, but that re-evaluates ownership of everything in the cluster with `prune: true` switched on, so I left it for its own change window. Less than three weeks later, an app named `rook-ceph` matched the label Rook puts on its operator. ArgoCD pruned the operator and took storage down across the cluster. I made the change that day.

## Failure two: backups that worked too well

The day after that fix, backups started running for real. Velero had `defaultVolumesToFsBackup: true`, so it backed up every volume mounted by every pod in the schedule. One of those pods was `beets`, the music tagger. Its `media` volume is an NFS mount of my whole media library.

That PodVolumeBackup went `InProgress` on August 19 and stayed there:

| Metric | Value |
|---|---|
| Time running | Over four days |
| Uploaded | 8.2 TB |
| Size Velero reported | 308 TB |
| Home WAN, inbound | About 180 Mbps, against under 10 normally |

At that speed, the full 308 TB would have taken about five months. Every later nightly run sat in `Prepared` behind it.

It went unnoticed for days, partly because the alerting metric was reassuring and wrong. `velero_backup_last_status` stayed at 1, meaning success, because the last backup to *finish* had succeeded. A backup that never finishes can't change that value. The metric that told the truth was `velero_backup_last_successful_timestamp`, which was 5.4 days stale by then.

It also muddied something else I was working on. I had an open ticket with my ISP about latency, and I took a "clean" baseline after a repair visit, right on top of this download. My own backup had quietly invalidated that baseline.

Cancelling it is one patch, and the queued runs then cleared in seconds:

```bash
kubectl -n velero patch podvolumebackup <name> \
  --type=merge -p '{"spec":{"cancel":true}}'
```

The fix was to make backups opt-in. `defaultVolumesToFsBackup` is now `false`, and each app names only the volumes worth saving:

```yaml
spec:
  template:
    metadata:
      annotations:
        backup.velero.io/backup-volumes: config-volume,postgres-data-volume
```

That came to 23 annotations across 16 apps. Media and downloads are never walked, because those can be replaced and they don't fit through the connection anyway.

New alerts cover what was missed: no successful backup in 36 hours, a run stuck for over 6 hours, and sustained inbound WAN traffic whatever the cause.

## Failure three: `Completed` didn't mean backed up

Two weeks later a run came back `PartiallyFailed`, with ten errors like this:

```
failed to list node-agent pods: client rate limiter Wait returned an error:
context deadline exceeded
```

That reads like API throttling, but all ten errors had the same timestamp, exactly four hours after the run began. That was the timeout expiring and surfacing in client-go.

node-agent handles one pod volume at a time per node by default, and this is a single-node cluster, so the whole schedule ran as one chain in roughly alphabetical order. That night it needed 4h47m against a 4-hour default for `--default-item-operation-timeout`.

The slow volumes weren't the big ones:

| Volume | Size | Time | Speed |
|---|---|---|---|
| Postgres data | 1.42 GB | 2h36m | About 155 KB/s |
| App config | 5.88 GB | 22m | About 4.6 MB/s |

Those slow volumes were limited by latency over the WAN rather than by bandwidth.

The dangerous part is what happens after the timeout. Velero stops waiting, but node-agent carries on, and the remaining PodVolumeBackups still finish as `Completed`. Those volumes aren't in the backup. Because the chain is alphabetical, the apps at the end go first: Sonarr, Radarr, Prowlarr and SABnzbd had no volume backup for three days, and each of those objects looked fine on its own.

I set `loadConcurrency` to 3 and raised the timeout to 8 hours. It's now worth checking coverage per namespace instead of trusting the backup's phase:

```bash
kubectl -n velero get podvolumebackups.velero.io \
  -o custom-columns='NS:.spec.pod.namespace,PHASE:.status.phase,BACKUP:.metadata.labels.velero\.io/backup-name'
```

## Failure four: copying a live database

The last failure came from backing up Postgres data directories while Postgres was running. Kopia hit `pgsql_tmp` files that Postgres deleted mid-copy, and the run went `PartiallyFailed`.

Copying a live data directory was never a great idea anyway. Each app now runs `pg_dumpall` as a Velero pre-hook into a memory-backed `emptyDir`, Velero backs up that one volume, and a post-hook removes the dump. The first app's dump took about 31 seconds and uploaded 239 MB in a little over two minutes, where its data directory had taken over two and a half hours.

## Where it ended up

The nightly run that took 4h47m now takes eight to ten minutes. As of late October, the last twelve runs had all ended `Completed` with no errors.

| Failure | What hid it |
|---|---|
| ArgoCD pruned every backup | The schedule looked fine, and nobody listed the backups |
| Media library upload | `last_status` still reported success |
| Timeout dropped late apps | Each volume backup said `Completed` |
| Live Postgres files | Crash-consistent copies usually work, until they don't |

Each of those was a status that was accurate about something other than what I cared about. What I wanted to know was whether I could restore last night's data, and nothing reported that.

## Caveats

- **I haven't tested a restore.** The cluster has no Restore objects, and restoring from the dumps has never been tried. Until that happens, all of this only shows that the data was uploaded.
- **Some nights can still be lost.** When the backup location fails validation at 03:00, the run is rejected outright and isn't retried that day. Five of ten runs were lost this way in late August and early September. That's a reachability problem between the cluster and the NAS, and I haven't fixed it. The recent clean runs don't prove it's gone.
- **The warnings are still there.** Each run logs 53 warnings. The ones I traced earlier were harmless noise from the snapshotter, but the count has grown since then and I haven't looked at each one.
- **The opt-in model can fail the other way.** A new app gets no backup until someone adds its annotation, and nothing alerts on that.
