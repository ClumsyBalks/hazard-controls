# Hazard Controls Generator (HCG) — The Ponds HS

Single-file web app (`hazard-controls.html`) that generates subcontractor-specific
daily hazard controls for the Lipman project at The Ponds High School, NSW
(Project #72601), formatted for pasting into HammerTech. Owner: Cameron MacArthur
(Site Manager, Lipman).

## Deployment

- Hosted on GitHub Pages from this repo's `main` branch:
  `https://clumsybalks.github.io/hazard-controls/hazard-controls.html`
- To ship a change: edit `hazard-controls.html`, push to `main`. Pages rebuilds in
  1–2 minutes; the user refreshes the pinned tab. Never deliver a fork of the file
  that bypasses this repo — the repo is the single source of truth.
- Validate before pushing: extract the `<script>` block and run `node --check`.
  For UI changes, render with Playwright and inspect a screenshot.

## Critical technical facts (do not re-litigate)

- Anthropic API direct from browser. Header MUST be
  `"anthropic-dangerous-direct-browser-access": "true"` — the `-calls` variant
  fails silently and cost days of debugging.
- Model: `claude-fable-5` with `output_config: {effort: "high"}` for both control
  generation and meeting-draft/whiteboard extraction. Verify any model string
  against current Anthropic docs before changing it.
- CSP in the page header allowlists `connect-src` to `api.anthropic.com`,
  `api.github.com`, `raw.githubusercontent.com` (and `img-src` includes `blob:`
  for the photo downscaler). Any new external call needs a CSP update.
- `captureActivities()` must run before EVERY operation that re-renders trade
  cards — textareas are wiped by `innerHTML` rebuilds otherwise.
- Android Chrome file inputs: full-size, `opacity:0`, absolutely positioned over a
  visible button. `display:none` inputs silently fail.
- Clipboard: always pair `navigator.clipboard.writeText` with the
  `execCommand("copy")` fallback plus visible feedback.
- API JSON replies: strip code fences and take the outermost `{...}` before
  parsing; check `stop_reason === "max_tokens"`.

## Browser-side state (per device, per origin)

- localStorage: `tphs_apikey` (Anthropic key), `tphs_subbies_v2` (subcontractors:
  `{name, htName, aliases[], benched}` — display name on boards, HammerTech
  register name and learned aliases in background), `tphs_docs_v4` (documents),
  `tphs_trades_v2` (day's trade cards), `tphs_locations_v1` (zones list),
  `tphs_ghrepo_v1` (SWMS repo config incl. fine-grained PAT).
- IndexedDB `tphs_files`: original PDFs of locally uploaded documents.
- Export/Import settings JSON (schema v4) is the cross-device sync; it does not
  carry the API key, PAT, or IndexedDB file blobs.

## SWMS repository sync

The app syncs documents from the private repo `ClumsyBalks/HammerTech`, branch
`claude/automate-hammertech-forms-mnj8p`, folder `reference/SWMS` (plus sibling
`reference/PRA` for the always-applied Project Risk Assessment — latest revision
wins). Conventions are documented in that folder's README.md and must be kept:
`manifest.json` metadata is preferred over filename prediction; `SUPERSEDED_`
prefixed files are ignored; supersede comparisons use document date then
revision (handles `Rev B`, `Rev 3-2`, `Rev 2026-1`, `(yyyymmdd)` dates). Branch
names containing slashes are resolved by probing branch/path splits. Manual
assignment of a synced doc teaches the app the legal-entity → display-name link
(stored as `htName`/alias) so future SWMS from that employer auto-assign.

## 1pm meeting integration

The "1pm Meeting" card ingests the typed meeting draft (PDF/TXT) or a whiteboard
photo. The draft is cloned from the previous day with only per-contractor work
activities and deliveries updated — extraction ignores carried-over panels
(compliance, programme, pours). Extraction returns strict JSON:
`{date, weather, deliveries, trades:[{subcontractor, tasks[], zones[]}], uncertain[]}`.
Zones are labels like `Building E - Level 1`; known zones tick automatically and
replace the previous day's ticks, new zones queue for one-click confirmation.
The extraction sets the day's roster: trade cards not on the draft are removed
(never on a failed/empty read). Trade cards render HammerTech-style columns
(Activity / Controls / Zones / Employer responsible), each with its own Copy
button; the employer dropdown reassigns a card and clears stale results.

## Related

- The daily 1pm meeting boards (Bluebeam TV board, Daily Activities Board) are
  built in a separate Claude project that consumes this app and the SWMS repo.
- Output limits for generated controls: 100 words target, 150 hard max, bullet
  list only. Copy includes a `Name - Hazard Controls (dd/mm/yyyy):` header line
  taken from the app's site date (daily evidence trail).
