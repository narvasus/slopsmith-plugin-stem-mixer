# Analyze — Stem Mixer

## Coverage

| Area              | Spec | Plan | Code               | Notes                                       |
|-------------------|------|------|--------------------|---------------------------------------------|
| Player UI         | ✅   | ✅   | `screen.js`        | Mixer button + popover                      |
| Global screen     | ✅   | ✅   | `screen.html` + JS | Profile, EQ, autolevel, stem rows           |
| Settings page     | ✅   | ✅   | `settings.html`    | Static blurb only                           |
| Web Audio bridge  | ✅   | ✅   | `screen.js`        | Bridges `window.stems.getState()`           |
| Persistence       | ✅   | ✅   | `screen.js`        | `localStorage` only                         |
| Health probe      | ✅   | ✅   | `routes.py`        | One GET                                     |
| Tests             | ❌   | ❌   | —                  | No automated tests                          |

## Drift

- **README claim**: "8-band graphic EQ" with "EQ profiles dropdown" matches code. ✅
- **README claim**: dependency on `stems` plugin matches code (the bridge + bootstrap timer). ✅
- **Manifest**: `nav.screen = "plugin-stem-mixer"` — confirm Slopsmith core
  resolves this id correctly. The pattern matches other plugins (e.g. update-manager
  uses `plugin-update_manager`). Naming style (`-` vs `_`) is inconsistent
  across the ecosystem — a documentation gap, not a bug.
- **Settings page**: README says "Local persistence via localStorage" but the
  `settings.html` does not include the actual toggles (those live on the
  global screen). The blurb directs the user there — minor UX papercut.

## Gaps

1. **No graceful "stems missing" state** on the global screen (Q6 in
   `clarify.md`). Today the EQ silently has no effect.
2. **No `version` in the health response** — `update-manager` plugin enumerates
   installed versions but `stem_mixer` does not echo its own version.
3. **No telemetry of autolevel decisions** — when listeners report the mix
   suddenly going quiet, we have no way to confirm/deny the autolevel was
   responsible without enabling devtools breakpoints.
4. **Profile namespacing** — profiles are global; there is no "per-stem-profile"
   concept (e.g. "lead practice" vs "rhythm jam"). Out of scope but worth
   documenting.
5. **No duplicate-instance protection** beyond DOM presence checks. A second
   plugin loading order could race with `stems`' own bootstrap.

## Recommendations

- **Resolve Q6**: render a small "Requires Stems plugin" notice on the global
  screen when `window.stems` never resolves after N seconds. Cheap; eliminates
  a confusing silent failure.
- **Add `version` to health probe** so update-manager can correlate.
- **Document the `stems` ↔ `stem_mixer` API contract** in a shared place (the
  `stems` repo's README, ideally) so contract changes get reviewed jointly.
- **Consider unit tests** for `transposeChord`-style pure helpers
  (`sanitizeState`, `cloneState`) — they're trivial to test and would prevent
  regressions in localStorage migrations.
- **Capture autolevel state in a debug overlay** behind a query-string flag
  (e.g. `?stem_mixer_debug=1`).
