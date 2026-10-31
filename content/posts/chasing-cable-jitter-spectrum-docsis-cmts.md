---
title: "Chasing Cable Jitter: What a Spectrum Ticket Taught Me About DOCSIS and the CMTS"
date: 2027-01-26
draft: false
categories: ["networking", "homelab", "debugging", "data"]
tags: ["spectrum", "docsis", "starlink", "influxdb"]
---

My home connection wasn't down, and it wasn't slow. Its latency just kept jumping: a typical ping took 11 ms, and one in a hundred took 69 ms or more on an idle line. That's a hard thing to report to an ISP, so I started measuring.

That turned into a monitoring stack, a Spectrum ticket, a technician visit that changed two things at once, a crash course in how cable internet works, and eventually a second ISP. I can now prove exactly where the jitter comes from, but I still can't tell you what causes it.

## What I measured

I run an exporter in the cluster that pings at 5 Hz, along the path one hop at a time:

| Target | What a spike here means |
|---|---|
| Home gateway | My own network |
| Modem's LAN port (`192.168.100.1`) | The link to the modem |
| Spectrum's first hop | The cable segment, modem through to the CMTS |
| Public anycast addresses | Everything past that |

It also scrapes the modem, an Arris SB8200, for its signal levels and event log.

Only data from idle windows counts, because a busy line proves nothing. I kept windows under 3 Mbps up and 10 Mbps down, which gave 4.8 hours and about 396,000 pings per target. A "spike" is any ping over 50 ms.

| Target | Share of pings over 50 ms |
|---|---|
| Home gateway | 0.000% |
| Modem LAN port | 0.016% |
| Spectrum's first hop | **1.323%** |
| Google | 1.366% |

That table is the whole diagnosis. The jitter appears at one hop, and everything beyond the CMTS adds 0.04 of a percentage point. The typical idle ping to the first hop was 11 ms, and the 99th percentile was 69 ms. There was no packet loss.

The usual objection is that routers deprioritise pings to themselves, so a slow hop might not be real. That's why you need the targets beyond it. If only the router's own replies were slow, Google wouldn't show the same spikes.

## First, rule out yourself

The worst numbers I had were my own fault. One evening, the first hop's p99 hit 1,600 ms. For those five hours, the house had upload pinned at 33-37 Mbps.

The proof that this was bufferbloat came at 01:20. Upload dropped to 5 Mbps and latency recovered instantly, while download was still running at 178 Mbps. So saturating the upstream ruins latency and download load doesn't.

If you bring that evening to an ISP, they'll say your own traffic is the problem and close the ticket, and they'd be right about that evening. So I reported it up front, separately from the idle measurements.

That 178 Mbps turned out to have its own story. It wasn't real use: my Kubernetes backups had started uploading my entire media library over the connection. The reasoning above survives, but it wrecked a baseline later on.

## How cable internet actually works

This is what I wish I'd known before I started.

**You share a cable with the neighbourhood.** Your modem talks to a node, which serves a group of homes, and the node connects to a **CMTS** (Cable Modem Termination System) at the headend. The CMTS is the far end of your connection, and it's usually your first traceroute hop that isn't your own hardware.

**Downstream is a broadcast, and upstream is a negotiation.** The CMTS sends to everyone and modems pick out their own traffic. Going up, modems can't just transmit, because they'd collide. Yours asks for time, the CMTS replies with a schedule of grants called a MAP, and the modem transmits in its assigned slot. That's why an idle cable connection sits around 10 ms to the first hop when fibre is usually much lower: every upstream packet waits for permission.

**Ranging is how the modem stays in tune.** The CMTS periodically measures each modem's timing and power and tells it to adjust. That's where the most-searched errors come from:

| Log message | Meaning |
|---|---|
| T3 timeout | The modem sent a ranging request and the CMTS didn't answer in time |
| T4 timeout | The modem went too long without being offered ranging at all |
| MDD timeout | The modem missed the periodic broadcast that describes the channels |
| OFDM profile failure | A downstream modulation profile stopped decoding |

**DOCSIS 3.1 moves modulation around per modem.** OFDM downstream and OFDMA upstream use profiles, which trade speed against robustness. The CMTS picks one based on how well it hears you. A modem bouncing between two upstream profiles is sitting on a signal-quality boundary at the *headend*, which is a measurement you can't see from home.

**The levels to know:**

