---
title: "Flashing a BMW From ID4 to ID6 on an M1 Mac"
date: 2027-02-09
draft: false
categories: ["debugging"]
tags: ["bmw", "e-sys", "parallels", "carplay"]
---

My 2016 340i shipped with the NBT EVO head unit running the ID4 interface: no native navigation, no CarPlay, and the older menu layout. The same hardware can run ID6 with wireless CarPlay. BMW just never flashed it. The dealer wanted a few hundred dollars to do the conversion, and the whole job is a software flash plus some coding, so I did it myself.

The complication: every BMW coding tool is x86 Windows software, and my only machine is an M1 MacBook. Apple Silicon can't run Boot Camp, so there's no native Windows. I did the entire flash through an emulated x86 Windows VM, hit a wall on the one small coding step that actually lights up the screen, and finished that step from my phone. This is the write-up.

## The hardware and the ceiling

The car is a 2016 F30 with the NBT EVO head unit. The thing that matters most about a given EVO is how much RAM it has: the early units have **2 GB**, later ones **4 GB**. The 4 GB units can take the newest software. The 2 GB units top out at an I-step around 18-03, which still gets you ID6 and CarPlay but is the ceiling. Mine is a 2 GB unit, so everything below is pinned to that generation of data.

That 2 GB detail also makes the flash the riskiest part. You're overwriting the bootloader and system software on a module that controls the whole dashboard, and a voltage dip partway through can brick it.

## The toolchain, running on emulated x86

The standard BMW coding stack is **E-Sys** plus **PSdZData** (the per-model data package — about 11 GB for this platform). Both are x86 Windows. My options on an M1 were limited, so:

- **Parallels** running **Windows 11 for ARM**
- E-Sys installed inside it, running as an **x86 app under Windows-on-ARM's emulation layer**

That's two layers of translation: an ARM VM, then x86-on-ARM inside it. I fully expected it to fall over. It didn't. E-Sys connected to the car, read the vehicle order, read all 22 control units, and ran a full flash without a single emulation-related failure. The emulation tax showed up only as slower UI, not wrong behavior.

The car talks to the laptop over Ethernet, not OBD-to-USB. BMW's protocol here (HSFZ) runs over plain TCP/IP on a link-local `169.254.x.x` network: a UDP broadcast on port 6811 to find the car, then diagnostics over TCP 6801. A cheap OBD-to-Ethernet ("ENET") cable is the whole interface. Parallels bridges the Mac's USB-Ethernet adapter into the VM, and from E-Sys's point of view it's just a car on the network.

## Power first

Before flashing anything, power. A dip mid-flash is the classic way to brick a 2 GB EVO. The usual advice is "don't use a cheap trickle charger," which is aimed at dumb 1–2 A chargers that can't hold voltage under load.

I used a 10 A smart charger with a 13.6 V supply mode, clamped on in normal mode with the battery connected, every other load killed (ignition on, engine off, climate and lights off, doors shut). The single biggest avoidable mistake is **opening a door mid-flash** — it wakes modules and spikes the current draw. The bulletproof free alternative is to just run the engine, so the alternator holds ~14.2 V and can't be out-drawn by one module.

## The flash

