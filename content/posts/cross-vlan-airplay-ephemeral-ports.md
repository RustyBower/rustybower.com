---
title: "It Wasn't PTP: Cross-VLAN AirPlay on UniFi"
date: 2027-02-09
draft: false
categories: ["networking", "debugging", "homelab"]
tags: ["unifi", "airplay", "vlan", "tcpdump"]
---

My TVs live on an isolated IoT VLAN, walled off from the devices that actually matter. That arrangement held up fine until I tried to AirPlay a laptop's screen to one of them from the main network. The TV showed up in the AirPlay picker, the mirror started connecting, and then: a black screen, a long pause, and *"could not connect."*

Every guide points at the same two things — turn on the mDNS reflector, and allow traffic between the VLANs. I already had the reflector. The second one is where it gets interesting, because "allow traffic between the VLANs" is both more than you need and, by default, pointed the wrong way.

This is the write-up of what's actually happening, how I proved it with a capture on the gateway instead of guessing, and the one narrow rule that fixed it without tearing a hole in the segmentation.

## The setup

Two VLANs, two firewall zones:

- **Main** — `10.0.1.0/24`, zone `Internal`. Laptops and phones.
- **IoT** — `10.0.9.0/24`, its own zone. The TVs, isolated.

The gateway already reflected mDNS onto the IoT VLAN, so discovery worked — the TV appeared in the picker from a laptop on Main. The inter-zone policy allowed `Internal → IoT` (Main can reach the TVs) and allowed only *return* traffic the other way, `IoT → Internal`. That reads like a complete setup. It isn't.

## An AirPlay session is three separate things

A screen mirror isn't one connection. It's three independent flows, and each has to cross the VLAN boundary on its own:

| Leg | Direction | Transport | Crosses? |
|---|---|---|---|
| Discovery | sender → TV | mDNS, UDP 5353 | yes — reflected |
| Control | sender → TV | TCP 7000 | yes — `Internal → IoT` allowed |
| Media + timing | **TV → sender** | **UDP, high ephemeral ports** | **no — dropped** |

That third leg is the whole problem. In AirPlay 2, once control is negotiated on TCP 7000, the **receiver opens its own flows back to the sender** to carry the screen data and timing — outbound from the TV, to a high ephemeral port the sender chose. To the firewall that's a brand-new `IoT → Internal` connection. My default there was "allow return traffic only," and a new flow is not a return of anything. So it's dropped. Control stays up just long enough to paint a black frame, the media never arrives, the session gives up.

## Proving it instead of guessing

The forums are loud with confident wrong answers here — "it's PTP multicast, you can't route it," "just open all the ports." Rather than cargo-cult a fix, I captured the session on the gateway, watching both bridge interfaces at once: `br0` (Main) and `br9` (IoT).

```
# AirPlay control, the PTP timing ports, and the TV's traffic,
# across both VLAN bridges
$ tcpdump -i any -nn -e \
    'host 10.0.9.41 or udp port 319 or udp port 320' \
    -w /tmp/airplay.pcap
```

Then I replayed it with one question: the TV's media packets reach the gateway — do they make it *across* to the sender, or stop at the boundary?

```
# TV's UDP media, arriving on the IoT bridge
$ tcpdump -nn -e -r airplay.pcap 'udp and src host 10.0.9.41' | grep -c 'br9'
174

# the same packets, forwarded out to the sender on the Main bridge
$ tcpdump -nn -e -r airplay.pcap 'udp and src host 10.0.9.41' | grep -c 'br0 '
0

# and the PTP theory everyone repeats?
$ tcpdump -nn -r airplay.pcap 'udp port 319 or udp port 320' | wc -l
0
```

There it is. **174 media packets reached the gateway from the TV; zero were forwarded.** Dropped at the inter-zone policy. And **zero PTP packets** in the entire session — the "it's multicast timing, give up" answer simply didn't apply to this hardware. Meanwhile the control channel on TCP 7000 showed healthy two-way traffic. Everything lined up with the reverse-media theory and nothing else.

## The fix: narrow, not wide

"Allow traffic between the VLANs" works, and it's what most people settle for. But it inverts the entire reason for putting TVs on an isolated network — now a compromised TV can reach anything on the trusted LAN.

The capture shows exactly how little you actually have to open: only the *TVs*, only *outbound to the sender's VLAN*, only on the *ephemeral range*. One rule:

- **Action:** Allow (with auto-allow return traffic)
- **Source:** zone `IoT` — the specific TV devices, *not* the whole VLAN
- **Destination:** zone `Internal`, network `Main (10.0.1.0/24)`
- **Destination ports:** `49152–65535`
- **Protocols:** TCP and UDP

Why this stays tight enough to not undo the segmentation:

- **Only the named TVs** are a valid source. Cameras, plugs, and everything else on IoT still can't initiate into the trusted network.
- **Only high ephemeral ports.** The TVs can't reach SSH, SMB, RDP, web, or anything that listens on a normal low port. The one thing a peer sends a laptop on `49152–65535` is exactly this kind of media stream.
- **Pin the TVs to fixed IPs** (DHCP reservations), so the device-scoped rule stays valid across reboots.

Same test after the rule, same capture filter — the TV's media now forwards from `br9` to `br0` instead of stopping, and the session volume jumped about ten-fold:

| Measurement | Before | After |
|---|---|---|
| TV media on IoT bridge (`br9`) | 174 | 20 |
| …forwarded to sender (`br0`) | **0 — dropped** | **forwarded** |
| TV → sender packets, total | 481 (stalled) | ~4,260 |
| Screen mirror | black → fail | works |

## Caveats

- **It genuinely wasn't PTP.** AirPlay 2 *can* use PTP (UDP 319/320, multicast) for buffered audio sync, and that really doesn't route across subnets. But mirroring to this TV never emitted a single PTP packet, so chasing a multicast relay would have been wasted effort. On other receivers or with audio-only AirPlay the balance may differ — capture your own session before assuming.
- **Guest and VPN clients still can't cast**, by design. Those zones are block-all to IoT and I didn't touch that. Only the main trusted VLAN can mirror.
- **The sender has to be on the VLAN the rule names.** Cast from a device on a different internal network and you get the same black screen until you add that network to the rule's destination.
- **Same-VLAN always worked.** A phone already on the IoT network mirrored fine the whole time — no boundary, no dropped return path. If you only ever cast from one network, the simplest "fix" is to put the casting devices on the TV's VLAN and skip the firewall rule entirely.

The lesson that keeps paying off: when a protocol *half works* across a firewall, the missing half is almost always a return path the protocol opens itself, in a direction or on ports you didn't anticipate. A five-minute capture on the gateway beats an afternoon of forum fixes — and it's the difference between a surgical rule and tearing a hole in the whole segmentation.
