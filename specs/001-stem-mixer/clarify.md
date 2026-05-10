# Clarifications — Stem Mixer

Open and resolved questions captured during the retrospective spec pass.
Resolved entries reference the in-tree decision; open entries are mirrored in
`spec.md` as `[NEEDS CLARIFICATION]`.

## Q1 — What is the relationship to `slopsmith-plugin-stems`?
**Resolved.** `stem_mixer` is a strict consumer of `window.stems`. It never
constructs its own per-stem `<audio>` elements. If `window.stems.getState()`
returns `[]` or is undefined, the player Mixer button MUST NOT appear (see
`screen.js` bridge / bootstrap section). The README requires installing the
`stems` plugin first.

## Q2 — Where is state persisted?
**Resolved.** `localStorage` only, under two keys:
- `stem_mixer:state` — `{levels, eq, autolevel, selectedProfile}`
- `stem_mixer:profiles` — named snapshots
The backend route is a health probe; there is no server-side persistence.

## Q3 — Why an 8-band EQ at those specific frequencies?
**Resolved.** `EQ_BANDS = [60, 170, 310, 600, 1000, 3000, 6000, 12000]` matches
a conventional graphic-EQ layout (sub, low, low-mid, mid, presence, highs,
brilliance). They are constants in `screen.js` and not user-editable.

## Q4 — How does Output Autolevel work?
**Resolved.** A closed-loop integrator over an analyser node drives a master
gain. Constants are pinned in the constitution (§IV) and `DEFAULT_STATE`. RMS
target 0.16, gain clamp `[0.45, 1.7]`, smoothing 0.14. Bypassed when toggle is
off.

## Q5 — How are stem id aliases handled?
**Resolved.** `STEM_ALIASES` maps inbound names (`voice`, `vocal`, `vocals`,
…) to the canonical id used inside the mixer. Display label can differ
(`vocals` shows as **Voice**) — see FR-7 and the open clarification on
relabelling.

## Q6 — Should the global screen render when `stems` is unavailable?
**Open.** Today the screen will boot but cannot bind any audio nodes. Decide:
(a) preview-only mode, or (b) explicit "requires Stems plugin" notice.

## Q7 — What happens when a song's stems change mid-session (e.g. switching
songs)?
**Resolved.** `screen.js` listens for the `stems` plugin's lifecycle (re-reads
`stems.getState()` and rebuilds the filter chain via the `stemBootstrapTimers`
poll path). Live `gain` references are re-fetched on every song change, per
the contract `stems` documents.

## Q8 — Can profiles be exported / shared?
**Open.** Currently no UI or API for export. If added, it should remain a
client-only feature (round-trip a JSON blob through clipboard or file
download), to preserve the "no server state" principle.