The flash itself is a TAL (a task list E-Sys calculates from the difference between what's on the car and the target software). I backed up everything read-only first — vehicle order, the software configuration, the head unit's coding, and the activation certificates already on the car — and saved copies off the laptop before writing anything.

Then the flash:

| Metric | Value |
|---|---|
| Duration | 41 min 53 s |
| TAL result | Finished |
| Errors | 0 |
| Target software | ID6 EVO, I-step 18-03 |
| Confirmed RAM | 2 GB |

Clean. The head unit rebooted into ID6 software.

## The screen was black

ID6 software was running, but the display showed **"no signal."**

This is expected and it's the part that bites people. The head unit drives the dashboard screen over an internal video link (APIX), and the correct mode depends on which physical screen the car has. A non-touch screen like mine wants one set of values; a touch screen wants another. Flashing the software doesn't set them. Until they're set, the unit is running fine and talking to the world, but putting out a signal the screen can't display.

The fix is three coding parameters on the head unit:

```
CID_APIX_MODE   = APIX1
CID_COLOR_DEPTH = 18
CID_COM_MODE    = SPI
```

(A touch screen would instead want APIX2 / 24 / MII plus a resolution parameter. Setting the wrong set leaves you exactly where you started, staring at "no signal.")

This is **FDL coding** — editing individual parameters inside a module's coding, as opposed to the higher-level "code it to match the vehicle order" operation. And this is where the whole setup hit a wall.

## The wall: the tool that won't run in a VM

High-level coding worked fine in plain E-Sys. FDL coding does not, because that specific capability is gated behind a signed token, and the tooling that provides it refuses to run inside a virtual machine or on ARM. I confirmed it the hard way: every attempt to open the head unit's parameters for editing failed with a "no token" error. The flash — the big, scary, 42-minute operation — worked from my emulated VM. The small final edit did not.

The normal answer is a real Intel or AMD Windows machine. I don't own one, and the point of the project was to not buy hardware for a one-time job.

## Finishing from my phone

The way out was an iOS app, **BimmerUtility**, which has an FDL editor built in and connects to the car the same way E-Sys does — over ENET, not Bluetooth. BMW's CarPlay and this whole protocol are IP-based, so "connect the phone to the car" is literally an Ethernet connection:

```
iPhone → USB-C-to-Ethernet adapter → ENET cable → car's OBD port
```

The one gotcha is that it needs a wired Ethernet adapter on the phone, not a Bluetooth OBD dongle — those speak a different, slower protocol that can't do this. With the phone wired into the OBD port and the engine running to keep the modules awake, the app read the head unit, I opened its coding editor, set the three parameters, and wrote them back.

The screen came on.

## CarPlay is wireless-only, which means an antenna

Last piece. On a BMW, **CarPlay is wireless-only** — there's no wired CarPlay over USB like other brands. It runs over the head unit's WiFi/BT antenna. If that antenna isn't physically connected to the unit, CarPlay can't pair at all, and you'll chase a software ghost that isn't there.

The coding tools had flagged a WiFi-antenna fault, which was the tell. Once I plugged the antenna into the back of the head unit and activated the CarPlay feature, the phone paired wirelessly and CarPlay came up. Native navigation was already working — this car's factory map license was intact, so it lit up on its own once the display was live.

## What actually worked, in order

1. **Flash** ID4 → ID6 — emulated x86 E-Sys in a Windows-on-ARM VM, on an M1 Mac. 42 minutes, zero errors.
2. **High-level coding** of the head unit — same VM, no problem.
3. **FDL coding** (the three display parameters) — *not* possible in the VM; done from the **BimmerUtility iOS app** over a wired Ethernet connection to the OBD port.
4. **CarPlay** — plug in the WiFi/BT antenna, activate the feature, pair wirelessly.

The surprise was which step broke. I assumed the flash would be the fragile part and the coding would be trivial. It was the reverse: the heavy flash sailed through two layers of emulation, and a three-line parameter edit was the thing that needed a completely different device.

## Caveats

- **This can brick the unit.** The 2 GB EVO is genuinely at risk during the flash, and a voltage dip is the usual cause. Solid power is not optional. I got a clean run; that's not a guarantee you will.
- **Licensing is a gray area.** Native navigation and CarPlay are license-activated features. Mine were activated with codes purchased from a third-party retrofit vendor, not from BMW, and the activation side of this involves tooling I'm deliberately not detailing. If you care about doing this strictly through official channels, this isn't that.
- **2 GB is a ceiling, not a floor.** A 2 GB unit tops out around I-step 18-03. You get ID6 and CarPlay; you don't get the newest software a 4 GB unit would take.
- **The VM route has a known gap.** It handles the flash and high-level coding, but not token-gated FDL coding. If you don't have a phone-app fallback or a real x86 Windows machine, you can end up flashed-but-black with no way to finish. Line up the FDL method *before* you flash.
- **Screen type matters.** Non-touch and touch screens need different display parameters. Confirm which one you have before coding, or you'll set the wrong values and stay at "no signal."
