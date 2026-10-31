---
title: "Running, Port Bound, Not Answering: When Plex Wedges on Its Own Database"
date: 2027-01-12
draft: false
categories: ["kubernetes", "homelab", "debugging"]
tags: ["plex", "sqlite", "liveness-probes"]
---

Plex has gone down on me twice in a way Kubernetes couldn't see. The pod said `1/1 Running`, with no restarts and no events, and the port was still bound. It just wasn't serving anyone. The second time, it stayed like that for about ten hours.

## Two versions of the same hang

The first one was on September 6. Plex accepted connections and never replied, even from inside its own pod:

```
curl localhost:32400/identity    # hangs until it times out
```

`/proc/net/tcp6` showed `[::]:32400` still listening, so the process was alive and the socket was open. From the outside it looked like a network problem, but the network was fine.

The answer was the last line of Plex's own log, in `Library/Application Support/Plex Media Server/Logs/`:

```
ERROR - Waited over 10 seconds for a busy database; giving up.
ERROR - Thread: Uncaught exception running async task ... std::exception
```

Plex keeps its library in SQLite, which allows one writer at a time. A bulk chapter-thumbnail job was holding the lock while a transcode was running, and Plex's request threads backed up behind it.

The second one, on September 25, looked different. `/identity` returned an instant 503 rather than timing out, and the tail of the log was only half-hourly connectivity warnings, so reading the last lines told me nothing. Searching for `busy database` found one hit at 23:48, and request logging had stopped a few minutes after that.

There were also `protocol is shutdown (SSL routines)` stream aborts before the first hang, and it would be easy to blame those. They're clients giving up, which is a symptom.

## Why Kubernetes didn't notice

This one was my fault. The Plex Deployment had no probes at all, so to Kubernetes a running process was a healthy process. A rollout restart fixed both incidents in about a minute, because the Deployment uses `Recreate` and the config volume hands over cleanly. I'd just been the only monitoring.

The fix is 16 lines:

```yaml
startupProbe:
  httpGet:
    path: /identity
    port: 32400
  periodSeconds: 10
  failureThreshold: 90
livenessProbe:
  httpGet:
    path: /identity
    port: 32400
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 3
```

An HTTP probe catches both versions of the hang: a 503 counts as a failure, and so does a timeout. `/identity` suits this because Plex serves it without authentication and it needs a working server to answer.

The startup probe is what makes the liveness probe safe. Plex runs database migrations after an upgrade and answers 503 while it does, and my pod installs a fresh Plex build whenever it restarts. Without 15 minutes of startup allowance, a slow migration could be killed partway through by the probe meant to protect it. That's a much worse failure than the one I'm fixing.

With these settings, a wedged Plex gets restarted after about 90 seconds rather than ten hours.

## The other way to misread Plex

The same month I had a separate complaint that looked similar: a remote client stuttering above 720p. It's worth including because the server had a good answer, and the obvious reading of it was backwards.

Ask Plex what it's doing rather than looking at the pod's CPU:

```
/status/sessions       # per-client bandwidth, lan or wan
/transcode/sessions    # throttled, speed, transcodeHwFullPipeline
```

| Reading | What it means |
|---|---|
| `throttled="1"`, `speed="0"` | Plex filled its lookahead and stopped, so it's ahead of the client and isn't the bottleneck |
| `throttled="0"`, `speed` under 1.0 | The server can't keep up |

The word "throttled" sounds bad, but here it's the healthy state. A transcode that has stopped at `speed="0"` has already done the work.

What settled it was comparing clients at the same moment. While one client stuttered, another was pulling 39 Mbps from the same server with the GPU at 0%. That rules out the server for everyone. Server-side data can't tell a client's ISP from its Wi-Fi, but it can get you out of the argument.

## Caveats

- **The probe treats the symptom.** It shortens the outage and leaves the cause alone. The durable fix is to stop bulk chapter-thumbnail generation, which is a setting in `Preferences.xml` on the volume, not in my repo. It's still set to `scheduled`, so this can happen again.
- **The thumbnail explanation comes from the logs of these incidents.** The job and the transcode were both active, and the lock errors fit. I didn't reproduce it on demand.
- **I haven't confirmed the probe saving a real wedge.** When I checked in late October, the current Plex log had no `busy database` lines. The pod had restarted twice, but on the same day as the weekly restart job, and I can't attribute those. Kubernetes events expire too quickly to tell.
- **A restart during a transcode is still a visible interruption.** Ninety seconds of failed checks plus Plex's start time is better than hours, but anyone watching will notice.
