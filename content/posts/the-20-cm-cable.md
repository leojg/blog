---
title: "The 20 cm Cable"
date: 2026-08-16T09:00:00-03:00
draft: false
tags: [raspberry-pi, hardware, edge, usb, debugging]
description: "A Pi 5 and a depth camera that crash-looped through two wrong diagnoses and one brand-new '6 A' cable. The root cause wasn't a component — it was a budget equation with 0.3 V in it."
---

A Raspberry Pi 5 with an OAK-D Pro depth camera bolted to it, recording material on a
conveyor belt. First boot on the rig: hard crash within minutes. `vcgencmd get_throttled`
returned `0x50000`, the journal had fourteen `Undervoltage detected!` lines, the USB bus was
dead — no camera, no tag reader — and the capture service was in a reset loop.

Two days of bench work later the fix was a 20 cm cable. Getting there went through two hypotheses that were both partly right and individually wrong, and ended somewhere more useful than a culprit: an equation with about 0.3 V of headroom in it.

## Chapter zero: the problem we thought we'd already solved

This wasn't the first time this hardware pair had browned out. An earlier rig had a
long-standing "USB-2 link instability" symptom — the camera cycling connect/disconnect every
second or two, `X_LINK_UNBOOTED` in the logs, `over-current change` in `dmesg`. It was
diagnosed as, and turned out to be, two compounding current limits:

1. **The supply was undersized.** A Pi 5 alone wants 3 A or better; a depth camera running three sensors, stereo matching, two on-device networks and hardware encoders wants real headroom on top of that.
2. **The Pi 5 caps each USB port at about 600 mA by default.** `usb_max_current_enable=1` in `/boot/firmware/config.txt` lifts it — but only ever *after* an adequate supply is in place, since raising the port ceiling on an undersized brick just lets the camera pull the whole rail down faster.

One diagnostic from that round is worth keeping, because it kills an entire branch of the search tree: **forcing the link to USB 2 does not fix it.** If the device still drops under load at high speed, the fault is current, not signal integrity. (Idle stability at USB 2
proves only that idle draw is under 600 mA.)

So when the belt rig crashed, we thought we knew this problem. We were wrong about which part of it.

## Hypothesis 1: the brick is a lottery

The belt rig had an adequate-on-paper supply — a generic 5 V / 3.5 A brick. Measuring the
input rail with `vcgencmd pmic_read_adc EXT5V_V` on a bare Pi, two units *of the same model*
read:

- unit A: **4.97 V**
- unit B: **4.68–4.71 V**

The Pi 5's undervoltage threshold is **4.8 V**. Unit B is below the threshold with nothing
plugged in but the board.

That's a real finding and it's worth internalizing: **a printed rating identifies a model;
the set-point is per unit.** Two bricks off the same production line, same box, same
sticker, half a volt apart in practice. Nothing on the packaging distinguishes them.

But swapping to unit A didn't fix the rig, so the lottery wasn't the whole story.

## The one reading that exonerates the board

At this point a nastier hypothesis was live: this particular Pi had "acted up" before, and a damaged input path would look exactly like a weak supply.

There's a clean discriminator, and it takes thirty seconds: **power the bare Pi from a
known-tight source and read the rail.** A laptop USB-C port measured **5.065 V** at the
Pi's own ADC. If the board reads healthy on a good source, the input path isn't dropping anything, and every low reading you've been collecting belongs to the supply or the cable.

Board exonerated, in one number. Worth knowing before you order a replacement you don't need.

## Hypothesis 2: it's the cable

The A/B that changed the investigation used one brick and three cables:

| cable               | result                                                 |
| ------------------- | ------------------------------------------------------ |
| short, camera-grade | works                                                  |
| old long cable      | won't run at all                                       |
| second old cable    | idles 4.96 V, but `0x50000` from boot transients alone |

That third row is the interesting one. It idles *above* threshold and has still already
failed — the sticky throttle bits recorded a dip that happened during boot, before anything
was measured.

So: buy a good cable. A brand-new 1.5 m unit marked **"6 A QC3.0"** went on, camera
attached, and the rig read **4.718 V** — then the Pi died mid-session, the journal ending
mid-stream at `Undervoltage detected!`.

The same new cable, measured from the laptop port: 4.80–4.85 V. Which decomposes the whole thing:

- The cable eats about **0.24 V at near-idle** — roughly a 0.15 Ω loop.
- The brick sits about **0.1 V below** the laptop source.

