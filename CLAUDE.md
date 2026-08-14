# rustybower.com

Hugo blog (Blowfish theme, `ocean` colour scheme). Posts live in `content/posts/`,
one Markdown file each.

**Blowfish requires Hugo ≥ 0.158.0.** The in-pod builder image is pinned in the
`kustomize` repo at `environments/rustycloud/apps/hugo/deployment.yaml` — bump it
there before any theme update that raises the floor, or the site fails to build
with no CI to catch it.

## Publishing cadence — posts are queued bi-weekly

New posts are **not** published the day they're written. They go into a queue and
release **one every two weeks, on Tuesdays**.

The rhythm is anchored to 2026-08-25: valid slots are that date and every
14 days after it (09-08, 09-22, 10-06, ...). Posts before 2026-08-25 were on a
weekly cadence, so don't infer the interval from older dates.

The queue is just the `date:` field in the front matter. Hugo excludes
future-dated posts from the build, so a post with a future date sits in the repo
committed and invisible until its date arrives.

When adding a post:

1. List the existing `date:` values (`grep -H "^date:" content/posts/*.md`) and
   find the last queued date.
2. Set the new post's `date:` to the **next open slot** — 14 days after the last
   queued one. Not today's date, and not a date already taken.
3. Leave `draft: false`. The future date does the holding, not the draft flag.
4. Commit and push to `master`.

Backdating a post to today publishes it immediately and jumps the queue. Only do
that if asked for explicitly.

**Re-check the dates immediately before pushing.** Inserting a post sometimes
means shifting everything after it back a slot instead of appending to the end,
so the slot that was free when you started may be taken by the time you push.
Always `git fetch` and re-read the queue before picking a date.

## Deployment

No CI pipeline. The site runs on K3s: git-sync polls `master` every 30 seconds,
Hugo rebuilds in-pod, nginx serves the output. Pushing to `master` is the deploy
— which is why queueing is done with dates rather than by holding commits back.

## Front matter

```yaml
---
title: "Sentence-case Title, Fairly Specific"
date: 2026-09-01
draft: false
categories: ["kubernetes", "homelab", "debugging"]
tags: ["k3s", "sqlite", "kine"]
---
```

Note that several posts use CRLF line endings. Edit front matter in place rather
than rewriting whole files — normalising the endings shifts byte offsets and has
previously tripped a `slice bounds out of range` panic in Hugo's `.Summary`.

## Taxonomy — `categories` are closed, `tags` are open

Every post carries **both** fields, and they do different jobs.

### `categories` — broad topics, closed set

The navigation backbone. Pick **2–4** from this table and nothing else:

| Category | Covers |
|---|---|
| `kubernetes` | K3s, kine, etcd, cluster internals |
| `homelab` | self-hosted infrastructure generally |
| `home-automation` | Home Assistant, Z-Wave, HVAC, AppDaemon, tablets |
| `python` | Python tooling, Textual/TUI apps |
| `gitops` | ArgoCD, Kustomize, Keel, Renovate, CI/CD, registries |
| `llm` | LLMs, ML, computer vision, OpenCV, Coral TPU |
| `networking` | DNS, Pi-hole, MetalLB, BGP, UniFi |
| `data` | DuckDB, Parquet, Jupyter, statistics, datasets |
| `debugging` | post-mortems and root-cause write-ups |
| `photography` | Immich, photo pipelines, scanning |
| `audio` | whole-home audio, Music Assistant, Subsonic |
| `dashboards` | Homepage, dashboard UIs |

**Never invent a category.** A post that seems to need a new one almost always
fits an existing one. Adding to this table requires two-plus posts that need it,
and the table must be updated in the same commit — single-use entries are how the
old flat tag list grew to 71 entries across 18 posts, 52 used exactly once.

### `tags` — product and tool names, open set

Specific things a reader would search for by name: `frigate`, `immich`,
`birdbuddy`, `argocd`, `duckdb`, `metallb`. Add new ones freely — a tag used by
one post is fine here, because this is a lookup index, not navigation.

Tag what the post is genuinely *about*, not everything it mentions in passing.
Two to four is typical.

Both fields are lowercase kebab-case YAML lists.

### Rendering

`/categories/` and `/tags/` both render as a pill cloud via
`layouts/partials/term-link/text.html` and `layouts/_default/terms.html`, which
override the theme's sparse default grid. Styles are in `assets/css/custom.css` —
Blowfish ships a **prebuilt** Tailwind bundle, so new utility classes have no CSS
behind them and anything custom must be written as plain CSS against the scheme's
`--color-*` variables.

## House style

Posts are technical write-ups of things actually built and run. They lean on real
output — measured numbers, verbatim console output, tables of results — rather
than description. Where a result is a ceiling, an estimate, or unvalidated, the
post says so, usually in a `## Caveats` section near the end.
