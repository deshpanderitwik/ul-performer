# ul-performer

A web-based timing trainer for the [Teenage Engineering EP-40](https://teenage.engineering/store/ep-40) (and its sibling EP-133 KO II) finger-drumming pads. The goal: build the kind of tight, locked-in timing that distinguishes a drummer from a person who can press buttons in time-ish.

## What it does

You configure a *lesson* — a pattern, BPM, bars per rep, reps per set, and rest length — then play through a set with a metronome and count-in. The pads light up "warm" on each scheduled hit (Guitar Hero–style anticipation). When you strike a pad, you get color-coded feedback in real time:

- **Green** — within ~13% of a subdivision (perfect)
- **Yellow** — within ~40% of a subdivision (good)
- **Red** — within the hit window but loose (off)
- **No color** — outside the window (stray)

Between reps a halftime panel shows your timing stats; at the end of the set, you get an aggregate.

## Why

Becoming a finger-drumming god requires more than vibes — it requires ruthless slow practice with feedback. Teachers drill students on rudiments at painful tempos until accuracy is robotic. This tool is an attempt to make that loop self-serve, on the device a player already has.

The roadmap mirrors percussion pedagogy stages 1–3:
- Stage 1 — pulse on a single pad
- Stage 2 — two-pad alternation (kick / snare quarters & halves)
- Stage 3 — subdivisions (eighths, sixteenths)

See [`PRD.md`](./PRD.md) for the full v2 product spec.

## Stack

Plain HTML / CSS / vanilla JS. No build step, no dependencies, no framework. Single `index.html`. The implementation deliberately avoids tooling so the project stays approachable and trivially deployable.

Browser APIs in use:
- **Web MIDI** for reading note-on/off events from the EP-40
- **Web Audio** for the metronome (sample-accurate scheduling via the lookahead pattern)
- **localStorage** for the last-used lesson and pad mapping

## Running

The app needs `localhost` (Web MIDI requires a secure origin or `localhost`). Any static server works:

```bash
python3 -m http.server 5174
# then open http://localhost:5174
```

Open in Chrome (or another browser with Web MIDI support) and grant MIDI permission when prompted. Plug in the EP-40 via USB-C.

## Architecture sketch

- **Pattern catalog** — patterns are described as `{ subdivision, events: [{ tick, pad }] }` within a single bar. The scheduler iterates ticks and emits a metronome click on each beat plus a warm flash on any tick that has a pattern event.
- **State machine** — `setup → playing → resting → playing → … → set-summary`, with `paused` overlaying either `playing` or `resting`. A single overlay container renders three modes (setup form / halftime / set summary).
- **Hit detection** — every pad strike during a rep is matched to its nearest expected event of the same note. Inside the hit window, a color-coded scored flash; outside, a stray flash. Hits and strays are logged per rep.
- **Hit window scaling** — the window is `min(300ms, subdivisionMs / 2)`, so quarter-note practice is forgiving and sixteenth-note practice is tight. Green / yellow thresholds scale with the window.

## Status

v2 is in active development:

- ✅ Pattern catalog + lesson model
- ✅ Multi-rep set lifecycle (auto-advance, skip rest, pause, soft stop)
- ✅ Setup screen with form
- ⏳ Halftime panel polish (real layout, cumulative stats)
- ⏳ Set summary page (per-rep sparkline, trend tag, repeat / new lesson)
- ⏳ Set history persistence (foundation for stage-7 progression tracking)

## License

No license yet — treat as all rights reserved until one is added.
