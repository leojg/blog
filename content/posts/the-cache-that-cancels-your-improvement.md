---
title: "The Cache That Cancels Your Improvement"
date: 2026-08-29T09:00:00-03:00
draft: true
tags: [caching, llm, data-pipelines, debugging]
description: >-
  I improved a matching pipeline, measured the improvement, deployed it, and
  reprocessed the backlog. The backlog reproduced every old decision exactly. The
  cache was keyed on the input and stored the verdict.
---

A batch job matches free-text line items against a large product catalogue. Supplier
wording never matches catalogue wording, so it runs a cascade — remembered aliases, then
embedded codes, then exact matches, then fuzzy retrieval, and finally one language-model
call to adjudicate the survivors. That last step costs money, so results are cached.

I improved the text normalisation feeding the cascade. Tested it properly: extracted the
production matching code, ran it against the real catalogue, confirmed **zero** previously
working matches broke and roughly **230** previously failing lines now cleared the
acceptance gate.

Deployed it. Reran the historical backlog to propagate the gains.

The backlog came back with exactly the decisions it had before.

## The cache stored the wrong thing

The cache key was the input text. The cached value was the **fully adjudicated,
post-gate result** — the final verdict, after normalisation, retrieval, the model call and
the acceptance gate.

So for every line the job had seen before, the pipeline returned the stored verdict and
never executed any of the improved logic. The improvement was real for new lines and
structurally invisible for old ones. The reprocessing run wasn't recomputing anything; it
was replaying a decision log at some expense.

Nothing failed. There was no error, no warning, no metric that moved. A reprocess that
changes nothing looks identical to a reprocess that confirms everything was already
correct — which is exactly the pleasant conclusion you'll reach if you don't check.

## The general rule, and why it keeps getting broken

**A cache key must cover everything that can change the value.** Here the value depended on
the input text *and* the normalisation code *and* the retrieval index *and* the gate
threshold *and* the model version. Only the first was in the key.

Everyone knows this rule. It gets broken anyway, and I think the reason is that the other
inputs don't look like inputs. They look like *the program*. When you write
`cache[text] = result`, the text is obviously data and the code is obviously not — until
you change the code and expect old entries to reflect it.

The tell is the question **"what would make this entry wrong?"** If the answer includes
anything you deploy, the key is incomplete.

## Caching the answer versus caching the expensive part

The deeper mistake was caching at the wrong layer.

The expensive step was the model call. Cache *that* — key it on the exact prompt, store the
raw response — and improving normalisation naturally produces a different prompt, hence a
miss, hence fresh work only where the change actually mattered. Retrieval and gating stay
cheap and always run.

Instead the cache wrapped the whole decision. That saves marginally more on a hit and makes
every downstream improvement unreachable. Cache the costly leaf, not the composed verdict.

## It was the second time

The same system had already been bitten by a cache outliving a change to its inputs, in a
different subsystem, months earlier. Same class, different surface, and nobody connected
them until both were written down next to each other.

That's the part I keep thinking about. A lesson learned in one subsystem doesn't propagate
to the next one by osmosis. It propagates when somebody writes it somewhere a future
reader will hit — and "somewhere" almost never means the commit message on the original
fix.

## What it costs to fix, and what I did instead

Propagating the improvement retroactively means invalidating the affected entries and
paying for the model calls again. That's a real, quantifiable spend against a benefit
measured in a few hundred corrected historical rows.

I documented it and didn't run it. The improvement is live for everything new; the
historical set stays as it was, with a note saying why. That's a defensible call — but only
because it's *recorded* as a decision. The failure mode isn't declining to spend the money.
It's believing the improvement already propagated because the reprocess job exited zero.

## Three checks worth stealing

1. **Add a version component to the key** — a hash of the code path, or a manually bumped
   constant you're disciplined about. Cheap, ugly, works.
2. **Cache the expensive leaf, not the composed decision.** If a cached value embeds
   judgement, improving the judgement can't reach it.
3. **When a reprocess changes nothing, treat that as a finding, not a result.** Sample a
   handful of entries and confirm the new code actually ran. "No diff" is what success and
   total no-op look like, and they are indistinguishable from the exit code.
