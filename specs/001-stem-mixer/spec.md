# Spec — Stem Mixer (`stem_mixer`)

> Retrospective spec for the existing v1.0.0 plugin. The implementation in
> `screen.js`, `screen.html`, `settings.html`, and `routes.py` is the source of
> truth; this document describes what is shipped today and flags ambiguities.

## Summary

A Slopsmith plugin that adds a single **Mixer** button to the player control
bar and a global plugin screen. The player-side panel hosts compact per-stem
volume sliders. The global screen hosts an 8-band graphic EQ, a named profile
dropdown (save / update / delete), and an Output Autolevel toggle. State is
persisted to `localStorage`. Depends on `slopsmith-plugin-stems` for the
underlying multi-stem `<audio>` graph.

## User stories

### US-1 — Adjust per-stem levels mid-song
**As a** player listening to a sloppak with multiple stems
**I want to** open the Mixer panel from the player and pull a stem's volume up
or down
**so that** I can hear the part I'm practising more clearly without leaving
the song.

- **Given** a sloppak song is playing and `slopsmith-plugin-stems` is loaded,
  **When** I click the **Mixer** button injected into `#player-controls`,
  **Then** a popover with one slider per discovered stem (Guitar / Bass /
  Voice / Drums / Piano / Other) appears anchored to the button.
- **Given** the panel is open, **When** I drag a stem slider,
  **Then** the corresponding `GainNode.gain.value` updates immediately and the
  new level is persisted under `localStorage["stem_mixer:state"].levels`.
- **Given** I close and re-open the panel on the same song,
  **Then** my last levels are restored without re-reading the song.

### US-2 — Apply a global EQ curve
**As a** user with a bass-heavy headphone setup
**I want to** trim the low end across all songs
**so that** I don't have to fight a muddy mix on every track.

- **Given** I open **Plugins → Stem Mixer**,
  **When** I drag any of the 8 band sliders (60, 170, 310, 600, 1000, 3000,
  6000, 12000 Hz),
  **Then** a chain of `BiquadFilterNode` peaking filters between the stem
  output and `AudioContext.destination` is updated in real time.
- **Given** I click **Flat EQ**,
  **Then** all 8 bands return to 0 dB and the change persists.

### US-3 — Save and switch named EQ profiles
- **Given** I have set bands and stem levels I like,
  **When** I click **Save** and supply a name,
  **Then** the snapshot is stored under `localStorage["stem_mixer:profiles"]`
  and appears in the profile `<select>`.
- **Given** I select a different profile,
  **Then** stem levels and EQ snap to the saved values immediately.
- **Given** I edit the active profile and click **Update**,
  **Then** the snapshot is overwritten in place. **Delete** removes it.

### US-4 — Output Autolevel
- **Given** I toggle **Output autolevel** on (either from the player panel or
  the global screen),
  **When** content plays through the stem mixer chain,
  **Then** an analyser drives a closed-loop integrator that adjusts the master
  output gain so RMS tracks `AUTLEVEL_TARGET_RMS = 0.16`, clamped to
  `[0.45, 1.7]`, smoothed at `0.14`.
- The toggle state syncs across both UIs and persists across reloads.

### US-5 — Inert without `stems`
- **Given** `slopsmith-plugin-stems` is **not** installed,
  **Then** the player-controls Mixer button MUST NOT appear and no Web Audio
  graph is constructed (degrade silently — the global screen may still render
  read-only). [NEEDS CLARIFICATION: should the global screen show a "stems
  required" notice when bridging fails?]

### US-6 — Health probe
- **Given** the backend is up,
  **When** a tool calls `GET /api/plugins/stem_mixer/health`,
  **Then** it responds `{"ok": true, "plugin": "stem_mixer", "config_dir": "..."}`.

## Functional requirements

| ID    | Requirement                                                                                          | Source                          |
|-------|------------------------------------------------------------------------------------------------------|---------------------------------|
| FR-1  | Inject one **Mixer** button into `#player-controls` when the player screen is active and stems are present. | `screen.js` (player UI section) |
| FR-2  | Provide a 6-stem panel with sliders for `STEM_KEYS = [guitar, bass, vocals, drums, piano, other]`.   | `screen.js` `STEM_KEYS`         |
| FR-3  | Provide an 8-band peaking EQ at `EQ_BANDS = [60, 170, 310, 600, 1000, 3000, 6000, 12000] Hz`.         | `screen.js` `EQ_BANDS`          |
| FR-4  | Persist per-tab state under `localStorage["stem_mixer:state"]` with `{levels, eq, autolevel, selectedProfile}`. | `screen.js` `STATE_KEY`         |
| FR-5  | Persist named profiles under `localStorage["stem_mixer:profiles"]` (object keyed by name).           | `screen.js` `PROFILES_KEY`      |
| FR-6  | Bridge to `window.stems.getState()` for live `gain` references; tolerate the API being undefined at boot via `stemBootstrapTimers`. | `screen.js`                     |
| FR-7  | Map alias stem ids (`voice ↔ vocals`, etc.) via `STEM_ALIASES`.                                       | `screen.js`                     |
| FR-8  | Output Autolevel uses the constants in Constitution §IV.                                             | `screen.js`                     |
| FR-9  | Expose `GET /api/plugins/stem_mixer/health` returning `{ok, plugin, config_dir}`.                    | `routes.py`                     |
| FR-10 | Render `settings.html` under **Settings → Stem Mixer** as an explanatory blurb only (no controls).   | `settings.html`                 |
| FR-11 | Mark the EQ UI hidden by default (`SHOW_EQ_UI = false` in player panel; global screen still shows it). | `screen.js`                     |
| FR-12 | Use the sentinel `<span style="display:none">` pattern when injecting controls, so other plugins' `button:last-child` lookups are not perturbed. | `screen.js` (sentinel comment)  |

## Non-functional requirements

- **Latency**: slider drag → audible level change ≤ 16 ms (one frame).
- **Memory**: do not leak `BiquadFilterNode`s on song change — chain is rebuilt
  via `filterChain` array and disconnected on teardown.
- **No telemetry**: no network egress beyond the health probe.

## Out of scope

- Server-persisted settings or per-song mix recall (handled by `stems` already
  via its own keys).
- A user-facing UI for editing the EQ band centres.
- Automatic loudness analysis offline (autolevel is real-time only).

## Open clarifications

- [NEEDS CLARIFICATION] When `window.stems` is missing entirely, should the
  global screen still render the EQ in a "preview only" mode, or refuse to
  initialise?
- [NEEDS CLARIFICATION] Profile sharing across browsers / devices is not
  supported — is that a permanent stance or a future feature?
- [NEEDS CLARIFICATION] Should the **Voice** label keep diverging from the
  underlying `vocals` stem id (FR-7), or rename for consistency with `stems`?
