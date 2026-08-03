---
title: "What Dogfooding an MCP Server Taught Me About Tool Output"
date: 2026-08-03T10:00:00-03:00
draft: false
tags: [mcp, agents, api-design, llm]
description: "A filter that discarded the answer, a retry that couldn't work, a partial result posing as a total, and cached prices with no freshness signal. All four found in one live session, all four the same class of mistake."
---

I built an MCP server that searches flights across flexible dates and multiple airports,
then used it for an actual trip I was planning. Three tool calls in, it had produced a
correct answer, and along the way had wasted most of a day's API quota and told the agent several things that weren't true.

None of the four problems was a crash. Every one of them was a *shape* problem: the output was well-formed, plausible, and misleading. That's the failure mode that matters when your consumer is a language model, because an agent can't smell that a number is wrong. It acts on whatever you hand it — and when the number is wrong in an expensive direction, it acts again.

Here's each bug and the rule I took from it.

## 1. The filter that threw away the answer

The server prices a leg by fetching a whole month's fare calendar, then filtering to the
dates inside your requested departure window. Search #1 asked for a window that didn't contain the good fare. The calendar it fetched *did* contain it — a fare four days outside the window, cheaper than anything inside it. The window filter silently dropped it. Two more searches, and two more rounds of quota, to rediscover a number the first call had already paid for and held in memory.

The fix wasn't to widen the filter — a window means something. It was to stop *discarding*.
The calendar fetch now buckets each day into in-window quotes and out-of-window
observations, and the result carries a `hints` block: the nearest fares just outside the
window, for legs that came back unpriced.

Two properties make this cheap rather than clever:

- **It costs zero extra API calls.** Only the calendar path can see out-of-window dates at all; the per-date path only probes dates you asked about. The hints are data that was already fetched and previously thrown in the bin.
- **They only appear where they're actionable** — attached to legs that failed to price, where "shift your window" is a real suggestion. Capped at three dates per side, as a module constant rather than a knob, because nobody should have to tune this.

The consequence is that a partial answer becomes actionable in one round trip. Instead of "no results, try again," the agent can say "nothing in your window, but the 20th is $603."

> **Rule:** never let a filter silently discard data the caller already paid for. Return it
> as a hint, labelled as out-of-scope.

## 2. The retry that could not possibly work

When a search came back empty, the natural next move — for me *and* for the agent — was to
retry with a finer date step. That retry was a guaranteed no-op, for two compounding
reasons: the step parameter did nothing at all on the calendar path, and on an empty
calendar month the per-date fallback re-asked *the same cached source*, one date at a time,
about twenty-one times.

So the retry burned twenty-one calls to re-confirm a "no" that one call had already
established.

The fix is a rule about when the fallback is meaningful. If every source behind the fetcher
is calendar-capable, an empty month **is** the answer, and the per-date fallback is skipped
entirely. If a per-date source sits behind the stack — one that can answer about a specific
date even when no calendar covers it — the fallback stays, because then it can genuinely
find something the calendar couldn't.

That's a one-call-versus-twenty-two difference on every empty route, and it looks enough
like a missing feature that I wrote *"do not fix this back"* into the decision record, along
with the caveat I accepted to get it: the month query is price-sorted with an upstream
result cap, so on a very dense route an in-window date could in principle hide beyond the
cap. In practice the repeated empty-calendar miss costs far more than that edge case.

> **Rule:** a retry knob that cannot change the answer must not exist. If your API exposes
> one, an agent will use it — patiently, repeatedly, at your expense.

## 3. The partial result that posed as a total

A multi-leg itinerary where one leg couldn't be priced returned this:

```json
{ "total": 524.0, "complete": false }
```

Technically defensible: `complete: false` says the trip isn't fully priced. In practice both
a human and a model read `total: 524.0` as *the price of the trip*, because that is what the
field is called. The flag is a footnote and the number is a headline, and nobody reads
footnotes.

The shape changed instead of the documentation:

| field          | behavior                                                                                   |
| -------------- | ------------------------------------------------------------------------------------------ |
| `total`        | **`null`** when the itinerary is incomplete — an unpriced leg means there is no trip total |
| `priced_total` | always present: the subtotal over the legs that *did* price                                |

Now the misleading read is impossible. If you want the partial sum you ask for the field
that says "partial" in its name. Nothing can accidentally sum to a wrong trip price, because
the wrong trip price no longer exists in the payload.

> **Rule:** a partial result must not have a field that looks like a complete one. Null the
> optimistic field and add an explicitly-scoped one beside it. Don't rely on a sibling
> boolean to be read.

## 4. Cached prices that looked fresh

The server caches fare calendars, and a long-running MCP process can serve a week-old price
without anything in the output hinting at it. Two things made this invisible:

- The fetcher's provenance banner was printed to **stderr** — which is exactly where it
  goes to die when the transport is stdio. The client sees the JSON. It never sees your
  banner.
- The timestamp field in the result shape existed but was never populated.

The fix was `sources` (which data sources the stack could consult) plus a per-leg
`fetched_at`. The interesting part is where `fetched_at` comes from.

The upstream price API exposes no freshness field at all — I probed for one and it isn't
there. But each offer's booking link embeds the observation date as a query parameter, so
the mapper parses the date out of the URL. That's a slightly grubby source for a timestamp,
and it is still much better than the obvious alternative: **stamping our own fetch time
would make week-old cached data look like it was observed a second ago.** A fetch time is
not an observation time. For a live per-date source they coincide, and there stamping is
correct; for a cache-backed source they can be a week apart, and that week is the whole
point.

`fetched_at` then flows all the way through the cache layer, so staleness introduced by
*our own* caching is visible in the output rather than hidden by it.

> **Rule:** freshness must describe the observation, not your fetch. And over stdio, anything
> not in the payload does not exist — provenance included.

## The pattern under all four

Every one of these is the same mistake wearing a different hat: **the output was optimized
for looking like a good answer instead of being an honest one.** A filtered-out fare doesn't
appear, so the result looks clean. A partial total is a number, so the result looks
complete. A cached price has no date on it, so it looks current. Each of those makes the
response *nicer* and the consumer *worse informed*.

With a human consumer you get away with a lot of this, because humans are suspicious. They
notice a suspiciously round number, they wonder whether "no results" really means no
results, they ask when the price was from. An agent extends you the courtesy of believing
your schema. If the field is called `total`, it's the total.

So the design rule I'd now apply to any tool an agent calls, before writing the first
handler:

1. **Every field an agent could act on must be honest about its own scope.** If it can't be,
   null it and add one that can.
2. **Never discard data the caller paid for.** Out-of-scope isn't worthless; label it and
   return it.
3. **Don't ship a parameter that can't change the outcome.** Agents retry; make retrying
   mean something.
4. **Put provenance and freshness in the payload**, sourced from the observation, not from
   your own clock.
5. **Dogfood it against a real task**, not a test suite. All four of these came out of one
   session of actually trying to book a trip. None of them would have failed an assertion —
   every response was valid, parseable, and wrong.

That last point is the cheapest one to act on. Test suites check that your output matches
what you expected. Using your own tool for something you actually need checks whether what
you expected was the right thing to want.

## Sources

- [rihla](https://github.com/leojg/rihla) — the MCP server these came out of
- [Model Context Protocol](https://modelcontextprotocol.io/) — transport and tool-definition
  docs, including why stderr is unavailable to the client over stdio
