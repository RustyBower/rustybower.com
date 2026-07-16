---
title: "Unit-Testing Home Automations That Run in Kubernetes"
date: 2026-07-16
draft: false
tags: ["home-assistant", "appdaemon", "python", "kubernetes", "testing", "homelab", "smart-home"]
---

Home automations are code that controls physical things. When they break, the consequences aren't a 500 error — they're a garage door that won't close in a snowstorm, or window shades that open at 3 AM, or a tablet that charges to 100% and stays plugged in for months until the battery swells.

I run about 20 [AppDaemon](https://appdaemon.readthedocs.io/) automations in my homelab, all deployed as Kubernetes ConfigMaps and tested with [appdaemontestframework](https://github.com/FlorianKemworthy/appdaemontestframework). The testing isn't academic — it's caught real bugs that would have been annoying or expensive to discover in production.

## Why AppDaemon over HA automations?

Home Assistant's built-in automations are great for simple trigger-action rules. "When the sun sets, turn on the porch lights." But anything with branching logic, state tracking, or multiple interacting conditions becomes hard to read and impossible to test.

AppDaemon lets you write automations as Python classes. Each one gets an `initialize()` method where you set up listeners and timers, and callback methods that fire when things happen. It's just Python — you can use conditionals, loops, data structures, and (critically) unit tests.

The tradeoff is deployment complexity: you need to run AppDaemon as a separate service alongside Home Assistant, manage the Python files, and wire up the configuration. Kubernetes and Kustomize make this manageable.

## The deployment model

Every AppDaemon automation lives as a `.py` file in the Git repo. Kustomize bundles them into a ConfigMap and mounts them into the AppDaemon container:

```yaml
# environments/bowerhaus/apps/appdaemon/kustomization.yaml
configMapGenerator:
  - name: appdaemon-config
    files:
      - appdaemon/apps.yaml
      - appdaemon/bedroom_shades.py
      - appdaemon/garage_door_auto_close.py
      - appdaemon/tablet_charger.py
      - appdaemon/doorbell_person_frigate_snapshot.py
      # ... 15+ more
```

The deployment mounts this ConfigMap at `/conf/apps`, which is where AppDaemon looks for automation modules:

```yaml
volumes:
  - name: config-volume
    configMap:
      name: appdaemon-config
containers:
  - name: appdaemon
    volumeMounts:
      - name: config-volume
        mountPath: /conf/apps
```

The workflow is: edit Python → push to Git → ArgoCD syncs the ConfigMap → AppDaemon picks up the changes. No SSH, no file copying, no manual restarts.

## Example: temperature-aware garage doors

This one has saved me real money on heating bills. In cold weather, if a garage door is left open, the automation starts a countdown based on the current temperature:

| Temperature | Timeout |
|-------------|---------|
| Below 30°F | 7 minutes |
| Below 40°F | 15 minutes |
| Below 50°F | 30 minutes |
| 50°F+ | No auto-close |

Before closing, it flashes the garage lights to warn anyone who might be in the way:

```python
class GarageDoorAutoClose(hass.Hass):
    def get_timeout_for_temperature(self):
        temp = float(self.get_state(self.temp_sensor, attribute="temperature"))
        if temp < 30:
            return 7 * 60
        elif temp < 40:
            return 15 * 60
        elif temp < 50:
            return 30 * 60
        return None

    def auto_close_door(self, kwargs):
        door = kwargs["door"]
        # Re-check temperature — it might have warmed up during the wait
        if self.get_timeout_for_temperature() is None:
            self.log(f"Temperature now above 50°F — skipping auto-close")
            return
        # Flash lights as warning, then close
        self.flash_lights_then_close(door, ...)
```

The re-check on close is important. If the door opens at 29°F and warms to 52°F during the 7-minute wait, the auto-close cancels. Temperature thresholds are checked twice — once when the timer starts and once when it fires.

## Example: forecast-aware window shades

The bedroom shade automation is the most complex one. It manages motorized shades based on time of day, weather forecast, and thermostat state:

- **Morning:** Open at 8:30 AM, *unless* the forecast high is above 85°F (solar heat gain would fight the AC all day).
- **Midday:** If the thermostat starts cooling, close the shades to reduce heat gain. When cooling stops, re-open — but only if it's before 5 PM.
- **Evening:** Close at sunset or 7 PM, whichever comes first.

The same `BedroomShades` class is reused across rooms with different parameters:

```yaml
# apps.yaml
bedroom_shades:
  module: bedroom_shades
  class: BedroomShades
  covers:
    - cover.upstairs_master_bedroom_valance
    - cover.upstairs_master_bedroom_south
  thermostat: climate.master_bedroom_thermostat
  forecast_high_threshold: 85
  morning_open_enabled: true

# Guest room: close-only (no morning open, no cooling re-open)
guest_shades:
  module: bedroom_shades
  class: BedroomShades
  covers:
    - cover.upstairs_guest_bedroom_valance
  thermostat: climate.guest_bedroom_thermostat
  morning_open_enabled: false
  forecast_high_threshold: 78
```

That `morning_open_enabled: false` flag exists because of a bug. Some rooms' shades were opening unexpectedly when the AC cycling stopped — the "cooling stopped → re-open" path didn't check whether the shades were supposed to be auto-opened in the first place. The test that caught this:

```python
@automation_fixture(
    (BedroomShades, {
        "covers": ["cover.guest_south"],
        "thermostat": "climate.guest",
        "morning_open_enabled": False,
    })
)
def guest_shades_app():
    pass

def test_no_reopen_when_morning_open_disabled(given_that, guest_shades_app):
    opens = []
    guest_shades_app.open_shades = lambda reason: opens.append(reason)
    guest_shades_app.sun_down = lambda: False
    guest_shades_app.is_before_reopen_cutoff = lambda: True
    guest_shades_app.closed_for_cooling = True

    guest_shades_app.thermostat_changed(
        "climate.guest", "hvac_action", "cooling", "idle", {}
    )

    assert opens == []  # Shades should NOT have opened
```

The `@automation_fixture` decorator from `appdaemontestframework` instantiates the real app class with mock Home Assistant services injected. You can call any method directly and assert on the side effects. The test above monkey-patches `open_shades` to record calls instead of actually calling the cover service — pure behavioral testing.

## Example: tablet battery management

Three Fire tablets run [Fully Kiosk Browser](https://www.fullykiosk.com/) as Home Assistant dashboards around the house. Each one sits on a Shelly smart plug. The automation keeps batteries between 25% and 80%:

```python
class TabletCharger(hass.Hass):
    def battery_check(self, entity, attribute, old, new, kwargs):
        battery = int(float(new))
        plug_on = self.get_state(cfg["plug"]) == "on"

        if battery <= 25 and not plug_on:
            self.turn_on(plug)
        elif battery >= 80 and plug_on:
            self.turn_off(plug)

    def periodic_check(self, kwargs):
        # Every 5 minutes, catch edge cases the state listener might miss
        for name, cfg in self.tablets.items():
            battery = int(float(self.get_state(cfg["battery"])))
            if battery >= 100 and self.get_state(cfg["plug"]) == "on":
                self.turn_off(cfg["plug"])  # Hard cutoff at 100%
```

The state listener handles the normal case. The periodic check is a safety net — if a state change event is missed (HA restart, network hiccup), the 5-minute sweep catches it. The 100% hard cutoff is separate from the 80% target because a tablet that jumps from 79% to 100% between state reports would otherwise stay charging indefinitely.

## Testing with time travel

Some automations are timer-based — they do something after N hours. The test framework has a `time_travel` helper that fast-forwards timers without actually waiting:

```python
def test_projector_sends_notification_after_6_hours(
    given_that, theatre_projector_app, time_travel, assert_that
):
    given_that.state_of("media_player.theater").is_set_to("on")
    theatre_projector_app.projector_turned_on(
        "media_player.theater", "state", "off", "on", {}
    )

    time_travel.fast_forward(21600)  # 6 hours in seconds
    theatre_projector_app.check_projector_6_hours(
        {"entity": "media_player.theater"}
    )

    assert_that().notification().was_sent_with_message(
        "Projector has been on for 6 hours"
    )
```

This tests that the projector automation sends a push notification after 6 hours and force-powers-off the projector after 12. Without time travel, you'd have to wait 12 actual hours to verify the behavior.

## The testing payoff

The tests have caught three classes of bugs:

1. **Flag interactions.** The `morning_open_enabled` bug above — a flag meant to disable one behavior accidentally gated a different behavior.
2. **Timer state leaks.** A cancelled timer's callback still firing because the handle wasn't properly cleared.
3. **Edge case arithmetic.** Temperature thresholds that didn't handle `None` (sensor unavailable) or string values from HA.

None of these would have been caught by "deploy it and watch what happens" — they're the kind of bugs that only manifest under specific conditions (cold snap + garage door + recently restarted HA) and are nearly impossible to reproduce manually.

Running the tests is just `pytest`:

```bash
cd environments/bowerhaus/apps/appdaemon
pip install appdaemontestframework
pytest tests/
```

No Kubernetes cluster needed, no Home Assistant instance, no real hardware. The tests run in CI alongside `kustomize build` validation.

## Is it over-engineered?

Maybe. Plenty of people run Home Assistant automations from the UI and never look back. But I manage 20+ automations across six thermostats, three garage doors, five window shade motors, three tablets, and a projector. At that scale, "edit YAML in the HA UI and hope for the best" stops working. Automations interact with each other in ways that are hard to predict without tests, and bugs in physical-world automations have real consequences.

The ConfigMap deployment model means every change goes through Git, gets reviewed in a diff, and can be rolled back. The tests mean I can refactor an automation without manually verifying every scenario. It's the same discipline we apply to production services — it just happens to control window shades instead of API endpoints.
