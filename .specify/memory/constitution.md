# Stem Mixer — Constitution

## Inheritance

Slopsmith's core plugin contract governs everything in this repo (manifest schema,
`window.playSong` / `highway` hooks, `setup(app, context)` signature, `screen.html`
mounting, asset serving, and lifecycle). This constitution only enumerates
principles specific to Stem Mixer.

## Core Principles

### I. Bridge, don't replace
Stem Mixer is a thin UI/EQ layer over `window.stems` (the public API exposed by
`slopsmith-plugin-stems`). It MUST NOT instantiate its own `<audio>` elements per
stem or duplicate Web Audio routing that already lives in `stems`. When `stems`
is absent, Stem Mixer degrades to a no-op rather than partially booting (`screen.js`
explicitly waits for `window.stems` and bridges via `stems.getState()`).

### II. Local persistence, no server state
All user state (per-stem levels, EQ curve, autolevel toggle, named profiles)
lives in `localStorage` under `stem_mixer:state` and `stem_mixer:profiles`. The
backend `routes.py` is a health probe only — adding server-persisted settings is
out of scope and would break offline / multi-host installs.

### III. Non-destructive composition with stems
EQ filters, analyser, and output gain hang off the existing stem `GainNode`s
exposed by `stems.getState()` — never by reconstructing the graph. The mixer
button is inserted alongside (not in place of) the per-stem buttons that
`stems` already draws into `#player-controls`. When the panel is hidden, the
underlying control bar must remain visually and functionally unchanged.

### IV. Determinism for autolevel
Output autolevel is a closed-loop integrator with fixed targets
(`AUTLEVEL_TARGET_RMS = 0.16`), bounded gain (`[AUTLEVEL_MIN_GAIN = 0.45,
AUTLEVEL_MAX_GAIN = 1.7]`), and fixed smoothing (`AUTLEVEL_SMOOTHING = 0.14`).
These constants are load-bearing for predictable loudness across songs; changes
require a measurement-backed justification, not a vibe tweak.

### V. Defer to stems for mute semantics
A stem's on/off state is the `stems` plugin's responsibility. Stem Mixer MAY
multiply through its own gain stage to 0 but MUST NOT toggle `gain.gain.value`
on the shared `GainNode` it doesn't own — race conditions with the `stems` UI
button would result.

## Governance

These principles supersede ad-hoc convenience changes. Amendments must update
this file and `specs/001-stem-mixer/plan.md` together. Compliance is verified
by reading the diff against `screen.js` and the manifest in PR review.

**Version**: 1.0.0 | **Ratified**: 2026-05-09 | **Last Amended**: 2026-05-09
