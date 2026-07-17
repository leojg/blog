---
title: "There Is No Sandbox for WhatsApp Coexistence"
date: 2026-07-17T15:00:00-03:00
draft: false
tags: [whatsapp, meta, n8n, testing, automation]
description: "Meta's test numbers can't exercise it, no BSP offers one, and the feature only exists on a real phone number. How we built a test environment out of a $1 prepaid SIM."
---

WhatsApp Coexistence is the feature that lets a business keep using the WhatsApp
Business App on a phone while the *same number* is also connected to the Cloud API
through a Business Solution Provider (BSP). Messages mirror in both directions, and the
API side receives a webhook event — `smb_message_echoes` — every time a human replies
from the app.

That echo event is gold if you're building a bot that shares a number with real staff:
the bot can detect that a human has stepped into the conversation and shut up. I was
building exactly that — an order-taking bot for a client whose employees answer the same
WhatsApp number all day — and the handoff logic hinged entirely on those echoes.

Which raised the question: how do you test any of this *before* onboarding the client's
production number?

The answer, after a deep research pass across Meta's docs and every major BSP, turned
out to be: **you can't, anywhere, with any sandbox that exists.** Here's what's ruled
out, the workaround that actually works, and two findings along the way that changed our
rollout plan.

## Everything that looks like a sandbox, and why it isn't one

**Meta's free WABA test numbers.** Coexistence onboarding only works in one direction:
from the app to the API. A number that's already on the Cloud API — which is what a test
number is from birth — can't be onboarded into coexistence. 360dialog's FAQ says it
plainly: *"If the number is connected to the WhatsApp API, this Coexistence Onboarding
won't work."* Test numbers also can't receive unsolicited inbound messages and only
message pre-registered recipients, so they're a poor stand-in for a customer-facing
number even without the coexistence problem.

**BSP sandboxes.** 360dialog's sandbox is a Cloud-API-only smoke-test tool (you message
`START` to their number; capped at 200 messages, and it can only message you back).
Twilio's WhatsApp sandbox is a shared Twilio-owned number — no WABA of your own, no
Business App, no echo events. Neither exercises coexistence at all. I also went through
YCloud, Gupshup, and Infobip docs: no BSP documents any coexistence test mode.

**VoIP and virtual numbers.** WhatsApp's policy doesn't support VoIP numbers for
registration. Some reportedly pass the OTP anyway — but ongoing monitoring can ban the
number later, and as you'll see below, a ban two weeks in doesn't just cost you the
number. It costs you the two weeks.

## The workaround: a $1 SIM and two weeks of patience

The only high-fidelity path that doesn't touch the production number is embarrassingly
low-tech: buy a cheap prepaid SIM (about a dollar at any kiosk here in Uruguay), put it
in a spare phone or a dual-SIM slot, and register it in the regular WhatsApp Business
App.

The catch is the warm-up. Newly created Business App accounts are **not immediately
eligible** for coexistence — Meta gates eligibility on account tenure and messaging
quality. 360dialog states the gating on three separate pages; practitioner reports put
the threshold at roughly **seven days of active use**, and a number that was previously
on the Cloud API needs its WABA deleted and one to two *months* of app usage before it
becomes eligible again. So the SIM needs to live like a real account for a week or two:
daily messages to real contacts, from the app.

That tenure clock is the long pole of the whole project, which inverts the usual order
of operations: **buy the SIM before you finish the research.** Start the clock on day
zero and do the rest of the work while it runs.

Once the account is warmed up, you onboard it through your BSP's embedded signup (a QR
scan from inside the app), point the webhooks at your dev tunnel, and you have the real
thing: genuine echo events, genuine history sync, and — as a bonus I didn't expect to
value as much as I did — a full dress rehearsal of the exact onboarding UX your client
will go through on cutover day.

## You don't need a real business

My first objection to the SIM plan was: onboarding requires a *business* — surely a
throwaway test number fails there. It turns out the "business" in this flow is a **Meta
Business Portfolio, not a legal entity**, and nothing in it is tied to the SIM:

- SIM ownership is never checked. The number is verified by OTP only; whoever
  registered it at the carrier is invisible to Meta.
- The Business App requires no business at all — the business name is free text.
- Embedded signup needs a Facebook login and a portfolio, which the flow lets you
  create on the spot. You type a name and a website — or tick *"my business does not
  have a website."* All self-declared, no documents.
