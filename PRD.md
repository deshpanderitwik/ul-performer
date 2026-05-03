# ul-performer v2 — PRD

## Overview
v2 evolves the app from a single-round timing trainer into a sets-based practice tool oriented around stages 1-3 of percussion pedagogy (two-pad patterns only). The user configures a *lesson* (pattern + BPM + bars/rep + reps/set + rest), runs a *set* of reps with rest between them, and reviews per-rep + cumulative stats at the end.

## Goals
- Replace the single 30-second round model with sets of configurable reps
- Introduce a Setup screen as the default landing
- Provide stage 1-3 coverage via four two-pad patterns
- Add a halftime "rest" experience between reps showing prev-rep + cumulative stats
- Add a set summary screen with per-rep breakdown, trend, and re-engagement buttons
- Persist last lesson config and set history (silent for now)

## Non-goals (v2)
- Single-pad patterns (stage 1) — covered later
- Three+ pad patterns / hi-hat layer
- Velocity / dynamics scoring
- Audio backing tracks
- Set history surfaced to the user (data captured but not displayed)
- Pattern preview visualization on Setup (text labels only for v2)
- Mid-set resume on reload

## Vocabulary
- **Lesson** — a configured exercise: pattern + BPM + bars/rep + reps/set + rest seconds
- **Rep** — one round of playing the pattern for the configured number of bars
- **Set** — N reps of a lesson with rests between
- **Session** — one sitting (potentially multiple sets, possibly different lessons)

## User flow
```
Setup → Begin set
  → Play (rep 1) → Rest (halftime)
  → Play (rep 2) → Rest
  → ...
  → Play (rep N)
  → Set summary
  → New lesson | Repeat lesson
```

## Screens / overlay
The pad grid + transport bar is always rendered as the **base layer**. A single
**overlay** sits on top with a non-dismissible scrim. The overlay has three modes
(only one visible at a time):

1. **setup** — default landing; lesson config form; "Begin set" CTA
2. **rest** — between reps; prev-rep stats + countdown + skip-rest
3. **set-summary** — after last rep or after Stop; aggregate + per-rep + buttons

The overlay is hidden during `playing` and `paused-from-playing` so the user
can interact with the pads.

## State machine
- `setup` — configuring lesson (default landing)
- `armed` — lesson committed; transient before audio starts
- `playing` — a rep is running (count-in + bars)
- `resting` — between reps
- `set-summary` — after final rep or after Stop
- `paused` — overlays `playing` or `resting`; resumes to underlying state

