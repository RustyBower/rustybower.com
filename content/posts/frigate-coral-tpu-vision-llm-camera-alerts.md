---
title: "Teaching Security Cameras to Describe What They See"
date: 2026-07-28
draft: false
tags: ["frigate", "coral-tpu", "llm", "homelab", "kubernetes", "home-automation", "computer-vision"]
---

A notification that says "person detected on doorbell" is useful. A notification that says "a delivery driver in a brown UPS uniform placed a medium-sized box on the porch and is walking back to the truck" is significantly more useful — especially when you're in a meeting and trying to decide whether to get up.

I wanted that second kind of notification. The pieces were already in my homelab: [Frigate](https://frigate.video) NVR for camera management, a Coral USB TPU for real-time object detection, and a Mac Studio running [llama-swap](https://github.com/mostlygeek/llama-swap) for LLM inference. The challenge was wiring them together so that every time Frigate detects a person or package, a vision model looks at the image and writes a one-sentence description.

## The detection pipeline

The full chain has four stages, each handled by different hardware:

```
Camera (RTSP) → Frigate (Coral TPU) → Vision LLM (Mac Studio) → Notification
     5 cams         object detect         describe image          HA / phone
                      ~50ms                  ~3-5s
```

**Stage 1: Camera streams.** Five cameras — doorbell, garage, dog run, and two library-facing cameras — push RTSP streams to Frigate through go2rtc for restreaming. The doorbell is an Amcrest; the others are Dahua. All decode via VAAPI hardware acceleration on the Intel NUC that runs Frigate.

**Stage 2: Object detection on the Coral TPU.** Frigate pulls frames at 4 FPS and runs them through a [Frigate+](https://plus.frigate.video/) custom model on the Coral Edge TPU. The model detects people and packages. A USB Coral on a dedicated node (kube02) handles this — it's fast enough that detection latency is under 50ms per frame.

**Stage 3: Vision LLM description.** When Frigate creates a review alert — meaning a tracked object persisted long enough to be interesting — it sends the snapshot to a vision language model. Frigate's built-in GenAI support handles this natively. I'm using `qwen3.5-vl:9b`, a 9-billion parameter vision-language model, served by llama-swap on the Mac Studio.

**Stage 4: Notification.** The description gets attached to the Frigate event. Home Assistant picks it up and pushes it to my phone.

## Getting the Coral into Kubernetes

The Coral TPU is a USB device physically plugged into one specific node. Making it available to a Kubernetes pod requires a few layers:

First, the node is labeled and tainted so only Frigate can schedule there:

```yaml
# Node label + taint on kube02
coral-node: "true"
coral-node=true:NoSchedule
```

Then [generic-device-plugin](https://github.com/squat/generic-device-plugin) runs as a DaemonSet to register USB devices as Kubernetes resources. It watches for the Coral's USB vendor/product IDs (Google and Global Unichip both make Coral variants):

```yaml
args:
  - --device
  - |
    name: coral
    groups:
      - usb:
          - vendor: "18d1"
            product: "9302"
          - vendor: "1a6e"
            product: "089a"
```

Frigate's deployment patch adds the corresponding nodeSelector and toleration:

```yaml
nodeSelector:
  coral-node: "true"
tolerations:
  - key: coral-node
    value: "true"
    effect: NoSchedule
```

In practice, the Frigate container runs privileged and accesses the Coral directly through `/dev`. The device plugin is mostly there for scheduling correctness — making sure nothing else lands on the Coral node and that Frigate only schedules where the hardware exists.

## The vision LLM integration

Frigate 0.14+ has native GenAI support. You configure it in `config.yaml` and Frigate handles the API calls, prompt construction, and response parsing:

```yaml
genai:
  provider: openai
  api_key: "sk-no-key-required"
  base_url: http://ollama.ollama.svc.cluster.local:11434/v1
  model: qwen3.5-vl:9b

review:
  alerts:
    labels:
      - person
      - package
  genai:
    enabled: true
    alerts: true
    image_source: preview
    activity_context_prompt: >-
      Suburban residential home. Household includes family members and a dog.
      Common expected visitors: delivery drivers, mail carrier, neighbors.
```

A few things worth noting:

**The "ollama" service isn't Ollama.** It's a Kubernetes Service with a manual Endpoints resource pointing at the Mac Studio's IP, where llama-swap serves an OpenAI-compatible API. Frigate doesn't know or care — it just hits the `/v1/chat/completions` endpoint. I wrote about this pattern in [Mac Studio as a Kubernetes Compute Satellite](/posts/mac-studio-kubernetes-compute-satellite/).

**Per-object prompts make a big difference.** Generic prompts produce generic descriptions. Telling the model exactly what to focus on for each object type produces much better output:

```yaml
objects:
  genai:
    enabled: true
    prompt: >-
      This image is from the {camera} camera at a suburban residential home.
      In one sentence, describe what the {label} is doing.
    object_prompts:
      person: >-
        In one sentence, describe this person: their appearance (clothing, build),
        what they are carrying, and what action they are taking.
      package: >-
        In one sentence, describe the package: its size, shape, and where it was left.
        Note if a delivery person is still visible.
```

**The activity context prompt reduces false alarm anxiety.** Without context, the model tends toward security-guard language: "an unidentified individual is approaching the premises." With context ("suburban home, family, dog, delivery drivers"), it calibrates better: "a woman in a gray jacket is walking a golden retriever past the driveway."

## The doorbell shortcut

The GenAI descriptions are great for review events that I check later. But for real-time "someone's at the door" situations, I also have an AppDaemon automation that bypasses the LLM entirely and just throws the Frigate snapshot onto a Google display:

```python
class DoorbellPersonFrigateSnapshot(hass.Hass):
    def person_count_changed(self, entity, attribute, old, new, kwargs):
        # Only trigger on 0 → >0 transition
        old_num = float(old) if old not in (None, "unknown", "unavailable", "") else 0.0
        new_num = float(new)
        if not (old_num == 0 and new_num > 0):
            return

        ts = int(time.time())
        snapshot_url = f"{self.frigate_base_url}/api/{self.frigate_camera}/person/snapshot.jpg?t={ts}"
        self.call_service(
            "media_player/play_media",
            entity_id=self.media_player,
            media_content_id=snapshot_url,
            media_content_type="image/jpeg",
        )
        self.run_in(self.stop_display, self.display_duration)
```

When Frigate's person count goes from 0 to any positive number, it fetches the latest person snapshot (cache-busted with a timestamp) and casts it to the office Google Nest Hub for two minutes. No LLM latency — the image is on the display within a second or two of detection.

## The config injection problem

Frigate expects to own its `config.yaml` and writes to it at runtime (for UI-based config changes). But I want the config in Git, managed as a Kubernetes ConfigMap. These two requirements conflict — you can't mount a ConfigMap as writable.

The workaround is a busybox init container that copies the ConfigMap onto the PVC before Frigate starts:

```yaml
initContainers:
  - name: copy-config
    image: busybox:latest
    command:
      - sh
      - -c
      - |
        cp /config-source/config.yaml /config/config.yaml
        cp /config-source/config.yaml /config/backup_config.yaml
    volumeMounts:
      - name: config-configmap
        mountPath: /config-source
      - name: config-volume
        mountPath: /config
```

This means every pod restart resets the config to what's in Git. Any changes made through Frigate's UI are ephemeral — which is exactly what I want. The Git repo is the source of truth.

## Model selection

I've tried several vision models for this:

| Model | Size | Speed | Quality |
|-------|------|-------|---------|
| minicpm-v | 3B | ~1s | Brief, sometimes misses details |
| llama3.2-vision | 11B | ~5s | Good but verbose |
| qwen3.5-vl:9b | 9B | ~3s | Best balance — concise and accurate |

`qwen3.5-vl:9b` has been the sweet spot. It's fast enough that descriptions are ready by the time I check the notification, and accurate enough that I trust it to describe what's happening without hallucinating details. The "one sentence" instruction in the prompt is critical — without it, every model produces a paragraph.

## What it actually looks like

A typical Frigate event now has:

- **Thumbnail**: Cropped image of the detected person/package
- **Snapshot**: Full camera frame at time of detection
- **GenAI description**: "A delivery driver in a dark blue uniform placed a small brown box on the front porch steps and is returning to a white van in the driveway."

The description shows up in the Frigate UI timeline and in Home Assistant notifications. It turns "motion detected" into something I can act on without opening the camera feed.

Total cost: a $30 Coral USB stick and GPU time on a Mac I already owned. No cloud APIs, no subscriptions, no video leaving the network.