Neither number alone would have killed it. Together they land under 4.8 V, and under load they land far under.

## The actual root cause is an equation

The final model isn't a culprit, it's a budget:

```
supply set-point − (loop resistance × current)  must stay above 4.8 V
                                                 (target ≥ 5.0 V)
```

...and it has to hold through **boot transients** and **full pipeline load**, not at idle.

Now put real numbers in it. Official supplies are set to **5.1 V**. The undervoltage
threshold is **4.8 V**. That 0.3 V is the *entire* budget — for supply tolerance and cable
resistance combined. A 0.15 Ω loop at 3 A spends 0.45 V of a 0.3 V budget on its own.

A **20 cm** cable on the same brick held **5.05–5.10 V through a full ten-minute recording**,
throttle flags clean. The camera also came back up at USB 3 SuperSpeed — it had been quietly falling back to USB 2 the whole time.

## Why a "6 A" cable wasn't

The cable that killed the Pi was newer, thicker-feeling, and rated for more current than the one that fixed it. Its rating wasn't a lie, it was answering a different question.

"QC3.0" is a fast-charge negotiation profile that does its work at elevated voltages —
around 9 to 12 V. At 12 V, moving a given wattage takes a quarter of the current it takes at 5 V, so the same conductor resistance produces a quarter of the voltage drop, against a supply rail more than twice as tall. A quarter-volt of IR drop is a rounding error up there. At 5 V and 3 A, that same quarter-volt is most of your headroom.

Which means: **at 5 V, length and gauge matter enormously; at phone-charging voltages they barely do — and phone cables are specified for the second case.** The ratings that actually correlate with conductor gauge are e-marked C-to-C 5 A cables and OEM fast-charge classes. Everything else printed on a no-name cable is marketing about a use case that isn't yours.

## The acceptance gate

The lasting output of the incident isn't the cable, it's the test — and specifically its
unit. **Judge power per physical brick-and-cable *pair*, never per model.**

Boot the rig with the exact pair that will ship, run a real recording with the camera
attached, then:

```sh
vcgencmd get_throttled          # want 0x0 AFTER load
vcgencmd pmic_read_adc EXT5V_V  # want ≥5.0 V during the recording
```

Two things make this gate work where a voltmeter reading doesn't:

- **`get_throttled` is sticky.** It records that a dip *happened*, so it catches boot
  transients and momentary sags that a one-shot voltage read walks straight past. A pair
  that idles at 4.96 V and shows `0x50000` after boot has already failed the test.
- **It's under real load.** The pipeline draws more than boot does, and boot draws more than
  idle. A rig that initialises fine and dies when recording starts is the signature of this
  entire class of problem: *stable at idle, drops under load.*

The flags reset on power cycle, so every re-plug is a fresh test of that exact pair.

## The early-warning symptom worth alerting on

The single most useful field signal to come out of this: **the camera enumerating at USB 2
high-speed instead of USB 3.** The link trains down before the Pi logs a single undervoltage
line. By the time you have throttle flags you have a crash; by the time you have a USB 2
fallback you have a warning.

On a rig you can't put a multimeter on, that's the cheapest health check available — one
`lsusb` line, checked on every session start.

## What I'd tell past me

1. **Judge power per physical pair, under load, by sticky flags.** Not per model, not at
   idle, not with a one-shot reading.
2. **Printed cable ratings are answering a question you didn't ask.** At 5 V, assume nothing
   until the pair passes the gate.
3. **Exonerate the board early.** One reading from a known-tight source saves you from
   debugging a hardware failure that isn't there.
4. **When two suspects each look partly guilty, stop looking for a culprit.** Both hypotheses
   here were real effects — a low brick set-point and a resistive cable — and neither on its
   own explained the crash. The answer wasn't a component. It was the sum, against a budget
   nobody had written down.

That last one is the part that travels. A system with a 0.3 V margin between "works" and
"corrupts your recording mid-session" doesn't fail because a part is bad. It fails because
nothing in the build process was tracking the margin, and every individually-reasonable
choice spent a little of it.

## Sources

- [Raspberry Pi documentation](https://www.raspberrypi.com/documentation/) — the Pi 5 power
  supply requirements and the `usb_max_current_enable` config option
- [Luxonis documentation](https://docs.luxonis.com/) — OAK-D power draw and USB requirements
  under full pipeline load
- `vcgencmd get_throttled` bit meanings: `0x50000` = under-voltage *has occurred* +
  throttling *has occurred* (the sticky, historical bits — not the live ones)
