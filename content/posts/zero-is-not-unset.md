---
title: "Zero Is Not Unset"
date: 2026-08-19T09:00:00-03:00
draft: false
tags: [api-design, data-modelling, configuration, bugs]
description: >-
  A form field prepopulated with 0 silently shadowed every rate below it in an
  inheritance chain. The bug wasn't the zero — it was that the chain had no way to
  distinguish "the answer is zero" from "I have no answer".
---

A cost model resolves an hourly rate by walking a chain: the machine's own override, then
its template, then the site, then the organisation. First non-null wins. Standard
inheritance, the same shape as CSS cascades, config layers or permission scopes.

A UI change prepopulated numeric fields with `0` so the form never showed an empty box.
Reasonable-looking polish.

From that moment, every machine created from a template resolved to a rate of zero. Not
because anything failed — because the chain worked exactly as designed. The template now
*had* a value. Zero is a value. The lookup stopped there and never consulted the site or
organisation rates underneath.

Nothing errored. Nothing logged. Costs computed cleanly and came out as zero.

## Why this is worse than a null-pointer bug

A missing value announces itself. You get an exception, a stack trace, a line number. This
does the opposite: it produces a *plausible* answer through a *correct* code path.

Worse, zero is semantically legitimate here. Some machines genuinely cost nothing per hour
— fully depreciated, borrowed, or not tracked. So you cannot fix it by treating zero as
falsy. That would break the real case and, in a chain, "sometimes falsy" is arguably worse
than "always truthy" because now the behaviour depends on the value rather than on the
schema.

The actual defect is upstream of both: **the field could not represent "I have no
opinion".** Once a form guarantees every field is populated, an inheritance chain has
nothing left to inherit. The chain still runs — it just always terminates at the top.

## The fix, and why it feels wrong at first

Revert the prepopulation. Make the field **required and blank**.

That reads as worse UX. The user now faces an empty box and must type something, including
typing `0` when they mean zero. But that keystroke is the entire point: it's the difference
between a stated zero and an accidental one. An explicitly entered `0` is a decision. A
prepopulated `0` is a default masquerading as one.

**Required-with-explicit-zero** is the pattern. If the value participates in a fallback
chain, refuse to guess it on the user's behalf.

Where a blank field genuinely isn't acceptable, the alternative is to make "unset" a real
state in the model rather than a convention — a nullable column, an explicit `inherit`
sentinel, a tri-state control. What doesn't work is hoping that a magic value will be
recognised as absence, because every consumer downstream has to remember the convention,
and one of them won't.

## Where else this hides

The shape recurs anywhere a default feeds a resolution chain:

- **Config layers.** A framework default of `0` for a timeout shadowing an environment
  override — the request hangs forever, or doesn't wait at all, depending on whose zero
  wins.
- **Feature flags.** `false` as a default is indistinguishable from `false` as a decision,
  so a rollout can't tell "not yet configured" from "explicitly off".
- **Pricing and discounts.** A 0% discount stored eagerly blocks an inherited promotion,
  and the customer sees full price for reasons nobody can reconstruct.
- **Permissions.** An empty grant list meaning "deny everything" versus "defer to the
  parent role" — same data, opposite security posture.

In each case the same question decides it: **can this layer say "I have no opinion", and
does the value chosen to mean that also mean something real?** If yes to the second, you
have this bug already; it just hasn't produced a wrong number that anyone noticed yet.

## The check worth adding

When reviewing anything with a fallback chain, ask what happens when a middle layer holds
the type's zero value — `0`, `""`, `false`, `[]`. If the answer is "it stops there", verify
that stopping is what you meant. Half the time the layer got that value from a default
rather than a decision, and nobody will notice until a total comes out wrong in a report
that nobody reconciles.

The bug that started this was a one-line UI convenience. It took a week to surface, and it
surfaced as "the costs look wrong" — which is the most expensive way to find anything.
