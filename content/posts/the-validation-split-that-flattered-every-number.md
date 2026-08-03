---
title: "The Validation Split That Flattered Every Number"
date: 2026-08-09T09:00:00-03:00
draft: false
tags: [machine-learning, computer-vision, evaluation, mlops]
description: "94% of my validation set was in training under a different filename. Fixing it dropped the numbers, the field benchmark said the leaked model was still better — and the field benchmark was broken too."
---

The classifier reported a macro F1 of **0.975**. It was, in the narrow sense, telling the
truth: that number came out of a validation set the model never trained on, computed
correctly, reproducibly.

What it measured was *"a new frame of a truck I have already seen."* Roughly **94% of the
validation images had another frame of the same truck pass sitting in the training set.**
Different filename, different moment, same vehicle, same load, same lighting, half a second apart.

That's a leak, it's the most common leak in video-derived datasets, and it's not the
interesting part of this story. The interesting part is what happened over the following
week, when I fixed it and then tried to find out how much it had mattered.

## Why video leaks by default

The pipeline is two stages: a detector finds the truck bed in a frame, a classifier reads
the crop and calls the material. Both train on frames extracted from recorded passes.

A truck pass yields dozens of frames. Consecutive frames of one pass are near-duplicates —
same bed, same material, same sun angle, a few pixels of motion apart. Split those frames
randomly by image and you have effectively copied most of your validation set into training.
The model doesn't need to generalize to a new truck; it needs to recognize a truck it
memorized, from a slightly different angle.

The mechanically annoying part is that **nothing about this looks wrong.** The split code
was correct. The images in validation genuinely were not in training. There's no assertion
to write, no leak detector to run — the leak lives entirely in the fact that "image" is the
wrong unit.

The fix is to group by whatever unit shares appearance, and split *groups*:

- Derive a capture ID from the filename by stripping the trailing frame number.
- Merge duplicate exports of the same capture — a recording exported twice ends up as `<id>` and `<id> (1)`, and both halves have to land on the same side.
- Move every image in a group together, with a stratified group split so class balance
  survives.

What made this genuinely embarrassing: **the detector's data module had done exactly this for months.** Capture-grouped splitting, the duplicate-export merge, all of it — sitting in the same repository, one directory over. The classifier's data module had been written later and simpler, and nobody went back to check whether the older component knew something. The fix wasn't research. It was reading my own code.

## The honest numbers are worse, and one of them is a tell

Retraining on capture-grouped splits, where no truck pass straddles the boundary:

| run                       | design               | honest val F1 | best epoch |
| ------------------------- | -------------------- | ------------- | ---------- |
| coarse, 4-way             | whole-crop input     | 0.945         | 3          |
| coarse, 4-way, full crops | whole-crop input     | 0.902         | **0**      |
| 5-way                     | high-res patch input | 0.928         | 17         |
| fine-grade refiner, 2-way | high-res patch input | **0.729**     | 0          |

That last row is the one that hurt. The two hardest classes are two grades of the same
crushed stone, distinguished only by grain size; under the leaky split that pair scored
around 0.97. Honestly split, **0.729**. Most of the apparent skill was memorizing which
grade this site's recurring trucks usually carry.

Then there's the epoch column, which I'd have skimmed past a month earlier. Two runs peaked
at **epoch 0** — best validation score after a single pass through the data, never improved
again. On a leaked split, "improvement" over later epochs is partly the model getting better at recalling specific trucks, and grouped validation simply doesn't reward that. When your best checkpoint is epoch 0, the training loop isn't learning anything your validation set considers generalization.

That's a diagnostic worth keeping: **the epoch at which validation peaks tells you something about your split, not just your schedule.**

## Layer three: the benchmark that was supposed to settle it

So the honest numbers are lower. Fine — that's expected, that's the whole point of fixing a leak. The real question is which model to *deploy*, and for that I had a field benchmark:
twenty tracked sessions from the live rig, with the material each one was recorded as.

