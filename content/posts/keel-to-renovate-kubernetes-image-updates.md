---
title: "From Keel to Renovate: Better Container Image Updates for GitOps"
date: 2026-01-17
draft: false
tags: ["kubernetes", "renovate", "keel", "gitops", "docker", "automation"]
---

For years I used Keel to automatically update container images in my Kubernetes clusters. It worked, but as I moved to GitOps with ArgoCD, Keel's push-based approach became a liability. I migrated to Renovate for PR-based image updates, and it's been a significant improvement.

## The Problem with Keel

Keel watches for new container images and updates deployments directly in the cluster. You can configure it via annotations:

```yaml
metadata:
  annotations:
    keel.sh/policy: major
    keel.sh/trigger: poll
```

When a new image appears, Keel modifies the deployment in-place.

### Why This Broke Down

1. **GitOps drift** - Keel updates the cluster, but not git. ArgoCD sees drift and wants to revert to what's in git. You end up fighting your own tools.

2. **No review process** - Updates happen automatically. A bad image goes live immediately. Rolling back means finding the old tag and manually updating.

3. **Notification-only visibility** - Keel can send Slack notifications, but there's no audit trail, no PR history, no way to see what changed when.

4. **Limited version control** - Keel's policies (major/minor/patch) are annotation-based and hard to customize per-image.

## Renovate: PR-Based Updates

Renovate takes a different approach. It scans your repository, finds container images, checks for updates, and opens pull requests. The update only happens when you merge the PR.

This fits GitOps perfectly - git remains the source of truth.

## My Renovate Setup

### Self-Hosted CronJob

I run Renovate as a Kubernetes CronJob rather than using the hosted service:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: renovate
  namespace: renovate
spec:
  schedule: "0 */4 * * *"  # Every 4 hours
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: renovate
              image: ghcr.io/renovatebot/renovate:42.19.5
              env:
                - name: RENOVATE_CONFIG_FILE
                  value: /config/renovate.json
                - name: RENOVATE_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: renovate-env
                      key: RENOVATE_TOKEN
              volumeMounts:
                - name: renovate-config
                  mountPath: /config
          volumes:
            - name: renovate-config
              configMap:
                name: renovate-config
```

Why self-hosted:
- Full control over scan frequency
- No rate limits from Renovate's hosted service
- Works with private registries
- Runs inside my cluster, close to my git server

### Configuration

The repository-level `renovate.json` controls what gets updated:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "enabledManagers": ["helm-values", "kustomize", "kubernetes"],
  "kubernetes": {
    "managerFilePatterns": [
      "/(^|/)base/.+/(?:[^/]*deployment)\\.ya?ml$/"
    ]
  },
  "prConcurrentLimit": 20,
  "prHourlyLimit": 0,
  "packageRules": [
    // ... rules
  ]
}
```

The `managerFilePatterns` is crucial - it tells Renovate to only look at deployment files in my `base/` directory. This prevents it from trying to update the same image in multiple overlay locations.

## Package Rules: The Power Feature

This is where Renovate really shines. You can define complex rules for how different images should be handled.

### Pinning Major Versions

Some apps I want to stay on a specific major version until I'm ready to upgrade:

```json
{
  "description": "Keep linuxserver/lidarr on 2.x",
  "matchDatasources": ["docker"],
  "matchPackageNames": ["lscr.io/linuxserver/lidarr"],
  "allowedVersions": "<3.0.0"
}
```

### Automerging Safe Updates

Minor and patch updates for stable images can merge automatically:

```json
{
  "description": "Automerge minor/patch for Linuxserver images",
  "matchDatasources": ["docker"],
  "matchPackageNames": ["lscr.io/linuxserver/*"],
  "matchUpdateTypes": ["minor", "patch"],
  "automerge": true,
  "automergeType": "pr"
}
```

The PR is still created (for visibility), but it merges without manual intervention.

### Handling Weird Versioning

LinuxServer images sometimes use date-based tags like `2021.12.15`. These confuse semver parsing:

```json
{
  "description": "Ignore Linuxserver date-style 2021 tags",
  "matchDatasources": ["docker"],
  "matchPackageNames": [
    "lscr.io/linuxserver/tautulli",
    "lscr.io/linuxserver/overseerr"
  ],
  "allowedVersions": "!/^2021\\./"
}
```

### Avoiding .0 Releases

Home Assistant .0 releases are often buggy. I skip them:

```json
{
  "description": "Home Assistant: avoid .0 releases",
  "matchDatasources": ["docker"],
  "matchPackageNames": ["ghcr.io/home-assistant/home-assistant"],
  "allowedVersions": "!/\\.0$/"
}
```

I also disable major/minor updates entirely for Home Assistant - I want to control those manually:

```json
{
  "description": "Home Assistant: disable major/minor updates",
  "matchPackageNames": ["ghcr.io/home-assistant/home-assistant"],
  "matchUpdateTypes": ["major", "minor"],
  "enabled": false
}
```

## The Workflow Now

1. **Renovate scans** every 4 hours
2. **PRs are created** for available updates
3. **I review** (or automerge handles it)
4. **Merge to master**
5. **ArgoCD syncs** the new image to the cluster

Everything flows through git. I can see exactly when an image was updated, who approved it, and what the previous version was.

## Dependency Dashboard

Renovate creates an issue called "Dependency Dashboard" that shows:
- Pending updates waiting for PRs
- Open PRs awaiting merge
- Updates blocked by version constraints
- Errors from failed lookups

This single issue gives you visibility into your entire update backlog.

## Migration Tips

### 1. Start with Automerge Disabled

Get comfortable with the PR flow before enabling automerge:

```json
{
  "extends": ["config:recommended", ":automergeDisabled"]
}
```

### 2. Use Semantic Commits

Enable semantic commits for cleaner git history:

```json
{
  "semanticCommits": "enabled"
}
```

PRs get titles like `fix(deps): update lscr.io/linuxserver/sonarr to v4.0.2`.

### 3. Limit Concurrent PRs Initially

Don't flood yourself with PRs on the first run:

```json
{
  "prConcurrentLimit": 5
}
```

Increase once you've caught up on the backlog.

### 4. Remove Keel Gradually

I removed Keel annotations from one app at a time, verified Renovate was creating PRs, then moved to the next. Don't rip out Keel all at once.

## Keel vs Renovate: Summary

| Aspect | Keel | Renovate |
|--------|------|----------|
| Update method | Direct cluster modification | Pull requests |
| GitOps compatible | No (causes drift) | Yes |
| Review process | None (notification only) | Full PR review |
| Version constraints | Basic annotation policies | Powerful regex rules |
| Rollback | Manual | Git revert |
| Audit trail | Logs only | Full git history |
| Automerge | N/A (always auto) | Optional per-package |

## Result

My image updates now have:
- **Visibility** - PRs show exactly what's changing
- **Control** - Rules prevent unwanted major updates
- **History** - Git log shows every update
- **Safety** - Automerge only for trusted packages
- **Consistency** - GitOps stays clean, no drift

The slight delay (PR review vs instant deploy) is worth the reliability. When something breaks, I know exactly which merge caused it and can revert cleanly.

If you're running GitOps with ArgoCD or Flux, Renovate is the right choice for container image updates. Keel solved a problem for the pre-GitOps era, but PR-based updates are the modern approach.
