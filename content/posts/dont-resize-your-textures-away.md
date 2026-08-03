---
title: "Don't Resize Your Textures Away"
date: 2026-08-12T09:00:00-03:00
draft: false
tags: [machine-learning, computer-vision, training, preprocessing]
description: "One class was hopeless and the rest were fine. The model was never the problem — a Resize() at the top of the transform destroyed the only signal that separated them. Right diagnosis, over-applied fix."
---

A five-class material classifier reads a crop of a truck bed and calls what's in it. Four of
the five classes were fine — F1 between 0.92 and 0.97. One was hopeless:

| class                             | F1          |
| --------------------------------- | ----------- |
| coarse classes (3 of them)        | 0.92 – 0.97 |
| the coarser of two stone grades   | 0.720       |
| **the finer of two stone grades** | **0.476**   |
| macro                             | 0.791       |

The two problem classes are the same crushed stone at two grain sizes. Between 13 and 18 of 30 validation frames of the finer grade came back labelled as the coarser one. A coin flip with extra steps.

The instinct is to reach for the model: more data for that class, different loss weighting,
a bigger backbone. I tried those and it taught me nothing, which turned out to be the useful part.

## Two checks before touching anything

**Check one: how much of the deficit lives in that one confusion?** Merge the two grades into a single class and retrain as a four-class problem. Macro F1 goes from **0.791 to 0.907.**

That single number reframes the project. The classifier isn't mediocre — it's *good*, with
one specific pair it cannot separate. Nothing about the other classes, the data pipeline, the backbone or the training recipe needs attention. Whatever is wrong is wrong about
distinguishing grain size, and only that.

This is a cheap experiment and I now run it reflexively. Collapsing confusable classes bounds your problem before you start fixing it, and it's the difference between "improve the classifier" (an open-ended research project) and "make one boundary separable" (a question with a physical answer).

**Check two: kill the plausible suspect.** The class weights were stale — computed against an earlier dataset composition and never refreshed. That's a real bug and exactly the sort of thing that explains a struggling minority class. I fixed it and retrained.

The finer grade went from 0.476 to **0.372**. Macro F1 dropped. The weights were a red
herring, and finding that out cost one training run and removed the most attractive wrong answer from the board.

## What the discriminating signal physically is

At this point the question stops being about machine learning: *what, physically, tells these two apart?* Nothing about shape, colour, or context — a bed of one grade and a bed of the other are the same grey mass in the same steel box. The only difference is **grain size**.

It's a texture distinction, and texture lives in high-frequency detail at a specific pixel
scale.

Now read the transform:

```python
A.Resize(image_size)   # image_size = 224
# ... normalization, augmentation, etc.
```

The bed crops come off a full-resolution frame at around **1300 px** across. That first line
squeezes them to 224 — roughly a **6× downscale** — and it ran at both training and
validation time. The distinction between the two grades is a few pixels wide at native
resolution. Six-fold downsampling doesn't degrade it; it deletes it.

The model was never given the information. No amount of capacity, data, or loss engineering recovers a signal destroyed in preprocessing, and every experiment I might have run on the model side was doomed before the first batch.

## The fix: sample patches, don't squash frames

If the signal is fine detail, keep native pixels and give up field of view instead:

- **Training:** `RandomResizedCrop` with a scale range of 0.1–0.5, applied *straight to the
  native-resolution crop* — no leading resize. The zoom range becomes the augmentation. Each batch sees a different real patch of real pixels.
- **Validation:** reflect-pad if needed, then a center crop of a native patch. Deterministic, and crucially still native — a validation transform that quietly downscales measures a model you aren't deploying.
- **Input size 224 → 320,** for more texture and context per patch.

That last one is worth a note, because "you can't change input size after pretraining" is
common folklore. You can, when the backbone's classifier head is fed by global average
pooling — the spatial dimensions collapse before the linear layer, so there's no shape to violate. Fine-tuning above the pretraining resolution is a known lever for fine-grained
tasks, and it's free here.

**And the thing I nearly got wrong:** my first instinct was that the *detector* upstream was
the bottleneck — it runs at 320, so surely it was handing over 320-pixel crops. It wasn't.
The detector predicts boxes at its own scale, then those boxes are scaled back to native
coordinates and the crop is taken from the **full-resolution frame**. The classifier had
native pixels available the entire time and threw them away in its own first transform.

Enlarging the detector would have cost real inference budget on an edge accelerator, slowed every frame, and fixed nothing. It's the expensive plausible fix, and the only thing that ruled it out was reading the crop path end to end instead of assuming the pipeline's resolution was set by its first stage.

