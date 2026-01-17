---
title: "GitOps for Homelabs: Kustomize + ArgoCD Patterns and Pitfalls"
date: 2026-01-17
draft: false
tags: ["kubernetes", "argocd", "kustomize", "gitops", "homelab", "infrastructure-as-code"]
---

I manage two Kubernetes environments - a home cluster (bowerhaus) and a cloud cluster (rustycloud) - using GitOps with Kustomize and ArgoCD. After running this setup for a while, I've learned what works, what doesn't, and some non-obvious gotchas.

## The Architecture

```
kustomize/
├── base/                          # Shared, environment-agnostic configs
│   ├── media/
│   │   ├── lidarr/
│   │   ├── radarr/
│   │   └── sonarr/
│   ├── home-automation/
│   │   ├── home-assistant/
│   │   └── frigate/
│   └── data-analytics/
│       ├── prometheus/
│       └── grafana/
├── environments/
│   ├── bowerhaus/
│   │   ├── applicationsets/       # ArgoCD ApplicationSet
│   │   └── apps/                  # Per-app overlays
│   │       ├── frigate/
│   │       ├── home-assistant/
│   │       └── prometheus/
│   └── rustycloud/
│       ├── applicationsets/
│       └── apps/
│           ├── plex/
│           ├── sonarr/
│           └── grafana/
```

The key principle: **base** contains environment-agnostic resources, **environments** contain overlays that customize for each cluster.

## ArgoCD ApplicationSets

Instead of creating individual ArgoCD Applications for each service, I use ApplicationSets with a git directory generator:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: bowerhaus-appset
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/RustyBower/kustomize.git
        revision: master
        directories:
          - path: environments/bowerhaus/apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/RustyBower/kustomize.git
        targetRevision: master
        path: 'environments/bowerhaus/apps/{{path.basename}}'
        kustomize: {}
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Every directory under `environments/bowerhaus/apps/` automatically becomes an ArgoCD Application. Add a new folder, push to git, and ArgoCD deploys it.

The `kustomize: {}` block forces ArgoCD to use Kustomize even if there's no explicit kustomization.yaml (though you should always have one).

## Base/Overlay Pattern

### Base: Generic, Reusable

The base contains the core deployment without environment-specific details:

```yaml
# base/media/lidarr/lidarr-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lidarr
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: lidarr
          image: lscr.io/linuxserver/lidarr:2.8.2
          ports:
            - containerPort: 8686
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: lidarr-config
```

No NFS paths, no environment-specific storage, no ingress hostnames.

### Overlay: Environment-Specific

The overlay adds what's unique to each environment:

```yaml
# environments/rustycloud/apps/lidarr/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/media/lidarr/
  - ingress.yaml
namespace: lidarr
patches:
  - path: deployment-patch.yaml
```

```yaml
# environments/rustycloud/apps/lidarr/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lidarr
spec:
  template:
    spec:
      volumes:
        - name: media
          nfs:
            server: 10.0.0.105
            path: "/cephfs/media/"
        - name: download
          nfs:
            server: 10.0.0.105
            path: "/cephfs/data/download/complete/music"
      containers:
        - name: lidarr
          volumeMounts:
            - mountPath: /download/complete/music
              name: download
            - mountPath: /data/music
              name: media
              subPath: Music
```

The patch adds NFS volumes specific to rustycloud's storage infrastructure.

## Kustomize Challenges I've Hit

### 1. ClusterRoleBinding Namespace Issues

ClusterRoleBindings reference ServiceAccounts with a namespace. The base might have:

```yaml
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

But your overlay uses `namespace: prometheus`. Kustomize's namespace transformer doesn't update references inside ClusterRoleBindings.

**Solution**: Use an inline patch in your overlay:

```yaml
patches:
  - target:
      kind: ClusterRoleBinding
      name: prometheus
    patch: |-
      apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRoleBinding
      metadata:
        name: prometheus
      subjects:
        - kind: ServiceAccount
          name: prometheus
          namespace: prometheus
```

### 2. Strategic Merge vs JSON Patches

Kustomize's default strategic merge patches work well for adding fields but struggle with:
- Removing fields
- Modifying array items by index
- Complex nested structures

When strategic merge fails, use JSON patches:

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |-
      - op: remove
        path: /spec/template/spec/containers/0/resources/limits
```

### 3. ConfigMap/Secret Name Hashing

Kustomize appends hashes to ConfigMap and Secret names by default. This breaks references if you're not careful.

```yaml
generatorOptions:
  disableNameSuffixHash: true
```

I disable hashing for configs that need stable names (like Renovate's config).

### 4. Image Tags in Different Locations

Renovate and Kustomize both want to manage image tags. I keep images in the base deployments and let Renovate update them there. The `kubernetes` manager in Renovate handles this:

```json
{
  "kubernetes": {
    "managerFilePatterns": [
      "/(^|/)base/.+/(?:[^/]*deployment)\\.ya?ml$/"
    ]
  }
}
```

This tells Renovate to only look at deployment files in the base directory.

### 5. ArgoCD Sync Waves

When deploying interconnected services, order matters. Use sync-wave annotations:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # Deploy before wave 0
```

Negative waves deploy first. I use this for:
- `-2`: Namespaces and CRDs
- `-1`: Secrets and ConfigMaps
- `0`: Main deployments (default)
- `1`: Ingresses and monitoring

## Directory Structure Tips

### One App Per Directory

Each app gets its own directory with a kustomization.yaml. This maps cleanly to ArgoCD Applications and makes it obvious what's deployed where.

### Consistent Naming

I use the app name as:
- Directory name: `apps/frigate/`
- Namespace: `namespace: frigate`
- ArgoCD Application name: `{{path.basename}}` → `frigate`

This consistency makes debugging easier.

### Base Categories

Group bases by function:
- `base/media/` - Plex, *arr stack
- `base/home-automation/` - Home Assistant, Frigate
- `base/data-analytics/` - Prometheus, Grafana, InfluxDB
- `base/dev-tools/` - Gitea, Drone, Renovate

## Self-Healing and Pruning

I enable both in the ApplicationSet:

```yaml
syncPolicy:
  automated:
    prune: true     # Delete resources not in git
    selfHeal: true  # Revert manual changes
```

This ensures git is the source of truth. Any kubectl edits get reverted. Deleted files remove resources.

**Warning**: `prune: true` means deleting a directory from git deletes the entire deployment. Good for cleanup, dangerous if you accidentally remove something.

## The Workflow

1. Make changes in a branch
2. Test with `kustomize build environments/bowerhaus/apps/myapp`
3. Push and merge to master
4. ArgoCD syncs within 3 minutes (or trigger manually)
5. Watch the sync in ArgoCD UI

No kubectl apply, no remembering which cluster you're on, no drift between environments.

## Takeaways

- **ApplicationSets** eliminate per-app boilerplate
- **Base/overlay pattern** enforces separation of concerns
- **Strategic merge patches** work 80% of the time; JSON patches handle the rest
- **Self-heal + prune** keeps clusters matching git
- **Test locally** with `kustomize build` before pushing

The initial setup takes effort, but the ongoing maintenance is minimal. Adding a new service is just creating a directory and pushing to git.
