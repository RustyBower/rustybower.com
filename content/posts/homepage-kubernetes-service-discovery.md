---
title: "Building a Self-Updating Dashboard with Homepage and Kubernetes"
date: 2026-01-17
draft: false
tags: ["kubernetes", "homelab", "dashboard", "homepage"]
---

One of the challenges of running a homelab with dozens of services is keeping track of what's running and where. I recently deployed [Homepage](https://gethomepage.dev/) - a modern, fully static dashboard that automatically discovers services from Kubernetes ingresses.

## Why Homepage?

I evaluated several dashboard options including Homarr and Heimdall. Homepage stood out for a few reasons:

- **Kubernetes-native service discovery** - no manual configuration needed
- **Real-time pod status** - shows if services are actually running
- **Clean, modern UI** - dark theme, customizable layout
- **Lightweight** - just a static site with no database

## The Setup

Homepage runs as a simple deployment in Kubernetes with a ServiceAccount that has read access to the cluster. The magic happens through ingress annotations.

### Base Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homepage
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: homepage
      initContainers:
        - name: copy-config
          image: busybox:1.36
          command: ['sh', '-c', 'cp /config-source/* /config/']
          volumeMounts:
            - name: config-source
              mountPath: /config-source
            - name: config
              mountPath: /config
      containers:
        - name: homepage
          image: ghcr.io/gethomepage/homepage:v0.10.9
          volumeMounts:
            - name: config
              mountPath: /app/config
      volumes:
        - name: config-source
          configMap:
            name: homepage-config
        - name: config
          emptyDir: {}
```

The init container pattern is important here - Homepage needs a writable config directory to create log files, but ConfigMaps are read-only. We copy the config to an emptyDir volume at startup.

### Service Discovery via Annotations

The real power comes from ingress annotations. For each service you want on the dashboard, add these annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frigate
  annotations:
    gethomepage.dev/enabled: "true"
    gethomepage.dev/name: "Frigate"
    gethomepage.dev/group: "Home Automation"
    gethomepage.dev/icon: "frigate.png"
    gethomepage.dev/pod-selector: "app=frigate"
```

The `pod-selector` annotation is crucial for status monitoring. Homepage uses this to find the actual pods and show whether they're running. Without it, you'll see "not found" errors.

### ConfigMap for Layout

The ConfigMap controls the dashboard layout and widgets:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: homepage-config
data:
  settings.yaml: |
    title: Homelab
    theme: dark
    color: slate
    headerStyle: clean
    layout:
      Home Automation:
        style: row
        columns: 4
      Media:
        style: row
        columns: 4
      Infrastructure:
        style: row
        columns: 4

  widgets.yaml: |
    - kubernetes:
        cluster:
          show: true
          cpu: true
          memory: true
          showLabel: true
          label: "cluster"
        nodes:
          show: true
          cpu: true
          memory: true

  kubernetes.yaml: |
    mode: cluster

  services.yaml: |
    []

  docker.yaml: |
```

Note that `services.yaml` and `docker.yaml` must be valid YAML (even if empty). Setting `services.yaml: "[]"` prevents JavaScript errors from empty files.

## Gotchas I Encountered

### 1. Empty Config Files Cause JS Errors

If `services.yaml` or `docker.yaml` are empty or contain only comments, Homepage throws a `classList` null reference error. Always set them to valid YAML.

### 2. Pod Selector Mismatches

Homepage defaults to looking for pods with `app.kubernetes.io/name=<ingress-name>`. Most of my deployments use simpler labels like `app=frigate`. The `gethomepage.dev/pod-selector` annotation fixes this.

### 3. Icon Names

Icons come from the [Dashboard Icons](https://github.com/walkxcode/dashboard-icons) project. Some names aren't obvious:
- ArgoCD: `argo-cd.png` (not `argocd.png`)
- Z-Wave JS: `mdi-z-wave` (Material Design Icons format)

## Result

Now every time I add a new service with the proper annotations, it automatically appears on my dashboard with real-time status. No more manually updating bookmark pages or forgetting what port something runs on.

The dashboard shows CPU/memory usage for the cluster and nodes, groups services logically, and immediately tells me if something is down. It's become my starting point for accessing everything in the homelab.
