---
title: "Self-Hosted CI/CD with Drone, Gitea, and Harbor"
date: 2026-01-17
draft: false
tags: ["cicd", "drone", "gitea", "harbor", "kubernetes", "docker", "devops"]
---

I run a fully self-hosted CI/CD pipeline using Drone for builds, Gitea for git hosting, and Harbor for container registry. No GitHub Actions, no Docker Hub, no external dependencies. Here's how it all fits together.

## The Stack

- **Gitea** - Lightweight git server with OAuth2 support
- **Drone** - Container-native CI/CD platform
- **Harbor** - Enterprise container registry with vulnerability scanning
- **BuildKit** - Modern Docker builder for efficient image builds

## Why Self-Hosted?

1. **Privacy** - Code never leaves my network
2. **No rate limits** - Build as often as needed
3. **Offline capability** - Works during internet outages (for local images)
4. **Learning** - Understanding the full DevOps stack
5. **Cost** - No per-minute billing for CI runners

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Kubernetes                           │
│                                                              │
│  ┌──────────┐     ┌─────────────────────────────┐           │
│  │  Gitea   │────▶│          Drone              │           │
│  │  (git)   │     │  ┌───────┐    ┌──────────┐  │           │
│  └──────────┘     │  │Server │    │  Runner  │  │           │
│                   │  └───────┘    └────┬─────┘  │           │
│                   └────────────────────│────────┘           │
│                                        │                     │
│  ┌──────────┐                   ┌──────▼─────┐              │
│  │  Harbor  │◀──────────────────│  BuildKit  │              │
│  │(registry)│                   │  (builds)  │              │
│  └──────────┘                   └────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

1. Push code to Gitea
2. Gitea webhook triggers Drone
3. Drone spawns a build job via the Kubernetes runner
4. BuildKit builds the container image
5. Image is pushed to Harbor
6. ArgoCD deploys the new image (separate workflow)

## Gitea Setup

Gitea is straightforward - a single deployment with PostgreSQL backend. The key configuration is creating an OAuth2 application for Drone:

1. Go to Gitea Settings → Applications
2. Create new OAuth2 application
3. Set redirect URI to `https://drone.rustybower.com/login`
4. Note the Client ID and Client Secret

## Drone Components

### Drone Server

The main Drone server handles the web UI, webhook processing, and build coordination:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drone
spec:
  template:
    spec:
      containers:
      - name: drone
        image: drone/drone:2.26.0
        envFrom:
          - configMapRef:
              name: drone-env
        volumeMounts:
        - mountPath: /data
          name: drone-data
```

Configuration via ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drone-env
data:
  DRONE_GITEA_SERVER: "https://gitea.rustybower.com"
  DRONE_GITEA_CLIENT_ID: "your-oauth-client-id"
  DRONE_GITEA_CLIENT_SECRET: "your-oauth-client-secret"
  DRONE_RPC_SECRET: "shared-secret-for-runners"
  DRONE_SERVER_HOST: "drone.rustybower.com"
  DRONE_SERVER_PROTO: "https"
  DRONE_USER_CREATE: "username:rusty,admin:true"
```

### Drone Kubernetes Runner

The runner executes pipeline steps as Kubernetes pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: runner
spec:
  template:
    spec:
      containers:
        - name: runner
          image: drone/drone-runner-kube:latest
          envFrom:
            - configMapRef:
                name: drone-env
      serviceAccountName: drone-runner
```

The runner needs RBAC permissions to create pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: drone-runner
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "secrets"]
    verbs: ["get", "list", "watch", "create", "delete"]
```

### BuildKit for Efficient Builds

Instead of Docker-in-Docker (security nightmare), I use BuildKit as a separate service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: buildkitd
spec:
  template:
    spec:
      containers:
        - name: buildkitd
          image: moby/buildkit:v0.26.3
          args:
            - --addr
            - tcp://0.0.0.0:1234
            - --config
            - /etc/buildkit/buildkit.toml
          securityContext:
            privileged: true  # Required for building
          volumeMounts:
            - name: config
              mountPath: /etc/buildkit
            - name: harbor-docker-config
              mountPath: /root/.docker