## The result, and the number I'm not going to quote as a headline

The finer grade went from 0.476 to 0.967. Macro F1 from 0.791 to 0.975. Twenty-nine of thirty validation frames correct, up from about half.

That number is inflated, and I know exactly by how much it isn't trustworthy: it was measured on a split that leaked by truck — [roughly 94% of validation images had another frame of the same truck pass in training](/posts/the-validation-split-that-flattered-every-number/). On a capture-grouped split, the same grade distinction scores **0.729**. Most of that 0.97 was the model recognizing which grade this site's regular trucks usually carry.

So what actually survives?

**The diagnosis survives, and it survives for a structural reason.** Both confirming checks are leakage-invariant: the merge experiment and the transform comparison used the *same* split on both sides, so the delta between them is real even when the absolute level is fiction. Downscaling was destroying the signal; native patches recovered it. That conclusion doesn't move.

**The absolute number doesn't survive**, and neither does the implied claim that the problem is solved. Grade separation on an honest split is a hard, partly-unsolved problem on this dataset — which is about 700 images, small enough that a recurring truck fleet can carry a lot of apparent skill. (A later run, after folding in human-labelled field data, reports 0.989 on this pair; it's also the least-validated model in the lineage, because the grade distinction is often not verifiable by a human from the imagery either. I'd treat that number with the same suspicion.)

## The part I got wrong: scope

Here's what a flattering validation number bought me — not just a wrong figure, but a wrong *decision* that went unexamined for weeks.

Native patches worked so well that they became the default transform for the whole
classifier, all five classes. Then field data arrived, and the comparison on real tracks
looked like this:

| transform                          | field confidence | behaviour                                  |
| ---------------------------------- | ---------------- | ------------------------------------------ |
| native 320 patch (the new default) | 0.73             | **incoherent** — empty beds called residue |
| whole-crop 224                     | 0.62 – 0.78      | coherent                                   |

Read the second column, not the first. The native-patch transform scored *higher* and was *wrong more usefully*: a context-free 320-pixel window of an empty, worn bed floor genuinely **is** residue texture. There is no information in that patch that distinguishes "empty bed, scratched" from "thin layer of fine material." The class that needs the whole-bed view had been given a keyhole.

One input view cannot serve both context and texture. That's not a hyperparameter, it's an architecture signal, and the pipeline split in two:

- a **coarse stage** on the whole crop at 224, where context is the signal — including the
  distinction between empty and not-empty;
- a **fine-grade refiner** on native 320 patches, run **only** when the coarse stage says
  this is the ambiguous stone superclass.

Native patches went from "the new default" to "correct in one gated branch." The diagnosis was right, the fix was right, and the scope was wrong — and validation at 0.975 was exactly why nobody looked. A number that good doesn't invite the question "but where does this stop applying?"

One more practical detail from that work, since it costs people real time: patches sampled from an already-downscaled stream measured **useless** (0.73 → 0.38). The native-patch trick requires a genuine full-resolution tap. A patch of a resized frame is a resized patch, and you've just paid pipeline complexity for the same destroyed signal.

## What travels

1. **Ask what the discriminating signal physically is, then check your preprocessing doesn't
   destroy it.** If the answer is "texture at a certain scale," any resize before the model is a lossy decision you're making blind. Grain size, thin fractures, small text, faint periodicity — all the same trap.
2. **Bound the problem before fixing it.** Merging confusable classes tells you how much of your deficit is one boundary. It's one training run and it can turn an open-ended project into a specific question.
3. **Kill the attractive suspect early.** Fixing the stale class weights made things worse,
   and that negative result was worth more than a successful tweak would have been.
4. **Read the pipeline end to end before optimizing the wrong stage.** The upstream component looked like the bottleneck and wasn't; the expensive fix would have bought nothing.
5. **One input view can't serve context and texture.** When two classes need different scales, that's telling you about your architecture, not your hyperparameters.
6. **A fix validated on a leaked split gets over-applied.** The leak didn't just inflate a
   number — it removed the pressure to ask where the fix stopped being right.

## Sources

- [Albumentations](https://albumentations.ai/docs/) — `RandomResizedCrop`; the scale range is
  the whole augmentation here
- Sandler et al., *MobileNetV2: Inverted Residuals and Linear Bottlenecks* (2018) — global
  average pooling in the head is what makes fine-tuning above the pretraining resolution legal
- Lin, RoyChowdhury & Maji, *Bilinear CNNs for Fine-grained Visual Recognition* (2015) — on why
  fine-grained tasks are a resolution problem before they're a capacity problem
