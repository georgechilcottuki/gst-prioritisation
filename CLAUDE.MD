# GST Prioritisation Framework

This repo contains a single-page HTML application (`index.html`) for the Gaming Super Tribe prioritisation framework, hosted on GitHub Pages.

The app connects to a SharePoint List on the gamingsupertribelt site via Microsoft Graph API (MSAL OAuth).

Key files:
- `index.html` — the entire app (HTML + CSS + JS, self-contained)

Deployment: push to `main` → GitHub Pages auto-deploys via Actions.

## Testing requirements
- Before marking any task complete, run the full test suite (`npm test` / `pytest` etc.)
- If tests don't exist for the code you're touching, write them first (or alongside the change), not after
- When fixing a bug, write a failing test that reproduces it *before* writing the fix
- If a change breaks an existing test, fix the regression before proceeding — don't skip or delete the test to make it pass
- After any change, tell me explicitly: "Ran X tests, Y passed, Z failed" — never just say "done"
- Never mark a task as complete if any test is failing