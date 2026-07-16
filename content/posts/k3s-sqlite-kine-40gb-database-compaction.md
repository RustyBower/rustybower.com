---
title: "How K3s Silently Grew a 40 GB SQLite Database and Took Down My Cluster"
date: 2026-08-25
draft: false
tags: ["kubernetes", "k3s", "sqlite", "homelab", "debugging", "etcd", "kine"]
---

My cluster — a single-node K3s instance running about 60 ArgoCD-managed applications — had been slowly dying for months. Pods crash-looped with "context deadline exceeded" errors. Leader elections failed. The metrics server stopped working entirely. I assumed it was resource pressure and tuned probe timeouts. The real problem was underneath all of it: K3s's SQLite datastore had silently grown to 40 GB.

## The symptoms

Everything pointed at the API server. Operators that use leader election kept losing their leases:

```
Failed to update lock: Put "https://10.43.0.1:443/.../leases/my-operator":
  context deadline exceeded
failed to renew lease: timed out waiting for the condition
```

kube-state-metrics crash-looped because its liveness probe got HTTP 503 — it couldn't reach the API server's `/livez` endpoint. The metrics-server hadn't served data in over a month. And the API server health endpoints themselves confirmed it:

```
$ kubectl get --raw /readyz
[-]etcd failed: reason withheld
[-]etcd-readiness failed: reason withheld
readyz check failed
```

But there was no etcd. This is K3s.

## K3s and kine

K3s doesn't run etcd by default. On single-node installs, it uses SQLite through a translation layer called [kine](https://github.com/k3s-io/kine) that makes SQLite look like etcd to the Kubernetes API server. The health endpoint still says "etcd" because the API server doesn't know the difference.

The kine schema is simple. There's one table — `kine` — and every Kubernetes API write (creating a pod, updating a deployment, renewing a lease) inserts a new row. Old revisions are supposed to be compacted away automatically. On my cluster, they weren't.

## 7.2 million rows

```sql
SELECT COUNT(*) FROM kine;
-- 7223033
```

Over 7 million rows. The top offenders were all leader election leases:

| Key | Rows |
|-----|------|
| Operator A lease | 1,285,730 |
| Operator B lease | 1,049,397 |
| Deleted ingress controller leader | 448,295 |
| API server master lease | 375,136 |
| Node lease | 339,453 |
| cert-manager leases (x2) | ~236,000 each |
| Storage operator leases (x6) | ~139,000 each |

Each operator renews its lease every few seconds. Each renewal is an INSERT. With no working compaction, these rows accumulated over the entire lifetime of the cluster.

The database file:

```
-rw-r--r-- 1 root root 40G state.db
-rw-r--r-- 1 root root 22G state.db-wal
```

62 GB total. The k3s server process had ballooned to **49 GB of RAM** trying to manage it. The machine had enough RAM that it wasn't OOM-killed — it just made everything slow.

## The vicious cycle

This is the part that made it hard to diagnose. The crash-looping pods weren't just victims of the slow database — they were making it worse.

1. The database gets slow enough that a lease renewal takes >10 seconds
2. The operator's leader election times out (default deadline: 10 seconds)
3. The operator crashes and restarts
4. On startup, it immediately tries to acquire the lease — a burst of writes
5. Those writes make the database slower
6. Repeat from step 1

One operator was the worst case. It runs a heavy reconciliation on startup (~4 minutes of CPU-intensive work), during which it can't renew its lease. So it would start, begin reconciling, fail to renew, crash, restart, and reconcile again — each cycle adding dozens of rows to the database.

## The fix

The compaction had to happen with k3s stopped. Running containers keep operating when the API server is down — they just can't be scheduled or updated — so the impact is limited.

```bash
# Stop k3s
sudo systemctl stop k3s

# Back up first
sudo cp /var/lib/rancher/k3s/server/db/state.db \
  /var/lib/rancher/k3s/server/db/state.db.bak-$(date +%Y%m%d)

# Compact in a single sqlite3 session: checkpoint WAL, delete old
# revisions (keep only the latest per key), then VACUUM to reclaim space
sudo sqlite3 /var/lib/rancher/k3s/server/db/state.db << 'EOF'
PRAGMA wal_checkpoint(TRUNCATE);
DELETE FROM kine WHERE id NOT IN (SELECT MAX(id) FROM kine GROUP BY name);
VACUUM;
EOF

# Restart k3s
sudo systemctl start k3s
```

