---
title: "Optimizing Z-Wave Thermostats for Home Assistant Control"
date: 2026-01-17
draft: false
tags: ["home-assistant", "z-wave", "hvac", "smart-home", "automation"]
---

I have six Honeywell T6 Pro Z-Wave thermostats throughout my house controlling a multi-zone HVAC system. Out of the box, these thermostats have their own scheduling and "smart" features that conflict with Home Assistant automations. Here's how I configured them to let HA take full control.

## The Setup

- **6x Honeywell T6 Pro Z-Wave thermostats** (TH6320ZW2003)
- **Z-Wave JS UI** running in Kubernetes
- **Home Assistant** with Z-Wave JS integration
- **Gas furnace + AC** (not heat pump) with multi-zone dampers

The zones:
- Great Room & Music Room - shared outdoor unit, multi-zone
- Master Bedroom, Office, Kids Room - shared outdoor unit, multi-zone
- Basement - standalone (heat only, no AC)

## The Problem

The T6 Pro has built-in features that fight against Home Assistant control:

1. **Built-in scheduling** - The thermostat tries to follow its own 5-2 or 5-1-1 schedule
2. **Adaptive Intelligent Recovery** - Pre-heats/cools based on learned patterns
3. **Temperature resolution** - Reports in 1°F increments by default, limiting precision

When HA sends a setpoint change, these features can override it or cause unexpected behavior.

## Key Parameters to Change

After digging through the Z-Wave JS node configuration, here are the critical parameters:

### Parameter 1: Schedule Type
**Set to 0 (No schedule)**

```
Parameter: 1
Value: 0 (None)
Options: 0=None, 1=Occupancy, 2=Every Day, 3=5-2, 4=5-1-1
```

This disables the thermostat's built-in scheduling entirely. HA will handle all scheduling via automations.

### Parameter 27: Adaptive Intelligent Recovery
**Set to 0 (Disabled)**

```
Parameter: 27
Value: 0 (Disabled)
Options: 0=Disabled, 1=Enabled
```

AIR learns how long it takes to reach setpoint and starts heating/cooling early. Sounds nice, but it means the thermostat ignores your setpoint timing. Disable it and let HA handle pre-conditioning if needed.

### Parameter 44: Temperature Display Resolution
**Set to 0 (1°F)**

```
Parameter: 44
Value: 0 (1°F)
Options: 0=1°F, 1=0.5°F
```

Actually, keep this at 0 for Fahrenheit. If you use Celsius, setting to 1 gives 0.5° resolution which can be useful for tighter control.

### Parameter 6: Cool Stages (Basement Only)
**Set to 0 for heat-only zones**

```
Parameter: 6
Value: 0 (No cooling)
Options: 0=None, 1=1 Stage, 2=2 Stages
```

My basement has no AC, so I set cool stages to 0. This prevents the thermostat from ever calling for cooling.

## How to Change Parameters

### Via Z-Wave JS UI

1. Navigate to your thermostat node
2. Go to Configuration Parameters
3. Find the parameter number
4. Set the new value and click Save

The changes take effect immediately - you'll see the thermostat update its display.

### Via Home Assistant Service Call

You can also use the `zwave_js.set_config_parameter` service:

```yaml
service: zwave_js.set_config_parameter
target:
  device_id: abc123...
data:
  parameter: 1
  value: 0
```

However, I found the Z-Wave JS UI more reliable for bulk changes across multiple devices.

## Multi-Zone Considerations

With multi-zone HVAC, the thermostats operate dampers rather than directly controlling the furnace/AC. A few things to keep in mind:

- **Zone coordination** - If one zone calls for heat while another calls for cooling, you have a conflict. HA automations should prevent this.
- **Minimum airflow** - Most systems need at least one zone open. Configure a "dump zone" or ensure automations don't close all dampers.
- **Equipment protection** - The furnace/AC has its own safeties, but avoid rapid cycling by adding delays between mode changes in HA.

## My HA Automation Strategy

With the thermostats now in "dumb" mode, HA handles:

1. **Time-based scheduling** - Different setpoints for sleep, away, home
2. **Occupancy detection** - Adjust zones based on presence sensors
3. **Weather integration** - Pre-condition before temperature swings
4. **Energy optimization** - Coordinate zones to minimize equipment runtime

The thermostats become simple actuators - they receive a setpoint from HA and maintain it. No more fighting between two "smart" systems trying to be helpful.

## Result

After making these changes across all six thermostats:
- Setpoint changes from HA take effect immediately
- No more mysterious temperature swings from AIR
- Schedules are 100% controlled by HA automations
- The system behaves predictably

The T6 Pro is a solid Z-Wave thermostat once you disable its autonomous features. It reports temperature accurately, responds quickly to commands, and the hardware is reliable. Just don't let it think for itself.
