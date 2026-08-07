---
title: "The Demo Video Was the Easy Part"
date: 2026-08-08T09:00:00-03:00
draft: false
tags: [agents, llm, video, data-quality, economics]
description: >-
  A margin that was arithmetically correct and substantively false, confidence
  scores from a model that wasn't deployed, and one in five invoice lines failing
  its own arithmetic. Generating the video took minutes. Making it true took two
  days.
---

I needed demo videos for two pre-revenue products: one that reads supplier invoices and
computes per-product margins, one that watches trucks leave a quarry and classifies what
they're carrying. No budget for an agency. So I built them with an agent, ffmpeg, a
headless Chromium and a free text-to-speech voice.

The rendering pipeline took an afternoon. Everything else took two days, and none of it was
video work.

## The problem isn't generation

Script-to-video is a solved commodity. Synthesia will turn text into a presenter for about
USD 18/month. Arcade will turn a screen recording into a guided demo for USD 32. Both are
good at what they do, and neither touches the part that actually costs anything.

Because the question a demo video has to survive isn't *"does this look professional?"* It's
*"is that number true?"* — asked by a prospect with a calculator, three weeks after you sent
it, with your credibility as the stake.

Every tool in that market is downstream of content. They assume you already have the
footage, the figures and the story. Producing those was the entire job.

## Four numbers that were true and wrong

**One.** The margin report showed a product selling at a 22% loss. Sale price 65, cost 79.
Arithmetically flawless. I was one edit away from making it the emotional centre of the
video — the exact "you are losing money and don't know it" moment the whole pitch was built
around.

Then I noticed the losses clustered suspiciously: −21.9%, −21.9%, −21.4%, −22.0%. That's not
a distribution, that's a signature. The cost figure *before tax* was identical to the sale
price, and a 22% tax rate had been applied on top. The entire loss was the tax assumption.
On a product category that should have been tax-exempt in the first place.

Thirteen rows shared that shape. All of them looked like findings.

**Two.** So I filtered them out and quoted the remaining count: 93 products sold below cost.
Also wrong. Of those 93, forty-three had a tax rate the pipeline *assumed* rather than read
off the document. Strip those, strip the exempt categories, and the number that survives an
audit is **40**.

93 and 40 are both defensible-sounding. Only one of them survives someone checking.

**Three.** For the computer-vision video I ran the models over real footage and put the
confidence scores on screen: 98% on one material, 82% on another. Genuine model output.
From the wrong model. The config the device actually reads pointed at a newer stack — with
an extra classifier I hadn't even known existed. Under the deployed models those same clips
score **74% and 41%**.

I'd also scored a single best frame. Production averages across the whole pass. Single-frame
scores read dramatically higher than anything the device will ever report.

**Four.** Building the invoice-extraction scene, one line showed 15 units at 10,840.91 with
a line total of 722.73. Inverted. I checked the rest of the dataset: **21% of invoice lines
fail `quantity × price = total`**. Twelve percent match no interpretation at all.

That one nearly went on camera, zoomed in, as an illustration of clean automated extraction.

## The pattern

None of those four were caught by the pipeline. Each was caught by asking a question the
pipeline had no way to ask:

- *Why are these losses all near the tax rate?*
- *What filter would someone apply before believing this?*
- *Is this the model that's actually deployed?*
- *Does this row's arithmetic close?*

That's the actual work. It's also why I stopped, three-quarters of the way through, and
worked out whether it could be a product.

## An economics detour

The whole exercise — six renders across two projects, five voice tests, six generator
scripts, three documents — cost roughly **USD 100** in model usage. Fine. But the shape of
that spend was surprising enough to change how I'd build it again.

The model did no encoding. ffmpeg did that, locally, for free. So where did USD 100 go?

I pulled the session transcript and summed it:

| | |
|---|---:|
| Unique content the model read | 3.51M tokens |
| Re-reads of that same content | **131.8M** |
| Output generated | 628k |

A **38× re-read factor**. Context per turn grew from 50k to 469k. Split by quarter, output
stayed flat — 162k, 150k, 157k, 156k — while input per quarter went 13.9M to 52.5M. Same
work, nearly four times the cost, purely because the conversation got longer.

By the final task the ratio was 243×: 56k of genuinely new content, 13.7M of re-reading.
About 85% of that run's cost was carrying a conversation that had nothing to do with the
task.

The lesson is structural, not incidental. Anything built to do this repeatedly must run
grounding, narration and verification as **separate short-context calls**. That's the
difference between USD 2 and USD 8 per video — and between USD 8 and USD 100 if you let one
session sprawl.

## So: is it a product?

The obvious pitch writes itself. A local tool — no uploading your production database to
someone's SaaS — that turns your own systems into a demo where every claim carries a
verifiable source. Call it a claim ledger: nothing reaches the screen without a command
that re-derives it. Refuse to render otherwise.

I think that's a good idea and a bad business.

The code is about 1,600 lines of glue over Playwright, ffmpeg, TTS and ONNX runtime. Every
component is public and documented. It took two days. Anyone competent rebuilds it in two
days — I just demonstrated that.

And running locally, which is the whole trust proposition, means it can't be metered, and
the buyer pays their own model costs so there's nothing to mark up. Unmeterable, no
cost pass-through, trivially copyable. Any one of those is survivable. Together there's no
pricing surface at all.

But the deeper problem is what was actually scarce. On the final run alone I was wrong three
times — about whether a volume metric was usable, about what the chatbot even did, about
whether a mock was acceptable — and corrected all three by reasoning about the domain. Not
by applying a rule.

A claim ledger enforces **provenance**. It does not supply **insight**. It would have caught
me *stating* the wrong confidence figure unverified. Nothing in it would have prompted me to
go looking for the document that made the volume metric defensible.

Which produces an ugly market: the tool is most valuable to someone who already has the
domain judgment — and that person can build it in two days. Sell it to someone without that
judgment and it generates confident, wrong numbers faster than before. That's worse than no
tool.

So it isn't a product. It's a capability, and the durable part of it isn't the code — it's
the procedure. I wrote that up as an internal skill instead: ground before you narrate,
build the ledger before you write, prove the anonymisation rather than asserting it, say the
number a viewer can count on screen.

## What actually shipped

Two narrated videos, two to three minutes each. Rough. Synthetic voice that nobody will
mistake for a person. Crossfades instead of motion design.

But the detection boxes are real model output at the deployed threshold. The material labels
are what the deployed classifier returns, averaged the way production averages. The spoken
count of loss-making products equals the number of red rows a viewer can count on screen,
because I removed the ones that couldn't survive being checked. The mocked screens say
they're mocked, on screen, while they're visible.

At pre-revenue validation stage, that trade is right. What you're testing is whether the
message lands, not whether the production quality converts — and a rough video whose numbers
hold up teaches you more than a polished one whose numbers don't.

The polish can be bought later for USD 1,000–6,000. The credibility can't be bought back at
any price.
