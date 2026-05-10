# Tasks — Stem Mixer

Status legend: `DONE` (shipped in v1.0.0), `OPEN` (not yet implemented), `[P]` (parallelisable with siblings).

## US-1 — Per-stem volume in player

- [DONE] Inject **Mixer** button into `#player-controls` (sentinel-aware).
- [DONE] Render popover with one slider per `STEM_KEYS` entry, labels from `STEM_LABELS`.
- [DONE] Wire slider → `gain.gain.value` via `stems.getState()` references.
- [DONE] Persist `levels` to `localStorage["stem_mixer:state"]` on change.
- [DONE] Restore levels on song reload from the same key.

## US-2 — Global 8-band EQ

- [DONE] Build chain of 8 `BiquadFilterNode` (peaking) at `EQ_BANDS`.
- [DONE] Render 8 vertical sliders into `#stem-mixer-plugin-eq-rows`.
- [DONE] **Flat EQ** button resets all bands to 0 dB and persists.
- [DONE] Disconnect / rebuild chain on song change.
- [OPEN] Q-factor exposure or per-band UI for users who want narrower notches.

## US-3 — Profiles

- [DONE] Save profile (prompt for name) under `localStorage["stem_mixer:profiles"]`.
- [DONE] Update / Delete the active profile.
- [DONE] Profile select sync between player popover and global screen.
- [OPEN] Export / import profiles as JSON (Q8).

## US-4 — Output Autolevel

- [DONE] Analyser node attached after filter chain.
- [DONE] rAF integrator with constants from constitution §IV.
- [DONE] Toggle reflected in both UIs and persisted.
- [OPEN] Surface current applied gain in the UI for transparency.

## US-5 — Inert without `stems`

- [DONE] Player Mixer button is suppressed when `stems` is missing.
- [DONE] Bootstrap timer (`stemBootstrapTimers`) polls until `stems` is present without busy-looping after success.
- [OPEN] [P] Decide and implement the global-screen behaviour when `stems` never appears (Q6).

## US-6 — Health probe

- [DONE] `GET /api/plugins/stem_mixer/health` returns `{ok, plugin, config_dir}`.
- [OPEN] [P] Add `version` field sourced from `plugin.json` for upgrade-manager parity.

## Cross-cutting

- [DONE] Idempotent injection (`hideStylesInstalled`, button presence check).
- [DONE] Sentinel `<span>` to avoid breaking sibling plugins' `button:last-child` lookups.
- [OPEN] Automated test harness (none today; UI-only changes are smoke-tested manually).
- [OPEN] Lint / format hooks aligned with the wider Slopsmith plugin set.
