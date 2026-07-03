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
