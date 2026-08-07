---
title: "One Substrate, Two Renderers"
date: 2026-08-22T09:00:00-03:00
draft: true
tags: [meta, agents, writing, documentation, process]
description: >-
  The pipeline that turns a production system into a demo video and the one that
  turns postmortems into blog posts are the same machine. Only the last step
  differs, and it's the cheap step.
---

I built a tool that generates demo videos from a running system, then decided not to
productise it. Separately, I keep a backlog of blog posts seeded from postmortems. It took
me an embarrassingly long time to notice these are the same pipeline with different output
formats.

## The shape

Both do this:

```
source of truth  →  verify the claim  →  scrub what identifies the client
                 →  structure  →  publish
```

For a video, that reads: query the production database and run the deployed models →
confirm every number can be re-derived by a command → replace client identities with
stable pseudonyms and prove nothing leaked → lay out scenes and narration → render an MP4.

For a post: read the postmortem → confirm the finding actually happened with those numbers
→ strip the client, the supplier names, the infrastructure identifiers → structure the
argument → publish markdown.

Same four steps. The fifth — ffmpeg versus a static site generator — is the only part that
differs, and it's the part that took an afternoon.

## The equivalences are exact, not loose

This isn't an analogy that falls apart under pressure. The mechanisms map one to one.

**The scrub gate is the same gate.** My postmortem template carries a `Publishable`
verdict: `yes`, `no`, or `with edits — name exactly what needs reworking`. The video
pipeline has a claim ledger that refuses to render anything it can't source. Both are a
hard stop between "true internally" and "safe to show outsiders", and both fail closed.

**The abstraction marker is the same marker.** My post backlog tags every candidate `(D)`
direct — write about the thing itself — or `(I)` indirect — extract the generalisable
lesson, strip the specifics. Client work is almost always `(I)`. That is precisely the
anonymisation step in the video pipeline: keep the magnitudes, replace the identities.
When I wrote "a client operates two businesses under one legal entity" instead of naming
them, I was running `redact.map` by hand.

**The verification discipline is the same discipline.** A video claim must survive a
prospect with a calculator. A post claim must survive a reader who works on the same
problem. Neither survives "I remember it being about 90%".

## What that reframes

I'd been treating the video work as a tooling project and the blog as a writing habit.
They're two renderers over one substrate, and the substrate is the scarce thing:

> **verified, scrubbed, generalisable findings drawn from a system you actually operate**

Each clause is load-bearing. *Verified* — someone will check. *Scrubbed* — it's not yours
to publish otherwise. *Generalisable* — a war story about your specific bug helps nobody.
*A system you actually operate* — this is the one that can't be bought, borrowed or
prompted into existence.

Most technical content fails on the last clause. Toy examples, tutorial repositories,
reasoning from documentation. Not because the authors are lazy, but because operating
something real and being willing to write about its failures are rarely the same person's
week.

## Why I didn't sell the tool

When I worked through productising the video generator, the numbers looked fine — a couple
of dollars of compute per video against a thousand-plus for the freelance alternative. The
code was about 1,600 lines of glue over public components. Two days of work.

Which is exactly the problem. Two days means anyone competent rebuilds it in two days. And
the local-execution property that makes it trustworthy — your production database never
leaves your machine — also makes it unmeterable and gives it no cost to mark up.

But the deeper reason is this framing. **The renderer was never the scarce part.** Selling
a renderer to someone without the substrate produces confident, wrong output faster than
before. The tool amplifies judgement; it does not supply it.

So the durable artifact wasn't the code. It was the procedure, written down: ground before
you narrate, build the ledger before you write, prove the anonymisation rather than
asserting it, say only the number a reader can count for themselves.

## The practical consequence

Once you see it as one pipeline, the fix for a gap in one half applies to the other.

My post backlog was built by periodic manual sweeps of the documentation tree. Anything
written *after* a sweep stayed invisible until the next one — which had quietly hidden five
publishable findings for weeks. The fix was to make the postmortem step seed the backlog
directly when it writes a publishable verdict, so the substrate feeds the renderer
automatically instead of waiting for me to remember.

That's the same instinct as the claim ledger: don't rely on the operator noticing. Make
the pipeline carry it.
