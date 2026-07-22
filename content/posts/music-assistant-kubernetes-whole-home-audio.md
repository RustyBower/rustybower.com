---
title: "Whole-Home Audio with Music Assistant: Kubernetes Gotchas and a Silent Raspberry Pi"
date: 2026-08-04
draft: false
tags: ["music-assistant", "kubernetes", "home-assistant", "homelab", "raspberry-pi", "picoreplayer", "audio", "subsonic"]
---

My music setup had accreted into a mess. Across two Kubernetes clusters I was running Navidrome, Lyrion Music Server (LMS), beets, and my own AI radio station (SUB/WAVE) - plus a Raspberry Pi running piCorePlayer wired into a Dayton DAX66 matrix amp for whole-home audio. Four things that all "played music," none of which talked to each other, and no single place to control any of it from Home Assistant.

I wanted one hub. This is the story of consolidating all of it onto [Music Assistant](https://music-assistant.io/), and every gotcha that fought me on the way there - most of which come down to the same theme: **software that assumes it can see the network the same way its clients do, running inside Kubernetes, where it can't.**

## What I had, and what I wanted

The pieces, and what each actually does:

| Component | Role |
|---|---|
| **beets** | Library tagging/ingest - writes clean files to shared storage |
| **Navidrome** | Subsonic API + web/phone player; also the library backend for my radio station |
| **Lyrion (LMS)** | SlimProto server for the piCorePlayer / Squeezebox client |
| **SUB/WAVE** | AI internet-radio station (liquidsoap → Icecast), pulls tracks from Navidrome |
| **piCorePlayer** | Raspberry Pi player wired into the DAX66's line input |

The trap is thinking Navidrome and LMS are redundant - "two music servers reading the same library." They're not. They speak different protocols to different clients: Navidrome is Subsonic (web/phone apps), LMS is SlimProto (the Squeezebox hardware protocol the Pi uses). The real redundancy was that **LMS's only job was to drive the Pi**, and Music Assistant can do that itself - it has a built-in SlimProto server.

So the target:

```
Navidrome (library, Subsonic API)
      │
      ▼
Music Assistant  ──SlimProto──►  piCorePlayer  ──►  DAX66  ──►  whole-home audio
   (the hub: browse, queue,          (the player)      (the amp/matrix)
    transport, HA integration)
```

Music Assistant becomes the brain - it reads Navidrome over Subsonic, drives the Pi over SlimProto, exposes every zone to Home Assistant as a `media_player` entity, and can throw SUB/WAVE or Spotify or a radio stream at the same speakers. Navidrome stays (SUB/WAVE needs it, and it's MA's library). LMS gets retired.

Simple plan. Then I turned it on.

## Gotcha #1: Authelia breaks the Subsonic API

First step: point Music Assistant's Subsonic provider at Navidrome. Instant failure:

```text
Attempt to decode JSON with unexpected mimetype: text/html; charset=utf-8
url='https://auth.rustybower.com/?rd=https://navidrome.rustybower.com/rest/ping.view'
```

Navidrome sits behind [Authelia](https://www.authelia.com/) forward-auth via a Traefik middleware. The Subsonic client hit `/rest/ping.view`, Authelia bounced it to an HTML login page, and the client choked trying to parse HTML as JSON.

This isn't Music-Assistant-specific - **any** Subsonic client (phone apps included) hits the same wall, because API clients can't complete an interactive browser login. The Subsonic API authenticates itself with its own credentials; putting forward-auth in front of it is a category error.

The fix is to let the API path bypass Authelia while keeping the web UI protected. Traefik applies middleware per-router, and its default router priority is by rule length, so a second Ingress matching the more-specific `/rest` prefix - with no auth middleware - wins for API traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: navidrome-subsonic
  # No Authelia here: the Subsonic API (/rest/*) uses its own credentials.
spec:
  tls:
    - hosts: [navidrome.rustybower.com]
      secretName: navidrome-tls
  rules:
    - host: navidrome.rustybower.com
      http:
        paths:
          - path: /rest
            pathType: Prefix
            backend:
              service: {name: navidrome, port: {number: 4533}}
  ingressClassName: traefik
```

Verify from a client's perspective - `/rest` should return a Subsonic error (not an auth redirect), while `/` still bounces to Authelia:

```bash
$ curl -s https://navidrome.rustybower.com/rest/ping.view
<subsonic-response status="failed" ...><error code="10" message="missing parameter: 'u'"/></subsonic-response>

$ curl -sI https://navidrome.rustybower.com/ | grep location
location: https://auth.rustybower.com/?rd=...
```

The "missing parameter" error is exactly what you want: the API is reachable and now only needs Subsonic credentials. Library imported.

## Gotcha #2: Music Assistant advertises its pod IP (the big one)

Now the player. I added the Squeezelite provider in MA, pointed piCorePlayer at MA's LoadBalancer IP, and it connected. MA showed the track "playing." No sound.

This one ate hours, because everything *looked* right. The player was connected, the control link was up, Home Assistant even flapped the player between "available" and "unavailable" every couple minutes, which sent me down a rabbit hole about SlimProto connection stability and MetalLB `externalTrafficPolicy` (a red herring - more below).

The answer was in the player's own debug log. SlimProto is two channels: a **control** connection, and a separate **HTTP audio stream** the player fetches when told to play. When I hit play:

```text
process_strm: strm command s          # START - I pressed play
connect_socket: connecting to 10.42.4.197:8097
```

`10.42.4.197` is Music Assistant's **Kubernetes pod IP**. It's handing the player an address that only exists inside the cluster's pod network - completely unroutable from a Raspberry Pi on the LAN. The control link worked (the Pi reached MA's LoadBalancer IP fine), but the moment it tried to pull audio, it was sent into the void.

MA detects "its" IP from the network interface it's running on. In a pod, that's the pod IP. Its own docs recommend host networking for exactly this reason - but there's a cleaner fix that keeps the LoadBalancer intact. Buried in **Settings → Streams** is a **Published IP address** field, and it was set to the pod IP. Change it to the LoadBalancer IP:

- **Published IP address:** `172.30.0.5`  *(the MetalLB IP - what MA advertises)*
- **TCP Port:** `8097`  *(reachable through the LoadBalancer)*
- **Bind to IP/interface:** `0.0.0.0`  *(leave it - the pod can't bind an IP it doesn't own)*

The subtlety that trips people: **published** and **bind** are different. Published is the address MA hands to clients; it must be the reachable LoadBalancer IP. Bind is where MA actually listens inside the pod; it must stay `0.0.0.0`, because `172.30.0.5` lives on MetalLB, not on the pod's interface. Set bind to the LoadBalancer IP and the server won't start.

One more trap: use the raw LoadBalancer IP, **not** the ingress hostname. `musicassistant.bowerha.us` goes through Traefik on ports 80/443, which doesn't expose the stream port 8097. The LoadBalancer IP does.

After the change, the player finally opened a stream socket to the right place:

```text
tcp  367537  0  10.0.10.198:34266  →  172.30.0.5:8097  ESTABLISHED
```

368 KB buffered and flowing. And still... quiet. Because I'd been fighting a *second* problem the whole time.

## Gotcha #3: the Raspberry Pi headphone jack's brutal volume curve

The Pi outputs through its onboard 3.5mm jack (the bcm2835 "Headphones" device). A laptop plugged into the same DAX66 input played fine, so the amp and speakers were good. squeezelite was connected and streaming. But the mixer told the real story:

```text
Simple mixer control 'PCM'
  Mono: Playback -3856 [60%] [-38.56dB] [on]
```

`[on]`, not muted, at 60% - so the naive read is "that's fine." It is not fine. The bcm2835's volume control is logarithmic with a range that bottoms out around **-102 dB**, so "60%" is **-38 dB** - technically audible, practically silence into a line input. Crank it to 100% and it's a healthy +4 dB:

```bash
amixer sset PCM 100% unmute
```

A test tone straight to the device confirmed the hardware path end to end:

```bash
speaker-test -D plughw:CARD=Headphones -c2 -t sine -f 440
```

Two related pitfalls while I was in here:

- **`hw:` vs `plughw:`.** squeezelite was configured with `-o hw:CARD=Headphones`, the raw device with no format conversion - it only plays if the stream's sample rate exactly matches what the chip accepts. `plughw:CARD=Headphones` inserts ALSA's plug layer for automatic conversion. On finicky onboard audio, always use `plughw`.
- **Software vs hardware volume.** squeezelite (with no hardware-mixer flag) attenuates in software from the volume Music Assistant sends. For a matrix amp like the DAX66, you want the opposite: a constant line-level feed from the Pi, with per-room volume handled by the amp. So keep the MA player near 100% and control loudness at the DAX66 zone.

## Gotcha #4: persisting mixer state on a RAM-based player

piCorePlayer runs from RAM (it's TinyCore Linux). I set PCM to 100%, ran the pCP backup, rebooted to verify - and it came right back at -38 dB. The usual `alsactl store` wasn't even present on this build.

The RAM-based fix is to force the value at boot. TinyCore runs `/opt/bootlocal.sh` late in startup, and it's included in the backup, so:

```bash
echo "amixer sset PCM 100% unmute" | sudo tee -a /opt/bootlocal.sh
pcp bu   # persist the backup
```

Because it runs *after* the pCP startup that was resetting the level, it wins. Reboot again and it held at 100%. If your player is RAM-based, don't trust a settings backup to capture live ALSA state - pin it explicitly at boot.

## Bonus: ICY metadata and the HEAD-vs-GET trap

A smaller one from wiring SUB/WAVE's now-playing into Home Assistant. I wanted to know whether the Icecast stream carried inline track titles, so I probed it:

```bash
curl -I -H "Icy-MetaData: 1" https://subwave.rustybower.com/stream.mp3
# station headers, but no icy-metaint
```

I concluded it didn't. Wrong. `curl -I` sends a **HEAD** request, and Icecast omits `icy-metaint` on HEAD - it only sends it on a real GET with the metadata header:

```bash
$ curl -s -H "Icy-MetaData: 1" -D - -o /dev/null https://subwave.rustybower.com/stream.mp3 | grep icy-metaint
icy-metaint: 16000
```

The stream carried live `StreamTitle` metadata all along. Lesson: to check streaming behavior, probe the way the client actually connects. HEAD lies.

For the richer bits ICY can't carry (album, year, the AI DJ's name, listener count) I poll the station's JSON API with a RESTful sensor - the ICY title covers the stream itself, everywhere, for free.

## The red herrings

Debugging honesty - two theories I chased that were wrong:

- **`externalTrafficPolicy: Local`.** The player flapping "unavailable" every 2-3 minutes screamed "connection getting rerouted through the LoadBalancer." I'd fixed exactly that for LMS once, so I added it to the MA service. It didn't help, because the flapping was a *symptom* of the pod-IP stream problem (#2) - stalled stream fetches tipping the player unhealthy - not a transport issue. Fixing the published IP made the flapping stop on its own.
- **Device contention.** The bcm2835 exposes 8 subdevices but they must all share one sample rate, so an active AirPlay session (shairport-sync, holding the device) really can block squeezelite. That was a genuine issue at one point - but it wasn't *the* issue, and I burned time on it before finding #2.

Both were plausible, both wasted time. The lesson I keep relearning: **get the primary evidence before theorizing.** The squeezelite debug log (`-d slimproto=debug`) handed me the pod IP in one line. I should have gone there an hour earlier.

## Where it landed

```
Navidrome (Subsonic, /rest bypasses Authelia)
      │
      ▼
Music Assistant  ──►  piCorePlayer (plughw, PCM pinned 100%)  ──►  DAX66 zones
   (Published IP = LoadBalancer IP)
      │
      └──►  Home Assistant integration → media_player.picoreplayer + every zone
```

One hub. Navidrome is the library, Music Assistant drives the Pi over SlimProto and surfaces every DAX66 zone to Home Assistant, SUB/WAVE is a one-tap favorite, and LMS is retired - its app directory deleted, ArgoCD pruned it.

The recurring theme, if there is one: nearly every failure was a networking assumption that holds on a flat LAN and breaks inside Kubernetes - a service advertising an IP its clients can't reach, an auth proxy in front of a machine-to-machine API, a probe that doesn't connect like a real client. None of it was exotic. All of it was invisible until I looked at the primary evidence instead of the symptom.
