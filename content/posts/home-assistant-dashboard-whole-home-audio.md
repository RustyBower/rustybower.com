---
title: "Building a Home Assistant Dashboard with Whole Home Audio Control"
date: 2026-01-26
draft: false
tags: ["home-assistant", "smart-home", "lovelace", "audio", "dashboard", "mushroom"]
---

My Home Assistant dashboard has evolved significantly over the past year. What started as the default auto-generated cards has become a carefully organized interface that my whole family actually uses. Here's how I built it, with a focus on the whole home audio system that ties everything together.

## Dashboard Philosophy

Before diving into implementation, a few principles guided the design:

1. **Function over flash** - Every card earns its screen space
2. **One-tap actions** - Common tasks shouldn't require drilling into menus
3. **Status at a glance** - The overview tells you what's happening without interaction
4. **Mobile-first** - Tablets mounted around the house are the primary interface

## The Stack

- **Lovelace YAML mode** - Full control over layout and structure
- **Mushroom cards** - Clean, consistent UI components
- **Custom Layout Card** - Responsive grid layouts that work on any screen
- **Stack-in-card** - Grouping related controls together

## Overview Page

The overview page is designed to answer "what's happening right now?" at a glance:

```yaml
- type: custom:mushroom-chips-card
  alignment: center
  chips:
    - type: weather
      entity: weather.katw

    - type: template
      entity: person.rusty_bower
      icon: mdi:account
      icon_color: "{{ 'green' if is_state('person.rusty_bower', 'home') else 'grey' }}"
      content: Rusty

    - type: template
      icon: mdi:lightbulb-group
      icon_color: "{{ 'amber' if states.light | selectattr('state', 'eq', 'on') | list | count > 0 else 'disabled' }}"
      content: "{{ states.light | selectattr('state', 'eq', 'on') | list | count }} lights"
```

The chip bar at the top shows:
- Current weather
- Who's home (person tracking)
- How many lights are on (tap to navigate to lighting)
- Garage door status (red = open, green = closed)
- HVAC status (heating/cooling active)
- Upcoming trash/recycling days

Quick actions provide one-tap scenes: Goodnight, Spa Mode, All Lights Off, Close All Garage Doors.

## Whole Home Audio Architecture

The audio system uses a **Xantech DAX66** multi-zone amplifier - a commercial-grade matrix amplifier that can route any of 6 sources to any of 18 zones independently. Each zone has its own volume control and can play different content simultaneously.

### The Zones

The house is wired with in-ceiling speakers in 15 zones:

**Main Floor:**
- Kitchen
- Music Room
- Deck
- Patio

**Upper Floor:**
- Master Bedroom
- Master Bathroom
- Rusty Office
- Jessica Office
- Loft

**Lower Level:**
- Garage
- Golf Simulator
- Spa Room
- Exercise Room
- Craft Room
- Workshop

Plus a dedicated **Theater** with an Anthem AV receiver for serious listening.

### Integration Approach

The Xantech communicates via RS-232, but I needed network access from Kubernetes. The solution is a **socat bridge** running as a sidecar:

```yaml
containers:
  - name: socat
    image: alpine/socat:latest
    args:
      - "-d"
      - "-d"
      - "TCP-LISTEN:7001,reuseaddr,fork"
      - "TCP:10.0.10.209:4999,keepalive,forever,interval=10"
```

This bridges TCP connections from Home Assistant to the serial-to-ethernet adapter connected to the Xantech. The `keepalive,forever` ensures the connection recovers from network hiccups.

Home Assistant sees each zone as a `media_player` entity with source selection and volume control.

## Audio Dashboard Design

Designing the audio UI was tricky. With 15 zones, a traditional media player card for each would be overwhelming. The solution: [mini-media-player](https://github.com/kalkih/mini-media-player) with `group: true` for a compact layout, organized by floor.

```yaml
# Whole Home Audio - Grouped by Floor
- type: custom:stack-in-card
  cards:
    - type: custom:mushroom-title-card
      title: Whole Home Audio
      subtitle: Main Floor
    - type: custom:mini-media-player
      entity: media_player.zone_24_2
      name: Kitchen
      group: true
      hide:
        power_state: false
    - type: custom:mini-media-player
      entity: media_player.zone_23_2
      name: Music Room
      group: true
      hide:
        power_state: false
    # ... more zones

    - type: custom:mushroom-title-card
      title: ""
      subtitle: Upstairs
    - type: custom:mini-media-player
      entity: media_player.zone_34_2
      name: Master Bedroom
      group: true
      hide:
        power_state: false
    # ... more zones

    - type: custom:mushroom-title-card
      title: ""
      subtitle: Basement
    # ... basement zones
```

The key settings:
- **`group: true`** - Compact single-line layout with inline volume slider
- **`hide: power_state: false`** - Shows on/off state while keeping the UI minimal
- **Mushroom title cards** with empty `title` and a `subtitle` create floor section headers without extra spacing

This puts all 15 zones in a single scrollable card. Each zone shows its name, power state, and a volume slider - everything you need for quick adjustments.

## Scene Integration

The real power comes from combining audio with other automations:

### Spa Mode

One tap: dims the patio lights, turns on the spa room audio to a relaxation playlist, adjusts the HVAC.

```yaml
- type: custom:mushroom-template-card
  primary: Spa Mode
  icon: mdi:hot-tub
  icon_color: cyan
  tap_action:
    action: call-service
    service: scene.turn_on
    target:
      entity_id: scene.spa_mode
```

### Goodnight

Turns off all audio zones, sets overnight temperature setpoints, dims lights to 5% in hallways, locks doors.

### Party Mode

Syncs kitchen, deck, patio, and music room to the same source at matched volumes.

## Theater Integration

The theater runs separately on an Anthem AV receiver, integrated via IP control. It gets its own section with proper media controls:

```yaml
- type: custom:mushroom-media-player-card
  entity: media_player.anthem_av
  name: Anthem AV
  use_media_info: true
  show_volume_level: true
  volume_controls:
    - volume_set
    - volume_mute
```

The theater lighting also has dedicated scene buttons: Off, Movie (5% seating lights), Intermission (50%), and Full brightness for cleaning.

## Tablet Deployment

Three Fire tablets run [Fully Kiosk Browser](https://www.fully-kiosk.com/) showing the dashboard:
- **Kitchen** - Most used, controls audio and lights for main floor
- **Loft** - Upstairs landing, quick access to bedrooms
- **Charging Station** - Portable tablet that docks when not in use

The Admin page shows tablet connectivity status so I know if one has gone offline.

## Lessons Learned

1. **YAML mode is worth it** - The visual editor can't do responsive layouts or template chips
2. **Group by function, not location** - "Lighting" view is more useful than "Living Room" view
3. **Limit visible options** - Show the 80% use case, hide the edge cases behind taps
4. **Test with family** - My initial designs were too information-dense

## What's Next

- **Music Assistant** integration for better source management
- **Voice control** via local Whisper for "play jazz in the kitchen"
- **Presence-based audio** - automatically route music to follow occupancy

The dashboard continues to evolve, but the core principle stays the same: make the smart home actually easier to use than the dumb version.