| Measure | Target |
|---|---|
| Downstream power | -7 to +7 dBmV is comfortable; spec is -15 to +15 |
| Downstream SNR | 33 dB or better for QAM256 |
| Upstream transmit power | Roughly 35-50 dBmV |

Upstream power works differently from what you might expect. The CMTS commands the modem to hit a target level at the headend, so high transmit power means a lossy return path.

**Zero uncorrectables doesn't mean a healthy line.** My downstream had no uncorrectable codewords and 41-42 dB SNR, and those are the numbers everyone checks. It looked clean because channels were dropping out cleanly rather than degrading. The event log held what the signal page didn't.

## What the modem said

The SB8200's event log over about two days:

| Event | Count |
|---|---|
| Commanded power over 6 dB below the top of the dynamic range window | 29 |
| OFDM downstream profile failure and recovery pairs | 22 |
| OFDMA upstream profile changes, flipping between 11 and 12 | 18 |
| MDD timeouts, each recovering seconds later | 12 |
| T3 timeouts | 4 |

The first row needs care. It's a standing condition logged again at each ranging, not 29 separate disturbances. The useful part was that both directions were being disturbed together, which points towards the physical path rather than noise on the return side alone.

## The technician

A Spectrum tech came out on August 19, tightened a loose connector at the box, and removed an inline 6 dB attenuator.

| Measure | Before | After |
|---|---|---|
| Downstream power | -7.4 dBmV | -0.4 dBmV |
| Upstream transmit power | 43-46 dBmV | 37-39 dBmV |

That's a nice lesson in how a pad works. A 6 dB attenuator costs 6 dB in both directions, so downstream rose by about 7 and the modem could transmit about 6 lower.

The jitter didn't change:

| Share of pings over 50 ms | p99 | |
|---|---|---|
| 1.32% | 69 ms | Before |
| 1.13% | 66 ms | 30 minutes after |

That visit also left two traps for me:

- **Two changes at once.** Connector and pad went together, so I can't separate their effects. It didn't matter much, since neither helped the jitter.
- **Fake evidence.** T3 timeouts went from 4 to 30, with 22 sync timing failures and the first packet loss I'd recorded. All of that came from a two-minute burst while the drop was disconnected during the work. Quoting those numbers at Spectrum would have been easy and dishonest. Anything you measure during a visit needs marking as such.

Then the baseline. I saved a "clean post-repair baseline" that same day, with the idle filter still in place. Four days later I found the backup job that had been running since that morning. No window could have passed a "under 10 Mbps down" filter, so that baseline was taken on top of a permanent 180 Mbps floor.

## Then I got Starlink

On September 2, Starlink became the primary connection and Spectrum became a standby, reachable through a policy route for a short list of addresses.

That caused a subtle monitoring problem. Every internet target was now silently measuring Starlink while still labelled as Spectrum, and the first hop was unreachable. So I split the targets into anycast pairs, with the same provider's two addresses on each link, sampled on the same clock:

| Provider | Starlink | Spectrum |
|---|---|---|
| Cloudflare | 1.1.1.1 | 1.0.0.1 |
| Google | 8.8.8.8 | 8.8.4.4 |

It also gave me an accidental experiment. Spectrum went from a busy household line to carrying almost nothing beyond my own pings. The router's counters for that port show under 10 kbps on average, which is less than the pings alone should generate, so I'd read that as "effectively idle" rather than a precise number. If load had anything to do with the jitter, this was the time for it to vanish.

Here's about 3.3 million pings per target, from eight days in late October:

| | Spectrum first hop | Spectrum to Cloudflare | Starlink to Cloudflare |
|---|---|---|---|
| p50 | 10.6 ms | 18.9 ms | 31.5 ms |
| p90 | 15.4 ms | 23.7 ms | 55.1 ms |
| p99 | 54.7 ms | 61.8 ms | 93.7 ms |
| p99.9 | 181 ms | 189 ms | 143 ms |
| Over 50 ms | **1.14%** | 1.36% | **14.28%** |

Two things stand out.

First, the Spectrum jitter never went away. It's 1.14% with the line almost idle, against 1.32% before the repair. More than two months, a repair and the removal of all household traffic moved it by less than a fifth of a point. It barely moves day to day either, between 1.05% and 1.37%.

