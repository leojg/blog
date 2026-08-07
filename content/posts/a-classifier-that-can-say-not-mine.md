---
title: 'A Classifier That Can Say "Not Mine"'
date: 2026-08-26T09:00:00-03:00
draft: true
tags: [machine-learning, computer-vision, classification, production-ml]
description: >-
  Almost all classification writing assumes a closed set. In production you need
  three answers, not two — and "wrong family entirely" is a different failure from
  "unsure which one".
---

A camera looks down at a road. A truck passes underneath and the system has to say what
it's carrying. Some materials are coarse families that split into commercial grades; the
grades matter for billing, and they look similar enough that a single flat classifier over
every leaf class does poorly.

So it's staged: a coarse model picks the family, then a specialist refiner splits that
family into products. Standard hierarchical classification.

The part worth writing about is what the refiners are allowed to say.

## Three answers, not two

The obvious design gives a refiner two outputs — grade A or grade B — plus a confidence
threshold. Below the threshold you decline and report the coarse family.

That covers one failure. It misses another, and they are not the same:

- **Low confidence:** *"this is in my family, I can't tell which product."*
- **Reject:** *"this isn't my family at all."*

The second happens whenever the coarse stage misroutes, which it does. A load that isn't
the family gets handed to a specialist trained only on that family's grades. A two-output
refiner has no vocabulary for "you've sent me the wrong thing" — it must return one of its
grades, and softmax will happily produce a confident one. The specialist's confidence is
about *which grade*, conditioned on an assumption that just turned out false.

So each refiner carries an explicit **reject class** alongside its products. It can answer
"neither of these".

## What each answer does

Both non-answers land in the same place, by design:

| Refiner says | Reported | Queued for review |
|---|---|---|
| a product, above the gate | that product | no |
| a product, below the gate | the coarse family | yes |
| reject, at any confidence | the coarse family | yes |

Falling back to the coarse family is the important bit. The system doesn't discard the
pass, and it doesn't guess. It reports the thing it does know — the family — and flags the
load so a person decides the product. **A product name is never guessed**, because a
product name is what gets billed.

Two real passes from a recent evaluation:

- One returned **reject at 0.66**. Not "unsure between the grades" — "this doesn't belong
  to me". Reported as the bare family, flagged.
- Another returned the **correct product at 0.60**, under the gate. The right answer, not
  confidently enough. Also reported as the bare family, also flagged.

Identical outcome, opposite reasons. That's fine — the outcome is deliberately the same.
But they're distinguishable in the logs, and that distinction is what tells you whether to
retrain the coarse stage or the refiner.

## Why the distinction earns its keep

Collapse both into "low confidence" and you lose the diagnostic. A rising reject rate
means the coarse stage is misrouting — its problem, not the refiner's. A rising
below-gate rate on correctly-routed loads means the refiner needs more data or the grades
aren't separable at your capture quality. Same symptom at the queue, completely different
fix.

There's a second benefit that shows up in demos. When you say *"and when it isn't sure, it
doesn't guess"*, you can point at the mechanism rather than assert a policy. Technical
buyers reliably respond better to a system that declines than to one claiming it never
errs — the first is checkable.

## The cost

Roughly a third of one recent sample got flagged rather than auto-resolved. That is a real
operational cost: somebody looks at those.

It's the right trade when a wrong product name is expensive to unwind and a flagged load is
cheap to resolve. Invert that — high volume, low unit value, no billing consequence — and
you should probably take the guess and accept the error rate. The pattern isn't free and
isn't universal.

## What I'd tell someone building this

**Give every specialist a reject class**, even where the router "shouldn't" misroute.
Yours will.

**Make the reject class semantic, not a threshold artifact.** A learned reject sees actual
out-of-family examples during training. A threshold on the max softmax is a proxy, and a
poor one — networks are routinely confident on inputs unlike anything they trained on.

**Route both non-answers to the same fallback, and log them differently.** The user-facing
behaviour should be identical; the operator-facing signal should not be.

**Decide what "I don't know" outputs before you tune anything.** In a closed-set benchmark
the answer is "the argmax anyway", which is why closed-set benchmarks make poor guides for
production. The interesting engineering is entirely in what happens when the answer is
none of the above.
