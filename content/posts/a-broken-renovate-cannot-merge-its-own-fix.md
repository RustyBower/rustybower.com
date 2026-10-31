---
title: "A Broken Renovate Can't Merge Its Own Fix"
date: 2026-12-29
draft: false
categories: ["gitops", "kubernetes", "debugging"]
tags: ["renovate", "cronjob", "automerge"]
---

Renovate keeps about 60 apps in my cluster up to date, and that includes Renovate. It runs as a Kubernetes CronJob every two hours and automerges its own minor version bumps. This works well right up until it merges a release that stops it from running, at which point nothing can undo that merge automatically.

## What happened

At 05:06 UTC on October 15, Renovate merged its own bump to 44.161.1. From then on, every run aborted before it cloned the repo:

```
(node:9) TimeoutNegativeWarning: -1000 is a negative number.
...
fatal: could not read Username for 'https://github.com': No such device or address
```

The run's result was `external-host-error`. There were no new pull requests for about 40 hours, which would be around 20 failed runs. Nothing alerted, because a dependency bot going quiet looks just like a week with no new releases.

## The misleading error

`could not read Username` says "bad token" about as clearly as an error can. Git wanted credentials, didn't get any, and tried to prompt in a container with no terminal.

The token was fine. It had `repo` scope and no expiry, and nothing about it had changed. The useful clue was timing: the failures began at the first run after Renovate had updated itself.

The negative timeout warning was a second hint, because it had nothing to do with authentication. I'd want to rule out the change that just merged before suspecting a credential that hasn't changed.

## Why it can't fix itself

Normally a bad release is someone else's problem for a few days. A newer version comes out, Renovate opens a PR, and CI passes. Here, the newer version was out, and the only thing that could have opened the PR was the thing that was broken.

So the recovery has to be manual. To avoid guessing at another version, I rendered a one-off Job from the CronJob with a newer image swapped in:

```bash
kubectl -n renovate create job --from=cronjob/renovate test-run \
  --dry-run=client -o yaml \
  | sed 's#renovate:OLD#renovate:NEW#' \
  | kubectl apply -f -
```

That proved 44.165.1 worked before anything went into Git. I then bumped the image by hand, and it merged on October 16 at 21:40 UTC. That restored it.

## The second time it went quiet

This wasn't the first time Renovate had silently stopped doing a job. In July I found 13 minor and patch PRs sitting open with `automerge: true` configured and all checks passing. Nothing was wrong with them. GitHub's native automerge isn't available on private repos on the free plan, so the merge never happened and nothing complained. Setting this makes Renovate do the merge itself:

```json
{
  "platformAutomerge": false
}
```

Both failures had the same shape. Your updates just stop arriving, and that absence doesn't trigger anything.

## What to check when it goes quiet

If Renovate's PRs or its dashboard issue stop updating:

1. Read the `"result"` line from the most recent job's logs.
2. Compare the first failure with the last Renovate image bump.
3. Test a newer image as a one-off Job.
4. Only then look at the token.

## Caveats

- **I never found the upstream cause.** I know 44.161.1 failed and 44.165.1 worked, but I didn't work out which change did it. "A release regression" is an inference from that, plus the token being unchanged.
- **This can happen again.** Renovate still automerges its own bumps, and no alert watches for it going quiet. The fix is a Prometheus rule on the time since the last successful run, and I haven't written it.
- **The 20 runs is arithmetic.** That's 40 hours at one run every two hours, not a count from the job history.
