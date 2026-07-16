---
title: "Mac Studio as a Kubernetes Compute Satellite"
date: 2026-07-21
draft: false
tags: ["kubernetes", "mac-studio", "docker", "homelab", "ml", "llm", "gitops"]
---

My K3s cluster runs on six Intel NUCs. They're great for the 50+ containers that make up my homelab — Plex, Home Assistant, Frigate, Paperless, the usual suspects. But they're terrible at machine learning. Four Skylake cores and 32 GB of RAM per node doesn't get you far when Immich wants to classify 80,000 photos or Frigate wants a vision model to describe who's at your door.

The Mac Studio sitting under my desk — M1 Ultra, 64 GB unified memory — is the opposite problem. Absurd single-machine ML performance, but I don't want to migrate my entire cluster to it. I want the K8s cluster to stay the control plane for routing, TLS, service discovery, and GitOps, while the Mac handles the heavy compute.

The solution turned out to be one of Kubernetes' oldest and least glamorous features: manual Endpoints.

## The pattern

In Kubernetes, a Service normally discovers its backends through label selectors — it finds pods with matching labels and routes traffic to them. But if you create a Service *without* a selector and pair it with a hand-written Endpoints resource, you can point that Service at any IP address. Including one outside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  type: ClusterIP
  ports:
    - port: 11434
      targetPort: 11434
      name: http
  # No selector — backends come from the Endpoints resource
---
apiVersion: v1
kind: Endpoints
metadata:
  name: ollama  # Must match the Service name
subsets:
  - addresses:
      - ip: 192.168.1.200  # Mac Studio's LAN IP
    ports:
      - port: 11434
        name: http
```

That's it. Every pod in the cluster can now reach `ollama.ollama.svc.cluster.local:11434` and the traffic routes to the Mac Studio. Add an Ingress in front and it's also available at `ollama.bowerha.us` with TLS termination, external-dns, and Homepage dashboard integration — all the same infrastructure every other service gets.

The Mac Studio doesn't know or care that Kubernetes exists. It's just listening on a port.

## What runs on the Mac

Six services currently use this pattern:

| Service | Port | What it does |
|---------|------|-------------|
| llama-swap | 11434 | LLM inference (OpenAI-compatible API) |
| Immich ML | 3003 | Photo classification and face detection |
| Pet Tagger | 2287 | Individual dog identification (YOLO + CLIP) |
| ComfyUI | 8188 | Image generation (Stable Diffusion) |
| Fooocus | 7865 | Image generation (simplified UI) |
| Stable Diffusion | 7860 | Image generation (A1111 WebUI) |

The first three run as Docker Compose on the Mac. The image generation tools run natively. llama-swap runs as a macOS LaunchAgent via a plist:

```xml
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.llama-swap</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/rusty/.local/bin/llama-swap</string>
        <string>--config</string>
        <string>/Users/rusty/.local/share/models/llama-swap.yaml</string>
        <string>--listen</string>
        <string>0.0.0.0:11434</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Each K8s-side app follows the same three-resource template: a Service (no selector), an Endpoints pointing to `10.0.10.109`, and an Ingress for external access. Adding a new proxied service takes about two minutes.

## Keeping the Mac in the GitOps loop

The obvious downside of running workloads outside the cluster is that ArgoCD can't manage them. The Docker Compose files and configs aren't Kubernetes resources. But they're still code, and they still need version tracking and dependency updates.

I solved this by committing the Mac's Docker Compose files into the same repo under `hosts/mac-studio/`:

```
hosts/mac-studio/
├── README.md
├── immich-ml/
│   └── docker-compose.yml
└── pet-tagger/
    ├── docker-compose.yml
    └── .env.example
```

Renovate's `docker-compose` manager is scoped to `hosts/**/docker-compose.yml`, so image updates get the same automated PRs as everything else in the cluster. When Immich releases a new version, Renovate opens a single PR that bumps both the in-cluster `immich-server` and the Mac's `immich-machine-learning` — keeping them in lockstep, which matters because their internal API isn't versioned.

Deploying a Mac-side update is still manual — SSH in, `docker compose pull && docker compose up -d` — but at least the change is tracked in Git and the version is always visible.

## Why not just run the Mac as a K8s node?

I tried this. K3s runs on macOS with some effort, and you can register a Mac as a worker node. The problems:

1. **Device access is painful.** The M1's GPU is the whole point, but exposing Apple Silicon's unified memory to a container runtime is an unsolved problem. Docker Desktop on Mac doesn't pass through the GPU. Native containers barely exist.
2. **launchd vs. kubelet.** macOS already has a solid process supervisor. Fighting it to let kubelet own everything creates more problems than it solves.
3. **The Mac reboots for updates.** macOS updates are not optional in the way Linux `unattended-upgrades` are. Having a K8s node disappear for 10 minutes during a macOS update is disruptive.

The Endpoints approach avoids all of this. The Mac is a black box that happens to speak HTTP on known ports. Kubernetes doesn't need to schedule pods on it, manage its lifecycle, or understand its hardware.

## The one service that skips the proxy

Immich ML is the exception. Rather than routing through a Service and Endpoints, the Immich server pod points directly at the Mac via an environment variable:

```yaml
env:
  - name: IMMICH_MACHINE_LEARNING_URL
    value: http://192.168.1.200:3003
```

This was a pragmatic choice. Immich ML processes thousands of photos during bulk imports, and the extra hop through kube-proxy added enough latency to be noticeable. For a service that only talks to one consumer, the direct connection is simpler and faster.

## What I'd change

**Static IPs are fragile.** The Mac's static IP is hardcoded in six Endpoints resources. If it changes, I need to update all of them. A DNS-based approach (ExternalName Service pointing at a hostname) would be cleaner, but ExternalName Services don't support port remapping, which several of these services need.

**No health checking.** With real pod backends, Kubernetes removes unhealthy endpoints automatically. With manual Endpoints, if the Mac is down, traffic routes to a dead IP and times out. I could add a sidecar that periodically checks the Mac and updates the Endpoints resource, but it hasn't been painful enough to build yet.

**Docker Compose deploys are manual.** A webhook that triggers `docker compose pull && up -d` on the Mac when Renovate merges a PR would close the GitOps loop completely. Watchtower could do this, but I'd rather build something that only acts on merged PRs, not on every image push.

Despite these rough edges, the pattern has been running for about six months without issues. The Mac handles all the ML inference for the cluster — photo classification, LLM serving, image generation, pet identification — while the NUCs handle everything else. The Endpoints proxy is invisible to every consumer; they just call a `.svc.cluster.local` address and get an answer.