The benchmark said the leaked model was better. Comfortably: median confidence 0.78 against 0.58 and 0.30 for the honestly-split retrains, with more coherent predictions.

There is a seductive story available here, and I believed it for about a day. It goes:
*the leaked model memorized the site's truck fleet, and the site keeps sending the same
trucks, so memorization is a legitimate deployment advantage.* It's plausible. It's the kind
of thing that gets written up as a counterintuitive lesson.

It was wrong, for two independent reasons, either of which alone invalidates the benchmark:

1. **The labels were circular.** The material recorded against each field session had been
   produced by the deployed model — the leaked one. A model that disagrees with it is scored
   as wrong *by construction*. I wasn't measuring accuracy, I was measuring agreement with
   the incumbent, which is a metric the incumbent wins by definition.
2. **A timebase bug** was clamping every lookup into those clips to the end of the clip, so a good fraction of the comparisons weren't reading the frames they claimed to. (That one has its own story, and its own set of lessons about mixing clock domains.)

Two broken benchmarks, discovered in the same week, both producing confident, plausible,
directionally-wrong verdicts.

## What human labels said

The only fix was ground truth that didn't come from a model. A person went through
filmstrips of all 177 routed sessions and marked what was actually there.

The first finding had nothing to do with model ranking and mattered more than the ranking
did: of the 108 sessions that contained a real truck, **the single most common real load was
something the class list didn't have a name for** — off-catalog material, mostly rock rubble,
in 48 of them. Nearly half the real traffic was a category the taxonomy denied existed.

With human labels in place, the field comparison reversed:

- The honestly-split model with an explicit off-catalog class: **45%** on real beds.
- The leaked model: **30%**.
- On the catalog classes alone the two were **identical, at 55%.**

The entire gap was the off-catalog loads. The leaked model didn't merely misclassify them —
it *invented a product* for every one, because a five-way softmax has no way to say "none of
these." The advantage the broken benchmark had credited to memorization was, when measured
against reality, a deficit caused by a missing class.

The rest was ordinary work: harvest 104 human-labeled crops from the marked seconds, fold
them into training with the off-catalog class included, retrain. Honest validation F1 0.917,
and held-out field accuracy from roughly 45% to **73%**.

## What I take from it

**An image-level split on video data is a leak by default, not a corner case.** If frames
come from a continuous capture, group by capture. The same applies anywhere your rows share
a hidden identity — multiple scans of one patient, several sessions from one user, repeated
photos of one item. Ask what unit shares appearance and split on that.

**Relative comparisons survive leakage; absolute numbers don't.** Two variants measured on
the same leaky split can still be compared — that's how I know a change I made around the
same time was a genuine improvement, even though its headline number was inflated. What you
cannot do is quote the absolute figure as a deployment expectation, or compare across splits.

**A leaked number doesn't just overstate your model — it hides your mistakes about it.**
While validation read 0.975, there was no pressure to ask whether any design decision was
over-applied, whether the class list covered reality, whether the field behaviour matched
the lab. A number that good ends inquiry. That's the real cost, and it's much larger than
the delta between 0.975 and 0.917.

**If your ground truth came from the model you're evaluating, you own a mirror, not a benchmark.** This one is worth a standing rule: before a benchmark is allowed to arbitrate anything, write down where each label came from. If the answer is "the system under test," stop. Everything downstream is agreement, dressed as accuracy.

**And the number going down is the deliverable.** 0.975 → 0.917 looks like a regression in a report and is the single most valuable thing that happened to this project, because everything built on top of the honest number has held up since, and nothing built on the flattering one did.

## Sources

- Scikit-learn's [`StratifiedGroupKFold`](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.StratifiedGroupKFold.html)
  — grouped splitting with class balance preserved
- Kapoor & Narayanan, *Leakage and the Reproducibility Crisis in ML-based Science* (2022) — a
  survey of how often this exact failure reaches publication, across a lot of fields that
  should know better
