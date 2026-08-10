---
title: '"Verify the Mechanism Is Firing" — It Had Never Been Deployed'
date: 2026-08-15T09:00:00-03:00
draft: false
tags: [deployment, verification, data-pipelines, postmortem]
description: >-
  A task framed as "check why this misbehaves" found the mechanism had been built
  months earlier and never shipped. Every central assumption in the spec was
  falsified by looking at real data instead of at the repository.
---

A client operates two businesses under one legal entity, at two addresses. Supplier
invoices arrive by email and have to be attributed to the right one. The mechanism to do
that had been designed, written and documented. The task on my desk was a diagnostic:
*confirm it's firing, and if it's mostly falling through to the AI fallback, find out why.*

The spec proposed a sensible first step — count how many invoices were resolved by the
database lookup versus guessed by the model. I ran it. The answer was zero and zero.

The workflow running in production was an older version with no classification in it at
all. The version with the mechanism existed only as a file in a repository. Nobody had
deployed it.

## One API call would have replaced the entire spec

The spec was several pages. It reasoned carefully about why the lookup might be
under-matching, proposed a diagnostic, and laid out two phases of remediation. All of it
rested on the assumption that the thing was running.

Checking that assumption was one authenticated request against the platform's own API:
list the active workflows, look at which version is live. Under a minute. It would have
invalidated the document before I wrote it.

The failure wasn't laziness — it was that the repository *looked* like the source of
truth. The file was there, it was committed, its logic was correct, and a colleague could
read it and reasonably conclude the system behaved that way. Deployment state lives
somewhere else, and nothing in the repository tells you what that somewhere else
currently holds.

## Then the second assumption fell over

With the thing actually deployed, the spec's diagnosis was next. It assumed the matcher
under-performed because supplier invoices frequently omit the buyer's address.

Measured across roughly 950 real invoices: the address was present on **98%** of them.

The problem wasn't absence, it was rendering. The same premises appeared a dozen different
ways — abbreviated street names, the word for "street" present or absent, the door number
in three positions, trailing city names duplicated. Substring matching resolved 28%. A
matcher keyed on a distinctive street word plus the door number reached 53%. Adding cross-
street tokens took it to 55%, with zero ambiguous cases and no regressions.

A remediation built for the assumed failure mode — "handle missing addresses" — would have
addressed 2% of the corpus.

## And the third

The plan for the remaining residue was to derive a supplier-to-business map from history:
this supplier only ever sells to that business, so future invoices from them classify
themselves.

It covers **40 of 428** unresolved invoices.

The reason is a correlation nobody would guess from a whiteboard: **address quality and
address junk are anti-correlated**. Suppliers who print clean addresses already resolve
without any map at all. The suppliers whose invoices need the map are exactly the ones
emitting placeholder garbage every single time — so there is no clean historical record to
derive the mapping *from*. The evidence you need and the problem you have occupy disjoint
sets.

The residue turned out to be mostly fuel, banking and telecoms. Those are genuinely
*shared* costs. Attributing them to one business wouldn't be uncertain, it would be wrong.
The right answer was to leave them unattributed and say so.

## The pattern

Four assumptions, each reasonable, each falsified by data that already existed:

| Assumed                              | Actual                                  |
| ------------------------------------ | --------------------------------------- |
| The mechanism is running             | Never deployed                          |
| Addresses are often missing          | Present on 98%, rendered inconsistently |
| History can bootstrap a supplier map | Covers 9% of the gap                    |
| The residue is a data problem        | The residue is genuinely shared costs   |

None of those required new instrumentation. The deployment state was one API call. The
address statistics were one query against a spreadsheet that had been accumulating for
months. The supplier coverage was a join. All of it was sitting there while the spec was
being written from the repository and from reasoning.

## What I do differently now

**Before writing a spec about why something misbehaves, establish that it runs.** Not "the
code is correct" — that it is deployed, active, and the version you think it is. Write the
observed state into the spec's opening lines, with the command that produced it. If you
can't produce that line, the spec isn't ready.

Two weeks after this, I hit the same thing from the other direction: I put confidence
figures on screen from a machine-learning model that my notes described as deployed. The
config the device actually reads pointed at a newer stack — with an extra classifier I
didn't know existed. Same class of error, different artifact. Documentation had drifted
from deployment, and I trusted the documentation because I wrote it.

Repositories, notes and specs all describe intent. Only the running system describes
behaviour, and it is almost always cheaper to ask it than to reason about it.
