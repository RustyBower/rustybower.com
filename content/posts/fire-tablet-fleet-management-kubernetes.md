---
title: "Fleet-Managing Fire Tablets from Kubernetes"
date: 2026-08-18
draft: false
tags: ["kubernetes", "homelab", "smart-home", "home-assistant", "cronjob", "fire-tablet"]
---

I have three Amazon Fire tablets mounted around the house running [Fully Kiosk Browser](https://www.fullykiosk.com/) as Home Assistant dashboards. They're cheap ($35-50 each during sales), they have decent screens, and Fully Kiosk turns them into locked-down kiosks that show a dashboard and nothing else.

The problem is configuring them. Each tablet has 60+ settings — kiosk lockdown, screensaver behavior, motion detection wake, screen brightness, crash recovery. Changing a setting means opening the Fully Kiosk admin UI on each tablet individually, navigating through menus, and tapping through options three times. When you inevitably forget one tablet, it behaves differently from the others in a way you don't notice for weeks.

I wanted the same thing I want for everything else in the homelab: a config file in Git that describes the desired state, applied automatically.

## The approach

Fully Kiosk exposes a REST API on port 2323. You can read device info, push settings, trigger reloads, and control the screen. The key endpoints for config management are `setStringSetting` and `setBooleanSetting`:

```
http://<tablet-ip>:2323/?cmd=setStringSetting&key=startURL&value=https://...&password=<pw>
http://<tablet-ip>:2323/?cmd=setBooleanSetting&key=kioskMode&value=true&password=<pw>
```

So the plan: define all settings in a config file, iterate over tablets, push each setting via curl. Run it as a Kubernetes CronJob every six hours to enforce drift correction.

## The config file

Settings live in a plain text file with a typed format — `type:key=value`:

```conf
# Kiosk Mode — lock down so users can't exit
bool:kioskMode=true
bool:lockSafeMode=true
bool:advancedKioskProtection=true
bool:disableHomeButton=true
bool:disablePowerButton=true
bool:disableVolumeButtons=true

# Screen Unlock — bypass Android lock screen
bool:forceScreenUnlock=true
bool:forceSwipeUnlock=true
bool:launchOnBoot=true

# Display — clean dashboard look
bool:showNavigationBar=false
bool:showStatusBar=false
string:startURL=https://ha.example.com/home-yaml/overview

# Screensaver — dim to black after 90s, wake on motion
int:screenBrightness=160
int:timeToScreensaverV2=90000
string:screensaverType=black
int:screensaverBrightness=0
bool:screenOffInDarkness=true

# Motion Detection — front camera wakes screen on presence
bool:motionDetection=true
bool:screenOnOnMotion=true
int:movementSensitivity=80
int:motionDetectionInterval=500

# Stability — auto-recover from crashes
bool:restartAfterCrash=true
bool:restartAfterUpdate=true
bool:reloadOnConnectivityChange=true

# Remote Admin — keep REST API enabled (required for this sync)
bool:enableRemoteAdmin=true
int:remoteAdminPort=2323
```

The type prefix (`bool:`, `string:`, `int:`) maps to the correct Fully Kiosk API command. Comments and blank lines are ignored. The file is human-readable and diffable — when I change a setting, the Git diff shows exactly what changed.

## The sync script

A short shell script reads the config and pushes each setting to each tablet:

```sh
#!/bin/sh
push_setting() {
  local tablet="$1" type="$2" key="$3" value="$4"
  local cmd
  case "$type" in
    string|int) cmd="setStringSetting" ;;
    bool)       cmd="setBooleanSetting" ;;
  esac
  local url="http://${tablet}:${FK_PORT}/?cmd=${cmd}&key=${key}&value=${value}&password=${FK_PASSWORD}"
  http_code=$(curl -s -o /dev/null -w "%{http_code}" -m 30 "$url") || true
  if [ "$http_code" = "200" ]; then
    echo "  OK: ${key}=${value}"
  else
    echo "  FAIL: ${key} (HTTP ${http_code})"
  fi
}

IFS=','
for tablet in $TABLETS; do
  echo "=== Syncing $tablet ==="
  # Check connectivity first
  if ! curl -s -m 30 "http://${tablet}:${FK_PORT}/?cmd=deviceInfo&password=${FK_PASSWORD}" > /dev/null 2>&1; then
    echo "  WARN: Tablet unreachable, skipping"
    continue
  fi
  # Push each setting
  while IFS= read -r line; do
    case "$line" in \#*|"") continue ;; esac
    type=$(echo "$line" | cut -d: -f1)
    rest=$(echo "$line" | cut -d: -f2-)
    key=$(echo "$rest" | cut -d= -f1)
    value=$(echo "$rest" | cut -d= -f2-)
    push_setting "$tablet" "$type" "$key" "$value"
  done < "$CONFIG_FILE"
done
```

It's deliberately simple — `curl` in a loop. No Python, no dependencies beyond a shell and curl. The `curlimages/curl` container image is 11 MB.

The connectivity pre-check is important. Fire tablets go to sleep, lose Wi-Fi, or occasionally just stop responding. Without the check, curl would time out on every setting for unreachable tablets, turning a 30-second job into a 15-minute job. With it, unreachable tablets are skipped in one second.

## The CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: fullykiosk-sync
spec:
  schedule: "0 */6 * * *"
  timeZone: "America/Chicago"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: sync
              image: curlimages/curl:8.14.1
              command: ["/bin/sh", "/scripts/sync.sh"]
              env:
                - name: FK_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: fullykiosk-credentials
                      key: password
                - name: TABLETS
                  valueFrom:
                    configMapKeyRef:
                      name: fullykiosk-settings
                      key: tablets
              volumeMounts:
                - name: scripts
                  mountPath: /scripts
                - name: settings
                  mountPath: /config
          volumes:
            - name: scripts
              configMap:
                name: fullykiosk-sync-scripts
            - name: settings
              configMap:
                name: fullykiosk-settings
```

The tablet IP list is a comma-separated string in a ConfigMap literal. Adding or removing a tablet is a one-line change. The Fully Kiosk password is a Sealed Secret.

The environment overlay wires everything together:

```yaml
# environments/bowerhaus/apps/fullykiosk-sync/kustomization.yaml
configMapGenerator:
  - name: fullykiosk-settings
    literals:
      - tablets=192.168.1.51,192.168.1.52,192.168.1.53
    files:
      - settings.conf
```

## The companion: battery management

Config sync handles the software side. The hardware side — specifically, not swelling the batteries by leaving them perpetually charging — is handled by an [AppDaemon automation](/posts/unit-tested-appdaemon-automations-kubernetes/) that manages smart plugs:

```python
class TabletCharger(hass.Hass):
    def initialize(self):
        self.tablets = {
            "kitchen":         {"battery": "sensor.kitchen_tablet_battery",  "plug": "switch.kitchen_charger",  "low": 25, "high": 80},
            "loft":            {"battery": "sensor.loft_tablet_battery",     "plug": "switch.loft_charger",     "low": 25, "high": 80},
            "charging_station":{"battery": "sensor.station_tablet_battery",  "plug": "switch.station_charger",  "low": 25, "high": 80},
        }
        for name, cfg in self.tablets.items():
            self.listen_state(self.battery_check, cfg["battery"], tablet=name)
        self.run_every(self.periodic_check, self.datetime(), 300)
```

Each tablet sits on a Shelly smart plug. When the battery drops to 25%, the plug turns on. At 80%, it turns off. A 5-minute periodic sweep catches any state changes the event listener might miss. There's a hard cutoff at 100% as a safety net.

The 25-80% range is a compromise. Lithium batteries last longest between 20-80%, but fire tablets' battery sensors aren't precise enough below 20% to be safe. 25% gives enough margin that a tablet won't die between the battery reading and the plug turning on.

## Two systems, one fleet

The full tablet management stack is:

| Layer | Tool | What it manages | Frequency |
|-------|------|----------------|-----------|
| Software config | K8s CronJob + curl | 40+ Fully Kiosk settings | Every 6 hours |
| Battery health | AppDaemon + Shelly plugs | Charge cycles (25-80%) | Event-driven + 5 min sweep |
| Dashboard content | Home Assistant | Lovelace dashboards | Real-time |

The tablets themselves are stateless from a configuration perspective. If one dies, I can buy a $35 Fire tablet, install Fully Kiosk, point it at the HA URL, and the CronJob configures everything else within six hours. The settings are the same because they come from a config file in Git, not from someone tapping through menus and trying to remember what they set last time.

This is probably the lowest-stakes use of Kubernetes CronJobs in my cluster. But it solves a real problem — keeping three devices identically configured without manual effort — using the same patterns (Git-tracked config, sealed secrets, scheduled reconciliation) that manage everything else.
