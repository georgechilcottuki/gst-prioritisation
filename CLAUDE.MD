# GST Gaming Prioritisation Framework

A single-page HTML app (`index.html`) for the Gaming Super Tribe prioritisation framework, hosted on GitHub Pages at https://georgechilcottuki.github.io/gst-prioritisation/

The app connects to a SharePoint List called "GST Prioritisation" on betfairprod.sharepoint.com/sites/gamingsupertribelt via Microsoft Graph API (MSAL OAuth). localStorage is used as a fallback.

Key files:
- `index.html` — the entire app (HTML + CSS + JS, self-contained)

Deployment: push to `main` → GitHub Pages auto-deploys via Actions.

## On every session start
1. Read this file in full
2. Read `index.html` in full before making any changes
3. Never work from memory or assumptions about what the file contains

## When making any change
1. Read the relevant section of `index.html` first
2. Explain what you found and what you plan to change
3. Make the change
4. Tell me explicitly what you verified and what should be tested in the browser
5. Commit with a descriptive message and push to main
6. If you discover something new about how this codebase works, add it to the Lessons learned section below and commit it

## Testing requirements
- Before marking any task complete, verify the change works logically in the code
- When fixing a bug, identify the root cause before writing the fix
- After any change, tell me explicitly what was changed and what to test in the browser
- Never mark a task as complete without explaining what to verify
- If a change could break something else, call it out explicitly

## Tech stack
- MSAL v3 (Microsoft Authentication Library) for OAuth
- Microsoft Graph API for SharePoint List read/write
- localStorage as offline fallback
- Vanilla JS, no frameworks
- GitHub Pages hosting

## Lessons learned
- CSV import: text labels use en-dashes (–) not hyphens (-) in score maps — normalise with normStr() before lookup
- SharePoint items arrive with all numeric scores = 0; always run resolveScores() before hasTBC()
- Empty commercial/bizrisk fields from CSV = N/A (score 1), not TBC (score 0)
- Brand field can contain non-brand text from CSV — defaults to All (score 2) if unrecognised
- MSAL v3 requires initialize() before handleRedirectPromise()
- isWip must be explicitly set on item before saving to SharePoint
- Literal newlines inside JS string literals break the entire script — use \\n not a real newline
- hasTBC() must call resolveScores() first so SP items with score=0 but valid text are handled correctly
- setStatus() and showSignInButton() must be defined before the Graph module calls them — place at top of script
- CSV row joins must use a real newline escape `'\n'`, not the two-character literal `'\\n'` — the latter compiles fine but produces backslash-n text instead of a line break, so the exported file renders as one line. saveBackupToSP() had the correct form; exportCSV() didn't.
- The priority score formula is `(brand+invest+urgency+mission+vsMetric+impact+commercial+risk) * (urgency+reason) / effort` — `impact` must be included. calc() (Add-form live preview) and parseImportCSV() both omitted it, so the live preview and CSV-imported items diverged from the score actually saved. Always check new/changed score-calculating code against calcScore() (the canonical implementation) for the full field list.
- fetch() only rejects on network failure, not on HTTP error status (4xx/5xx) — every write to Graph (spSaveItem, spDeleteItem) must explicitly check `response.ok` and throw, otherwise a rejected write (auth expired, bad field, etc.) is silently treated as a successful save.
- resolveSiteAndList() is called from many places (spSaveItem, spDeleteItem, initMSAL) and only guarded re-entry via `if (_siteId && _listId)`, so concurrent callers before the first resolve completes would each fire duplicate Graph lookups. Fixed with a `_resolvePending` queue: the first caller starts the fetch, later concurrent callers are queued and all fire once the in-flight resolve finishes.
- The priority formula now lives in exactly one place: `computePriority(brand,invest,urg,mm,vs,impact,comm,risk,reason,effort)`, defined just above calcScore(). Every score-calculating call site (calc(), addInitiative(), editCalc(), saveDetail(), parseImportCSV(), repairAll() via calcScore()) must call it rather than inlining the formula — inlining it separately is what caused `impact` to silently go missing from two call sites before. If the formula ever needs to change, it should only need to change here.
- The Add-form "Live score" breakdown panel was missing a "Reason" cell even though Reason is a real scoring input included in the formula — it was tracked in calc()'s anyTBC check but never rendered. When adding/removing a scoring field, check both the calculation (computePriority call sites) AND the score-panel HTML/calc() DOM updates — they're two separate things that can each drift independently.
- CSV import now reviews for duplicates before writing anything. `parseImportCSV()` matches each row's initiative name (via `findBacklogMatchByName()`, normalised comparison against the current backlog+WIP) and also against earlier rows already seen in the same file. Each row is tagged `_dupType` (null | 'existing' | 'batch'), `_dupId`, `_dupName`, `_include` (default true for new rows, false for duplicates). The import modal shows two checklists — "New items" and "Possible duplicates" — with per-row and select-all/none toggles; nothing is written to `initiatives`/SharePoint until the user clicks "Apply".
- IMPORTANT: checked duplicate rows in the import review are added as new, independent items — same as new rows — not merged into the existing matching item. An earlier version tried to "smart merge" checked `_dupType==='existing'` rows into the matching backlog item in place; this looked like a bug ("checking a duplicate does nothing") whenever the CSV data matched what was already stored, since the merge produced no visible change. `confirmImport()` just strips the `_dup*`/`_include` fields and pushes every checked row via `initiatives.push()`, exactly like a plain new item. If a "sync latest data into an existing item via CSV" feature is wanted later, it needs to be a clearly separate, explicit action — not the default behaviour of checking a duplicate.
- `dateAdded` is now a `dd/mm/yy hh:mm:ss` timestamp via `nowStamp()` (defined near the other small helpers, esc/vv/tt/sel), not just a date, so it's possible to tell exactly when an item was added, not just the day. Set at creation time in `addInitiative()` and `parseImportCSV()` only — never touched on edit/save, so it reflects when the item first entered the backlog. Visible in the initiative detail modal and in CSV exports; no dedicated backlog-table column for it yet.
- `dateAdded` is a free-format string, not something ever parsed back into a Date — CSV export/import and the detail modal just pass it through and quote/escape it like any other text column, so changing its format (date-only vs dd/mm/yy hh:mm:ss) never risks breaking export/import. Old items created before a format change simply keep their old string until edited.
- Both exportCSV() and saveBackupToSP() build a filename timestamp — previously duplicated inline in each. Consolidated into `fileTimestamp()` (next to `nowStamp()`): a filesystem-safe `YYYY-MM-DD_HHMMSS` stamp (no slashes/colons, seconds precision) so repeat exports/backups in the same session get distinct filenames instead of overwriting.
