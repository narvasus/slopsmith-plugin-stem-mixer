# Plan — Stem Mixer (as built)

## File map

| File             | Lines | Purpose                                                                 |
|------------------|-------|-------------------------------------------------------------------------|
| `plugin.json`    | 15    | Manifest. `id: stem_mixer`, version `1.0.0`, registers nav `Stem Mixer`, declares `screen.html`, `screen.js`, `settings.html`, `routes.py`. |
| `screen.html`    | 53    | Global plugin screen: profile dropdown, autolevel toggle, per-stem rows, 8-band EQ rows, Flat-EQ button. Wrapped in Slopsmith's standard `max-w-3xl` container. |
| `screen.js`      | 1154  | All client logic: state load/save, profile CRUD, Web Audio bridge to `window.stems`, autolevel loop, in-player Mixer button + popover panel, global-screen bindings. IIFE with no exports. |
| `settings.html`  | 9     | Static blurb pointing the user at the player panel and the global screen. No controls. |
| `routes.py`      | 11    | Single `GET /api/plugins/stem_mixer/health` returning `{ok, plugin, config_dir}`. No DB, no I/O. |

## Architecture

```
              window.stems (from slopsmith-plugin-stems)
                          │  getState() → [{id, gain, audio, …}]
                          ▼
   ┌──────────────────────────────────────────────────────────┐
   │ stem_mixer screen.js                                     │
   │                                                          │
   │  bootstrap (poll until stems is present)                 │
   │  ──────► bridge audio: per-stem MediaElementSource ──┐   │
   │                                                      ▼   │
   │  build filter chain: 8 BiquadFilterNode (peaking) ──►AnalyserNode──►outputGain──►destination
   │                                                          │
   │  autolevel: rAF over analyser RMS → adjust outputGain    │
   │                                                          │
   │  state: cloneState/sanitizeState ↔ localStorage          │
   │  profiles: load/save/update/delete under PROFILES_KEY    │
   │                                                          │
   │  UI surfaces:                                            │
   │    • injectMixerButton() into #player-controls           │
   │    • injectMixerPanel() popover (per-stem sliders)       │
   │    • bind #stem-mixer-plugin-* (global screen)           │
   └──────────────────────────────────────────────────────────┘
```

## Key constants (from `screen.js`)

```js
const PLUGIN_ID = 'stem_mixer';
const STEM_KEYS = ['guitar', 'bass', 'vocals', 'drums', 'piano', 'other'];
const EQ_BANDS  = [60, 170, 310, 600, 1000, 3000, 6000, 12000];   // Hz
const STEM_LABELS = { guitar:'Guitar', bass:'Bass', vocals:'Voice', ... };
const STATE_KEY    = 'stem_mixer:state';
const PROFILES_KEY = 'stem_mixer:profiles';
const AUTLEVEL_TARGET_RMS = 0.16;
const AUTLEVEL_SMOOTHING  = 0.14;
const AUTLEVEL_MIN_GAIN   = 0.45;
const AUTLEVEL_MAX_GAIN   = 1.7;
const SHOW_EQ_UI = false;     // hidden in player panel; visible on global screen
```

## Lifecycle

1. **Plugin load** — Slopsmith executes `screen.js` once; the IIFE installs an
   observer on the player controls and starts a bootstrap timer waiting for
   `window.stems`.
2. **Stems available** — fetch `stems.getState()`, mint
   `MediaElementAudioSourceNode`s for each stem audio, build filter chain,
   wire analyser + output gain, install Mixer button.
3. **Song change** — live `gain` references are re-read; the previous filter
   chain is disconnected and rebuilt against the new stem nodes (per contract
   in `stems` plugin's API doc).
4. **Settings render** — when the user opens **Settings → Stem Mixer**,
   `settings.html` is mounted; no JS is wired beyond the explanatory copy.
5. **Global screen mount** — when the user navigates to `plugin-stem-mixer`,
   the script binds to `#stem-mixer-plugin-*` ids and reflects state to/from
   `localStorage`.

## Backend

`routes.py` is intentionally minimal:
```python
@app.get("/api/plugins/stem_mixer/health")
def stem_mixer_health():
    return {"ok": True, "plugin": "stem_mixer",
            "config_dir": str(context.get("config_dir", ""))}
```
No persistence, no DB, no async work.

## Dependencies

- **`slopsmith-plugin-stems`** (hard runtime dependency; see README and Q1 in
  `clarify.md`).
- **Slopsmith core** providing `window.playSong`, `highway`, `#player-controls`,
  `showScreen`, and the standard plugin manifest loader.

## Risks / drift watchpoints

- `stems` plugin's `getState()` shape is informal — any rename of `gain`,
  `audio`, or `id` breaks the bridge silently. Track via Q7.
- `STEM_ALIASES` does not cover non-Latin or arbitrary user-defined stems; an
  unrecognised id falls into `other` by default.
- LocalStorage quota is shared with all Slopsmith plugins; profiles are small
  (<1 KB each) but unbounded in count. No eviction policy is implemented.
