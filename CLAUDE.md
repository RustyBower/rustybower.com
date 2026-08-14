# rustybower.com

Hugo blog (PaperMod theme). Posts live in `content/posts/`, one Markdown file each.

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
tags: ["lowercase", "kebab-case"]
---
```

## House style

Posts are technical write-ups of things actually built and run. They lean on real
output — measured numbers, verbatim console output, tables of results — rather
than description. Where a result is a ceiling, an estimate, or unvalidated, the
post says so, usually in a `## Caveats` section near the end.
