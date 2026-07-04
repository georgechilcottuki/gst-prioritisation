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
