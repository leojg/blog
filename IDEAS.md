# Post pipeline

Not built by Hugo — lives outside `content/`.

Slots are **Wed + Sat 09:00**. `scripts/blog queue` shows what's free.
Each entry below names the evidence it rests on, because the ones that land are the ones
with a real measurement behind them rather than an opinion.

---

## Next 5

### 1. A Classifier That Can Say "Not Mine"
`machine-learning, computer-vision, classification, design`

Two-stage material classification where each refiner carries an explicit **reject class**
(`not_basalt`, `not_tosca`) — semantically distinct from low confidence. Low confidence
means *"probably this, unsure"*; reject means *"wrong family entirely"*. Both fall back to
the coarse label plus review, so a product name is never guessed.

**Evidence:** `sampler/sampler.py::_classify_track`, `sampler/config.py` (gates at 0.70).
Real cases from the demo work — one pass returned `not_basalt` at 0.66; another returned
the right answer (`tosca`) at 0.60, under the gate. Both correctly refused to name a
product.

**Why it's worth writing:** almost all classification writing assumes a closed set. The
reject-class-plus-gate pattern is what you actually need in production, and it's
under-documented.

---

### 2. The Cache That Cancels Your Improvement
`caching, llm, data-pipelines, debugging`

An LLM matching pipeline cached the **fully adjudicated, post-gate result**, keyed on input
text. So improving the normalisation upstream changed nothing on any cached item — a
recompute faithfully reproduced the old decisions. The improvement was real going forward
and invisible retroactively.

**Evidence:** `docs/postmortem/invoice-margins-prod-hardening.md`. Second occurrence of the
same class of bug in that system — an earlier cache also outlived a change to its inputs.

**Angle:** a cache must be keyed on everything that affects the result, and a cache holding
*post-decision output* silently cancels any improvement to the logic that produced it. Easy
to state, apparently very easy to re-commit.

---

### 3. Error-Tolerant Nodes Turn Failures Into Silent Corruption
`n8n, automation, reliability, postmortem`

Two separate defects were invisible *specifically because* a node was configured to
continue past errors — and one of them still wrote a "success" record afterwards. Separately,
a scheduled trigger failed for days on an API rate limit with nobody notified: the data
already loaded was fine, so nothing looked broken.

**Evidence:** the same postmortem, plus the price-list sync trigger found failing during the
demo-video work (`Quota exceeded … queries per minute`, 6/6 recent runs errored).

**Thesis:** tolerating errors is only safe where a downstream check can tell the difference
between "nothing to do" and "it didn't run". Otherwise you've converted a loud failure into
a quiet one, which is strictly worse.

---

### 4. Assert on What's Deployed, Not on What You Sent
`devops, deployment, verification`

A one-character expression prefix was lost on the hosted copy of a database node, turning
its insert into a literal string. It survived a deploy because the deploy script asserted
the prefix on **the payload it was about to send**, never on the hosted copy afterwards.
Combined with error-tolerant settings, a future run would have truncated a catalog, failed
the insert, recorded "version loaded", and reported success.

**Evidence:** same postmortem. Pairs naturally with a second instance from the video work —
notes and docs described a v8 model stack while the config the device reads pointed at v10,
which put wrong confidence figures on screen.

**Thesis:** verification that reads your own intent instead of the deployed artifact isn't
verification. Applies equally to config, models and infrastructure.

---

### 5. Measure What You Can Defend
`measurement, uncertainty, machine-learning, honesty`

Two volume estimates from the same project: per-truck (±25%, and the published headline
pair didn't reproduce once a sign error was corrected — 4.65 and 17.35 m³ for a reported
≈15) versus belt throughput (15.19 m³/h, SEM ±6.1%, 11 sessions, range 10.0–19.0).

One is a number; the other is a measurement. The difference isn't precision, it's whether
the error bar is known and stated.

**Evidence:** `core/docs/TRUCK_VOLUME_ESTIMATION.md`, `core/docs/BELT_VOLUME.md`.

**Angle:** the decision to cut the truck figure from a sales video entirely, and ship the
belt figure *with* its ± visible. Stating uncertainty reads as more credible than hiding it
— especially to technical buyers who will go looking.

---

## Also candidate, lower priority

- **The 38× re-read** — agent session token economics in depth. Partly covered in *The Demo
  Video Was the Easy Part*; only worth writing standalone if there's more than the same
  measurement restated.
- **Proving an anonymisation** — cross-checking 2,091 published rows against 444 real
  identities before showing a client's data. Thin on its own; may fold into a wider post
  about demo data.
- **Ambiguity must return nothing** — the address-matching query that yields no row when
  two businesses both match, rather than picking one. Good principle, needs a second
  example to carry a post.