The DELETE took about 15 minutes on the 40 GB database. VACUUM took another 90 seconds. The result:

```
Before: 40 GB database, 22 GB WAL, 7.2M rows, 49 GB k3s RSS
After:  271 MB database, 134K rows, ~1 GB k3s RSS
```

A 99.3% reduction in database size.

## Why compaction broke

K3s's kine driver has a built-in compaction routine that should prevent this. Looking through the logs after the fix, I found zero compaction-related entries — it appears the compaction goroutine either wasn't running or was blocked by the same database contention it was supposed to prevent.

This is a known class of issue. The kine compaction runs as part of the k3s process, using the same SQLite connection pool. When the database is under heavy write load (hundreds of lease renewals per minute), the compaction queries compete with the writes and can time out or get starved. Once the database is large enough that compaction queries are slow, they'll never catch up.

## Preventing it from happening again

Beyond the database fix, I also addressed the pods that were generating the most write amplification:

**Relaxed leader election timeouts.** The worst-offending operator had a default lease duration of 15 seconds with a 10-second renew deadline. On a single-node cluster where there's no other replica to take over, aggressive leader election is pure overhead. I increased the lease duration to 120 seconds and the renew deadline to 90 seconds. Similar changes for other operators' liveness probes.

**Set resource requests.** kube-state-metrics was running as BestEffort (no resource requests), making it the first eviction candidate under memory pressure. Being evicted means restarting, which means more lease churn. Adding a 64Mi request moved it to Burstable QoS.

**Removed dead state.** A deleted ingress controller's leader lease still had 448K rows — the controller was removed months ago, but the lease key kept accumulating rows from other controllers trying to clean it up.

## Monitoring so it doesn't happen again

I also added Prometheus alerting for the database size. The tricky part: K3s holds the SQLite database open with an exclusive lock during heavy writes, so you can't always query the row count. But you can always `stat` the file.

I enabled the [textfile collector](https://github.com/prometheus/node_exporter#textfile-collector) on node-exporter and wrote a small cron script that runs every 5 minutes:

```bash
#!/bin/bash
# /usr/local/bin/k3s-db-metrics.sh
OUTPUT=/var/lib/node-exporter/textfile/k3s_kine.prom
DB=/var/lib/rancher/k3s/server/db/state.db

[ -f "$DB" ] || exit 0

DB_SIZE=$(stat -c %s "$DB" 2>/dev/null || echo 0)

WAL_SIZE=0
for f in "${DB}-wal" "${DB}-journal"; do
  [ -f "$f" ] && WAL_SIZE=$(($WAL_SIZE + $(stat -c %s "$f")))
done

# Row count times out gracefully if the DB is locked
ROW_COUNT=$(timeout 5 sqlite3 "$DB" "SELECT COUNT(*) FROM kine;" 2>/dev/null)
ROW_COUNT=${ROW_COUNT:--1}

cat > "${OUTPUT}.tmp" << EOF
# HELP k3s_kine_total_size_bytes Total size of K3s kine database including WAL/journal
# TYPE k3s_kine_total_size_bytes gauge
k3s_kine_total_size_bytes $(($DB_SIZE + $WAL_SIZE))
# HELP k3s_kine_row_count Number of rows in the kine table (-1 if locked)
# TYPE k3s_kine_row_count gauge
k3s_kine_row_count ${ROW_COUNT}
EOF
mv "${OUTPUT}.tmp" "$OUTPUT"
```

The key insight is that `stat` never blocks on SQLite locks, so the file size metric is always available. The row count uses a 5-second timeout and reports -1 if the database is busy — Prometheus can filter those out.

The alert rules fire on total size (database + WAL/journal combined):

```yaml
- alert: K3sKineDBSizeLarge
  expr: k3s_kine_total_size_bytes > 2e+09
  for: 1h
  labels:
    severity: warning

- alert: K3sKineDBSizeCritical
  expr: k3s_kine_total_size_bytes > 5e+09
  for: 30m
  labels:
    severity: critical

- alert: K3sKineRowCountHigh
  expr: k3s_kine_row_count > 500000
  for: 1h
  labels:
    severity: warning
```

2 GB is a generous threshold — my compacted database is 271 MB. If it ever hits 2 GB, something has gone wrong with compaction and I'll get a Pushover notification before it spirals.

If you're running K3s with SQLite (the default for single-node), check your database size. If it's over a few hundred megabytes, you might be on the same path.