Second, the link I complained about is still the better one for latency at nearly every percentile. Starlink is over 50 ms about ten times as often. That isn't a fair fight, since Starlink carries the household and Spectrum carries nothing. But it's a useful correction: 1.3% that annoyed me was a pretty good result compared with the alternative. Spectrum's bad moments do go further out, with the p99.9 being worse than Starlink's.

The spike overlap also confirms the earlier reasoning. 94.9% of first-hop spikes have a matching spike towards Cloudflare within a second, which is far above chance, so these are real delays on the forwarding path rather than a router being slow to answer pings.

## The clock tells on both of them

A dashboard panel that started as a hunch turned out to be the most interesting one: spikes grouped by which second of the minute they landed on. Random congestion should spread evenly over all 60 seconds. Something scheduled should cluster.

Early on, 83% of a small sample of 178 spikes fell in :35-:40. With eight days of data, the pattern is much stronger and has shifted:

| Second of the minute | Share of Spectrum spikes |
|---|---|
| :02-:05 | 26.5% |
| :11 | 5.2% |
| :32-:35 | 18.4% |
| Most other seconds | About 1% or less each |

Those two main windows make up about 45% of all spikes in 13% of the clock, and they sit 30 seconds apart. The line was idle, so something on a 30-second cycle looks to be stalling it. The smaller bump at :11 doesn't fit that cycle, and I can't explain it.

I can't see what that is. A 30-second period is in the right range for the periodic ranging the CMTS does with every modem, which makes it my leading suspect. It could also be the modem's own housekeeping, or something at the CMTS. The monitor itself looks clear: its modem scrape is on a 15-minute cycle, and it wasn't reaching the modem at all during this period.

Starlink's clock pattern is much weaker but very neat. Its spikes are spread nearly evenly, except for four seconds that stand out at about 2.3% each against about 1.6% elsewhere:

```
:12  :27  :42  :57
```

That's every 15 seconds, and it matches the satellite reassignment schedule that others have reported. So a Starlink dish appears to get a new satellite slot four times a minute, and I can see it in a ping log. It's a small part of the total, though. Most of Starlink's spikes follow the time of day instead, from about 6% in its best hours to about 25% in its worst. That could be a busy shared cell, which would be the satellite version of cable's node problem, but Starlink also carries my whole house, and I can't separate the two.

## What I'd tell myself at the start

1. **Measure per hop, and past the hop.** One ping target can tell you something's wrong. A ladder of them tells you where, and the target beyond tells you it's real.
2. **Filter for idle before claiming anything.** Then check what your filter actually kept.
3. **Volunteer your own failures.** The ISP will find them anyway, and it keeps the real claim credible.
4. **Keep the event log.** The SB8200's log is a rolling buffer and is wiped on reboot, so entries are gone within days unless you copy them.
5. **Keep the data longer than the argument.** Prometheus only holds 15 days here, so everything also goes to InfluxDB with no expiry.
6. **Group by second of the minute.** It's a one-line query that separates "busy" from "scheduled".

A few things specific to scraping this modem, if you have one: each status page takes about 37 seconds to load, so serve metrics from a cache and never fetch inside a scrape. The upstream table's HTML has no row separators. An unlocked channel reports 0.0 dBmV, which will quietly drag down your average. Its clock runs six hours behind UTC and ignores daylight saving. `ping` also prints nothing for a silent timeout, so counting "unreachable" lines undercounts loss; compare expected and received instead.

## Caveats

- **I still don't know the cause.** I've located the jitter between the modem and the CMTS, and shown it's independent of my load and has a 30-second cycle. Ranging is a hypothesis.
- **An oversubscribed node isn't excluded.** The 30-second pattern argues against plain congestion, but only Spectrum can see upstream MER on that channel and node utilisation.
- **The modem isn't excluded either.** Pinging `192.168.100.1` tests its Ethernet side, not its DOCSIS transceiver. A swap is the one variable I haven't tried, and it should be the only change that day.
- **The modem data stops short.** As of late October, the exporter had been unable to reach the modem for at least two weeks, so the event counts above are from August. I can't say whether the profile flips continued.
- **The comparison isn't like for like.** Starlink carries all household traffic and Spectrum carries almost none, and all of this comes from one location over eight days in October.
- **Before and after used different windows.** The August figures are 4.8 filtered hours and the October ones are every sample from eight days. The line was idle throughout October, so no filter was needed, but they aren't the same measurement.
- **The Starlink schedule is a match, not a proof.** My data shows spikes at those seconds. The reason comes from other people's research.
