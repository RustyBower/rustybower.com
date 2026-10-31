---
title: "One SDR, Six Flight Trackers: Running an ADS-B Feeder on Kubernetes"
date: 2026-11-03
draft: false
categories: ["homelab", "kubernetes", "home-automation", "debugging"]
tags: ["ads-b", "readsb", "ultrafeeder", "flightradar24"]
---

Aircraft broadcast their position, altitude, and callsign on 1090 MHz, unencrypted, about twice a second. A cheap RTL-SDR dongle can hear them. I plugged one into a NUC, pointed the output at my Kubernetes cluster, and two weeks later that one dongle feeds six flight tracking networks and a set of Home Assistant sensors.

It also turned out to be losing about 7% of everything it heard over USB, for a reason I didn't expect. More on that below, because it's the most interesting thing I found.

## The layout

Only one box touches the radio. The NUC runs [readsb](https://github.com/wiedehopf/readsb) against the SDR and exposes the decoded messages as a Beast stream on port 30005. Everything else lives in the cluster and never sees the hardware:

```
SDR -> NUC (readsb) -> Beast :30005 -> Ultrafeeder (cluster)
                                         |-> adsb.lol, adsb.fi, airplanes.live, ADS-B Exchange
                                         |-> piaware (FlightAware)
                                         |-> fr24feed (Flightradar24)
                                         |-> adsb2mqtt -> Home Assistant
```

[Ultrafeeder](https://github.com/sdr-enthusiasts/docker-adsb-ultrafeeder) runs with `--net-only`, so it never looks for a local SDR. It takes the stream from the NUC and serves the tar1090 map and graphs1090 charts. It also acts as the hub for the feeders, which keeps a single link to the NUC no matter how many networks are consuming it.

Splitting it this way has one big payoff: moving the radio doesn't touch the cluster. As long as the NUC keeps its address, the antenna can go anywhere.

The open aggregators are all one environment variable:

```yaml
- name: ULTRAFEEDER_CONFIG
  value: >-
    adsb,<nuc-ip>,30005,beast_in;
    adsb,feed.adsb.lol,30004,beast_reduce_plus_out;mlat,feed.adsb.lol,31090,39001;
    adsb,feed.adsb.fi,30004,beast_reduce_plus_out;mlat,feed.adsb.fi,31090,39000;
    adsb,feed.airplanes.live,30004,beast_reduce_plus_out;mlat,feed.airplanes.live,31090,39002;
    adsb,feed.adsbexchange.com,30004,beast_reduce_plus_out;mlat,feed.adsbexchange.com,31090,39003
```

FlightAware and Flightradar24 each get their own Deployment, pulling Beast from the in-cluster Ultrafeeder Service. Their keys are Sealed Secrets, and the receiver location is one ConfigMap shared by everything that needs it. Feeding these networks means handing them your coordinates, and MLAT wants them accurate to about 25 metres.

## Three small mistakes

Most of the first day went to small configuration problems.

**MLAT was off because of one variable name.** I had set `ULTRAFEEDER_UUID`. The variable is `UUID`. `mlat-client` won't start without a `UUID` or `MLAT_USER`, and the symptom shows up somewhere else: connection refused errors from the MLAT hub on ports 39000-39003.

**FlightAware needed a pinned feeder ID.** Until the claimed ID is persisted, a pod restart can come back as a new feeder. It's now a Sealed Secret.

**Flightradar24 rejected my feed for being too old, and the obvious image tag couldn't fix it.** The first image bundled fr24feed 1.0.37-0, which was refused. The newest semver tag bundled 1.0.48-0, which is also refused, because FR24 now wants 1.0.52-0 or later. The semver tags had simply stopped being updated. The `latest-build-NNN` tags are current, and they still have a build number for Renovate to track.

## The DNS failure that looked like a stuck process

Two days in, the FR24 feed went offline after some node reboots. Every connection attempt failed the same way:

```
Could not connect feed! (result -1, fd -1)
```

That included connections to the in-cluster Ultrafeeder, one hop away. A shell in the same pod resolved every name perfectly, which made it look like a wedged process rather than a network problem.

The clue was in `/etc/resolv.conf`. The rebooted nodes had picked up Tailscale's MagicDNS in their host resolver config, and kubelet copies the host's search domains into every pod:

```
search adsb.svc.cluster.local svc.cluster.local cluster.local <tailnet>.ts.net <home-domain>
nameserver 10.43.0.10
options ndots:5
```

The one node that hadn't rebooted didn't have the extra domain, which is what gave it away. fr24feed is a static binary with its own resolver rather than glibc's, and it couldn't cope with that search list. The shell tools used glibc, so they were fine.

The fix was to stop it expanding search domains for dotted names at all:

```yaml
dnsConfig:
  options:
    - name: ndots
      value: "1"
```

This only works here because `adsb-ultrafeeder.adsb.svc.cluster.local` is fully qualified. A bare service name wouldn't resolve.

## Getting it into Home Assistant

`adsb2mqtt` is a small Python Deployment that polls tar1090's `aircraft.json` every 20 seconds, works out range and bearing from the receiver, and publishes sensors through MQTT discovery:

- Aircraft Count
- Aircraft With Position
- Nearest Aircraft, with distance and altitude as attributes
- Nearest Distance
- Max Range

Unlike the CronJob-style bridges I run for slower data, aircraft data is live, so this stays running.

## The baseline

These numbers are from the first two weeks. Message counts come from the NUC, and range comes from Ultrafeeder, which has the receiver location.

| Metric | Value |
|---|---|
| Messages decoded | 18,478,734 over 14.1 days (about 15/s) |
| Positions | 2,858,784 |
| Tracks | 27,038 |
| Most aircraft at once | 22 |
| Best daily range | 160-204 km |
| Longest position seen | 218.5 km |
| Gain | 59 dB |
| Noise floor | -39 dBFS |

Theory says it should do better. A cruising aircraft at 11,000 m is above the radio horizon out to about 440 km, and I'm reaching about half of that. Height isn't the reason. Raising the antenna from 5 m to 9 m moves that horizon by 3 km. What matters is how low in the sky the distant aircraft are:

| Distance | Elevation above the horizon |
|---|---|
| 150 km | 3.7° |
| 218 km | 2.2° |
| 300 km | 1.1° |
| 400 km | 0.2° |

Everything at the edge of my range sits within a few degrees of the horizon, right where walls and neighbouring houses are.

So I bought a FlightAware 1090 MHz antenna to replace the stock one that came with the SDR, and the next step is getting it into the attic for an "unfettered" view of the sky. Unfettered apart from the roof, anyway. The range history has no step change in it, so don't read the table as a before-and-after for the new antenna. It's the baseline the attic has to beat.

## The USB problem

While pulling those numbers, I noticed something wrong in the stats readsb publishes:

```json
"estimated_ppm": -43118.0,
"local": {
  "samples_processed": 2718659575808,
  "samples_dropped": 0,
  "samples_lost": 101864374272
}
```

RTL-SDR oscillators are cheap, but they aren't 43,000 ppm wrong. That number says the dongle is delivering about 4.3% fewer samples than the wall clock expects. `samples_lost` agrees on the direction: 3.6% over two weeks, and 4.4% in the five-minute window I looked at. That counter is an undercount, though. Comparing `samples_processed` with what 2.4 million samples a second should have produced, readsb received about 93% of the expected samples, so the real loss over those two weeks is closer to 7%.

readsb's source is clear about what this means. When the measured drift goes past -1000 ppm, it logs:

```c
fprintf(stderr, "Lost %d packets (%.1f us) on USB, MLAT could be UNSTABLE, check sync! (ppm: %.0f)"
        "(or the system clock jumped for some reason)\n", packets_lost, diff_us, ppm);
```

The log on the NUC has 37,243 of those warnings. The first one appeared 73 seconds after readsb first started, and they have continued ever since, at anywhere from 14 to 88 lost packets per warning depending on the hour. A clock jump doesn't fit a loss that never stops. `samples_dropped` also rules out the other usual suspect. That counter goes up when the decoder falls behind and the buffer fills, and it's exactly zero, so the NUC's CPU is keeping up. The samples are vanishing before readsb ever sees them.

Losing that much of the airwaves is annoying but survivable. The worse damage is probably to MLAT. Timestamps come from counting samples, so when USB loses a block the count silently falls behind real time, and multilateration depends on those timestamps agreeing with other receivers to within a very small margin. On that same morning, the FlightAware client reported this:

```
Server status:   clock unstable
Aircraft: 1 of 1 Mode S, 5 of 5 ADS-B used
```

The aggregator clients were producing less than a position a minute, and there wasn't a single MLAT-derived aircraft on the map.

The CPU was nearly idle, which turned out to be the clue. The NUC spent most of its time in C10, its deepest sleep state, which takes 890 µs to wake from. So I turned off everything deeper than C1E for five minutes, then turned it back on:

| Five-minute window | Warnings | Packets lost |
|---|---|---|
| Before | 10 | 197 |
| Deep C-states off | 1 | 11 |
| Restored | 10 | 654 |

A box that's too idle was starving the radio. The permanent fix is one line in `/etc/tmpfiles.d/`, which needs no reboot and reapplies at boot:

```
w /sys/devices/system/cpu/cpu*/cpuidle/state[3-8]/disable - - - - 1
```

Seven minutes later, the last five-minute window showed zero lost samples, 100% of the expected sample rate, and an `estimated_ppm` of -0.9. The dongle's oscillator was never the problem.

MLAT took a little longer to come back. The FlightAware client still said `clock unstable` 11 minutes after the fix, and 26 minutes after it, the status had changed:

```
Server status:   synchronized with 102 nearby receivers
```

The aggregator clients went from less than a position a minute to as many as 23.

That cleared the way for the attic. A radio that loses data to the USB bus wouldn't have improved with a better view, and the baseline above was measured with the problem in place, so the attic now has to beat numbers from a receiver that was missing about 7% of its samples.

## Caveats

- **The USB result is short-term.** Minutes of clean data on a single morning is strong evidence, not a long-term measurement, and limiting C-states costs some idle power. It also doesn't explain the occasional control errors from the dongle (`i2c wr failed=-9`). The kernel log shows no USB resets or disconnects.
- **The MLAT recovery is one data point.** The timing fits the fix, but MLAT also depends on traffic and nearby peers, and that morning was quiet. It needs a longer look before I call it solved.
- **The elevation figures are estimates.** They use a 4/3-earth model and assume an aircraft at 11,000 m.
- **The DNS fix treats the symptom.** The Tailscale search domain is still in every pod. `tailscale set --accept-dns=false` on the nodes is the real fix, and anything else with a static resolver is exposed until then.