```

BuildKit needs registry credentials to push images:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-docker-config
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

## Harbor Configuration

Harbor provides:
- Container image storage
- Vulnerability scanning with Trivy
- Robot accounts for CI/CD access
- Image replication between registries

Create a robot account for Drone:
1. Harbor → Administration → Robot Accounts
2. Create with push/pull permissions
3. Use credentials in Drone secrets

## Pipeline Example

Here's a real pipeline for my Hugo blog:

```yaml
# .drone.yml
kind: pipeline
type: kubernetes
name: default

clone:
  depth: 1
  submodules: true

steps:
  - name: init submodules
    image: alpine/git
    commands:
      - git submodule update --init --recursive

  - name: docker build and push
    image: plugins/docker
    settings:
      registry: harbor.rustybower.com
      repo: harbor.rustybower.com/hugo/hugo-site
      tags:
        - latest
      dockerfile: Dockerfile
      build_args:
        - HUGO_BASEURL=https://www.rustybower.com/
      username:
        from_secret: harbor_username
      password:
        from_secret: harbor_password
```

The Dockerfile uses multi-stage builds:

```dockerfile
# Build Stage
FROM hugomods/hugo:exts AS builder
ARG HUGO_BASEURL=
ENV HUGO_BASEURL=${HUGO_BASEURL}
COPY . /src
RUN hugo --minify --enableGitInfo

# Final Stage
FROM hugomods/hugo:nginx
COPY --from=builder /src/public /site
```

## Secrets Management

Drone secrets are stored per-repository:
1. Go to repository settings in Drone UI
2. Add secrets for registry credentials
3. Reference in pipeline with `from_secret`

Never commit credentials to `.drone.yml` - always use secrets.

## Troubleshooting Tips

### Build Not Starting

Check the runner logs:
```bash
kubectl logs -n drone deployment/runner
```

Common issues:
- RPC secret mismatch between server and runner
- Runner can't reach Drone server
- RBAC permissions missing

### Registry Push Fails

Verify BuildKit has credentials:
```bash
kubectl exec -n drone deployment/buildkitd -- cat /root/.docker/config.json
```

Check Harbor robot account permissions.

### Pipeline Stuck

The Kubernetes runner creates pods for each step. Check for stuck pods:
```bash
kubectl get pods -n drone
```

Look for ImagePullBackOff or resource constraints.

## Integration with GitOps

After the image is pushed to Harbor, I use Renovate to create PRs updating the image tag in my Kustomize repo. ArgoCD then deploys the new image.

The full flow:
1. **Push** code to Gitea
2. **Drone** builds and pushes to Harbor
3. **Renovate** detects new image, creates PR
4. **Merge** PR to update manifest
5. **ArgoCD** syncs new image to cluster

This keeps the GitOps principle intact - images are built automatically, but deployment still flows through git.

## Performance Optimizations

### Layer Caching

BuildKit caches layers automatically. For faster builds:
- Order Dockerfile commands from least to most changing
- Use multi-stage builds to minimize final image size
- Mount build caches for package managers

### Resource Limits

Set appropriate limits in Drone steps:

```yaml
steps:
  - name: build
    image: node:20
    resources:
      limits:
        memory: 2Gi
        cpu: 2
```

### Parallel Steps

Independent steps can run in parallel:

```yaml
steps:
  - name: test
    image: node:20
    commands:
      - npm test

  - name: lint
    image: node:20
    commands:
      - npm run lint
```

## Security Considerations

1. **Network policies** - Restrict what build pods can access
2. **Robot accounts** - Use minimal permissions for registry access
3. **No privileged unless needed** - Only BuildKit needs privileged mode
4. **Scan images** - Harbor's Trivy integration catches vulnerabilities
5. **Audit logs** - Gitea, Drone, and Harbor all log activity

## Result

My self-hosted CI/CD:
- Builds in ~2 minutes for most projects
- Zero external dependencies
- Full control over the pipeline
- Complete audit trail across all components

The initial setup is more complex than GitHub Actions, but the understanding you gain of CI/CD internals is valuable. Plus, there's something satisfying about `git push` triggering a build on your own infrastructure.