- Meta business *verification* is not required to complete signup or start messaging.
  It's a later, optional step to lift limits. Unverified limits — two phone numbers per
  portfolio, a couple hundred business-initiated conversations per day — are irrelevant
  when you're testing a customer-initiated bot. (Unverified portfolios can be
  restricted at any time, which is an acceptable risk for a disposable one.)

The one real decision is *whose* portfolio to use — and the answer is emphatically not
your client's. A number's portfolio binding **cannot be changed after registration**,
and test assets would pollute their Business Manager forever. Create a fresh throwaway
portfolio under your own account. Eligibility gating lives on the app account, not the
portfolio, so this choice doesn't affect the warm-up clock.

## The fear that turned out to be mostly unfounded

The reason we were researching alternatives at all was a specific fear: *if we hook a
webhook to the production number and the bot breaks, customer service breaks with it.*

The docs don't support that fear. Meta, YCloud, and 360dialog all agree: under
coexistence, the Business App keeps sending and receiving independently of the API
connection. Messages mirror both ways; a dead webhook just means the API side misses
events until it's back. The bot can crash all weekend and the phones keep working.

What *is* real — and the actual reason to rehearse on a test number first — is the list
of onboarding side effects:

- **All linked companion devices are unlinked** at onboarding. If staff answer via
  WhatsApp Web or Desktop, that's a real disruption (they re-link afterward — and
  replies from some companion platforms don't fire echo events at all, a hole in any
  handoff logic that assumes they do).
- Several app features are disabled: disappearing messages, view-once, live location,
  broadcast lists (read-only), message edit and revoke.
- **Group chats never sync to the API.** Group-based customer service stays app-only.
- The app must be opened roughly every **14 days**, or the connection can drop.
- The contact list and up to six months of 1:1 chat history sync to the BSP — history
  sharing is optional and chosen at onboarding, but it's a data-exposure decision
  someone should make consciously, before the onboarding screen is in front of them.
- API throughput is capped around 5 messages/second for coexistence numbers.
- Rollback is **client-side only**: disconnect from inside the app's settings. The
  Deregister API can't offboard a coexistence number, and whether the disabled app
  features come back after disconnecting is not documented anywhere I could find.

That last point deserves emphasis. With an undocumented rollback, treat coexistence
onboarding as a **one-way door** for planning purposes — one more reason the first
number through it should cost a dollar.

## The free track you run in parallel

While the SIM warms up, you can still make progress for free: simulate the webhooks.
The real payload shapes are documented (360dialog's coexistence-webhooks page is the
best reference): a message sent from the app produces **two** webhook deliveries — an
inbound `messages` event on the recipient's side, and an `smb_message_echoes` event on
the sender's number, both to the phone-number-level webhook URL.

A handful of curl fixtures replaying those shapes against a local n8n instance
exercised our entire echo branch — the "human replied, set `needs_human`, bot exits
early" state machine — months of logic, no Meta involved. It validates *your* branching,
not Meta's behavior, so it complements the real test rather than replacing it. But it
means that when the warmed-up number finally onboards, the only things left to verify
are the ones you genuinely couldn't fake.

## The playbook

1. **Day 0:** buy the prepaid SIM, register it in the WhatsApp Business App, start
   daily use. Build the webhook curl fixtures in parallel.
2. **Day 7–14:** attempt coexistence onboarding through your BSP. If Meta rejects for
   tenure, keep warming up and retry weekly.
3. **Only after an end-to-end pass on the test number:** schedule the production
   onboarding in a low-traffic window — staff warned about device re-linking, the
   history-sync decision made in advance, and the bot gated behind a sender allowlist
   until you deliberately open it up.

The general lesson travels beyond WhatsApp: when a platform feature has no sandbox, you
can usually build one out of the same raw material production uses — and the fidelity is
perfect precisely because nothing about it is fake. And when eligibility is gated on
account tenure, the first step of the project isn't research. It's starting the clock.

## Sources

- [Meta — Onboarding WhatsApp Business App users (Coexistence)](https://developers.facebook.com/docs/whatsapp/embedded-signup/custom-flows/onboarding-business-app-users/)
- [360dialog — WhatsApp Coexistence: onboarding, FAQ, and webhooks](https://docs.360dialog.com/partner/onboarding/whatsapp-coexistence)
- [YCloud — WhatsApp Business App Coexistence](https://www.ycloud.com/blog/whatsapp-business-app-coexistence-meta-update)
- [Twilio — WhatsApp Sandbox](https://www.twilio.com/docs/whatsapp/sandbox) (Cloud-API-only, for contrast)