## Pattern catalog (v2)
Four two-pad patterns. Pads referenced by MIDI note (C2 = 36, C#2 = 37).

### Schema
```js
{
  id: string,           // unique slug
  name: string,         // display label
  meter: [num, denom],  // [4, 4]
  subdivision: number,  // 4 = quarter, 8 = eighth, 16 = sixteenth
  events: [             // events within one bar
    { tick: number, pad: midiNote },
    ...
  ]
}
```

`tick` is in subdivision units. Tick spacing in seconds = `(60 / BPM) * (4 / subdivision)`.

### Patterns
```js
{ id: "ks-quarters",   name: "Kick/snare quarters",   meter: [4,4], subdivision: 4,
  events: [{tick:0,pad:36},{tick:1,pad:37},{tick:2,pad:36},{tick:3,pad:37}] }

{ id: "ks-halves",     name: "Kick/snare halves",     meter: [4,4], subdivision: 4,
  events: [{tick:0,pad:36},{tick:2,pad:37}] }

{ id: "ks-eighths",    name: "Kick/snare eighths",    meter: [4,4], subdivision: 8,
  events: [{tick:0,pad:36},{tick:1,pad:37},{tick:2,pad:36},{tick:3,pad:37},
           {tick:4,pad:36},{tick:5,pad:37},{tick:6,pad:36},{tick:7,pad:37}] }

{ id: "ks-sixteenths", name: "Kick/snare sixteenths", meter: [4,4], subdivision: 16,
  events: [{tick: 0,pad:36},{tick: 1,pad:37},{tick: 2,pad:36},{tick: 3,pad:37},
           {tick: 4,pad:36},{tick: 5,pad:37},{tick: 6,pad:36},{tick: 7,pad:37},
           {tick: 8,pad:36},{tick: 9,pad:37},{tick:10,pad:36},{tick:11,pad:37},
           {tick:12,pad:36},{tick:13,pad:37},{tick:14,pad:36},{tick:15,pad:37}] }
```

## Lesson schema
```js
{
  patternId: string,    // references pattern catalog
  bpm: number,          // 40-200
  barsPerRep: number,   // integer in [1, 65]
  repsPerSet: number,   // 1-20
  restSeconds: number,  // 0-60
}
```

## Setup screen spec
Default landing. Pre-filled with last-used lesson (or defaults if none).

| field | default | constraints | input |
|---|---|---|---|
| Pattern | ks-quarters | one of catalog | radio buttons (4) |
| BPM | 90 | 40–200 | numeric input |
| Bars / rep | 16 | 1–65 (integer) | numeric input |
| Reps / set | 5 | 1–20 | numeric input |
| Rest (s) | 10 | 0–60 | numeric input |

Single CTA: **Begin set**.

Pad mapping reset still accessible (small button in a corner of the Setup screen).

## Play screen spec
- Pad grid (existing 3×4 layout, mapping unchanged)
- Transport bar: pattern name, BPM, **rep K/N**, bar X/Y, beat 1-4, Pause + Stop buttons
- Hit feedback (warm flash for scheduled beats, color-coded green/yellow/red for hits, ms badge) carries over from v1
- Scheduler emits warm-flash + click on every event in `pattern.events` for each bar of the rep

## Rest / halftime page spec
Shown between reps (rep K complete, K < N). Auto-advances to next rep when countdown reaches 0.

- Header: "Rep K complete · Rep K+1 in Ts"
- Big countdown to next rep
- **Skip rest** button → immediately advances to next rep
- Pause + Stop buttons remain visible
- **Previous rep summary** — same panel layout as v1 summary (hits, misses, strays, mean, std dev, distribution)
- **Cumulative set summary** — same shape, computed across all reps so far

## Set summary spec
Shown after final rep finishes naturally OR after Stop.

- Header: "Set complete" (natural end) or "Set ended" (after Stop)
- **Cumulative stats** (same shape as halftime cumulative)
- **Per-rep breakdown** — small bar chart, one bar per rep, height = beats-hit %, color tint = mean accuracy
- **Best rep / worst rep** — explicit callouts (rep #, %)
- **Trend tag** — derived from rep-over-rep deltas:
  - "you got tighter" — std dev decreased rep over rep (linear regression slope < -2ms)
  - "you held steady" — std dev within ±5%
  - "you slipped" — std dev increased OR mean offset drifted by >10ms
- **Total practice time / hits** — vanity stats
- **Buttons**: New lesson (→ Setup) | Repeat lesson (→ Play same lesson, fresh set)

## Hit window scaling
v1: ±300 ms with green ≤40, yellow ≤120 (constant, regardless of subdivision).

v2: scales with subdivision so window is always ~half a subdivision unit, with hard upper bound of 300ms.

```js
const subdivisionMs = (60 / bpm) * (4 / pattern.subdivision) * 1000
const windowMs = Math.min(300, subdivisionMs / 2)
const greenMs  = windowMs * 0.13
const yellowMs = windowMs * 0.40
```

Examples:
- 90 BPM quarter: subdivisionMs ≈ 666, windowMs = 300, green ≈ 40, yellow ≈ 120 (= v1)
- 90 BPM eighth: subdivisionMs ≈ 333, windowMs ≈ 167, green ≈ 22, yellow ≈ 67
- 90 BPM sixteenth: subdivisionMs ≈ 167, windowMs ≈ 83, green ≈ 11, yellow ≈ 33

## Persistence (localStorage)
| key | shape | purpose |
|---|---|---|
| `ul.padMapping` | `{ [padNum]: midiNote }` | pad ↔ MIDI note (existing) |
| `ul.lastLesson` | Lesson object | repopulate Setup form |
| `ul.setHistory` | `[{ ts, lesson, reps:[...], aggregates }]`, capped at 200 | future progression tracking (silent in v2) |

No mid-set resume.

## Controls
- **Pause** — works in `playing` and `resting`. Freezes timers/audio. Resume restores exact state.
- **Stop** — soft end. Lands on Set summary with whatever data was collected.
- **Skip rest** — only visible in `resting`. Jumps to next rep immediately.

## Count-in
Each rep starts with the v1 4-beat count-in. Bar tracking starts only after the count-in finishes. Bars per rep refers to pattern bars, not including count-in.

## Constants
```js
const COUNT_IN_BEATS = 4
const BPM_MIN = 40, BPM_MAX = 200
const BARS_MIN = 1, BARS_MAX = 65
const REPS_MIN = 1, REPS_MAX = 20
const REST_MIN = 0, REST_MAX = 60
const SET_HISTORY_CAP = 200
const HIT_WINDOW_MAX_MS = 300
const HIT_GREEN_FRACTION = 0.13
const HIT_YELLOW_FRACTION = 0.40
```

## Implementation notes
- Single HTML file (`index.html`), vanilla JS modules. No build step. Match existing code style.
- **Each prompt must leave the app in a working state** — no half-broken intermediate commits.
- CSS dark theme; monospace; existing color tokens (`#0b0b0d` background, `#f2f2f2` text, etc.).
- Each new screen is a `<section>` with its own ID; only one visible at a time, controlled by a `data-screen` attribute on `<body>`.
- All hit/timing math goes through the audio context clock, never `Date.now()`.
- Persist to localStorage on every meaningful state change (lesson change, set complete).
- Don't introduce frameworks, dependencies, or build tooling.
