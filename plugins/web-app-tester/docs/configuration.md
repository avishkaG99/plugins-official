# Configuration: `.web-app-tester.json`

A consumer repo may place a `.web-app-tester.json` file at its root to give the plugin named environments, authenticated sessions, and read-only enforcement. **The file is never required** — when absent, the plugin behaves exactly as 1.0 (URL scraped from comments, unauthenticated, substring-heuristic read-only check).

**Credentials never go in this file** — only paths and URLs.

## Example

```json
{
  "defaultEnvironment": "staging",
  "environments": {
    "staging": {
      "baseUrl": "https://staging.example.com",
      "mutationsAllowed": true,
      "storageStates": { "admin": "tests/e2e/.auth/admin.json", "user": "tests/e2e/.auth/user.json" },
      "defaultRole": "admin"
    },
    "prod": { "baseUrl": "https://app.example.com", "mutationsAllowed": false }
  },
  "authSetupCommand": "npm --prefix tests/e2e run auth:refresh"
}
```

## Schema

| Field | Required | Meaning |
|---|---|---|
| `environments.<name>.baseUrl` | yes | Base URL for the environment |
| `environments.<name>.mutationsAllowed` | no (default `false`) | `false` = read-only mode for this env — data-modifying test cases are skipped with reason `Skipped — environment is read-only` (same enforcement as the legacy `PRODUCTION_WARNING`) |
| `environments.<name>.storageStates` | no | Map of role → Playwright storage-state file path (relative to repo root) |
| `environments.<name>.defaultRole` | no | Role used when the run doesn't demand a specific one via `--role` |
| `defaultEnvironment` | no | Environment used when no `--env`/`--url` argument is given |
| `authSetupCommand` | no | Command the plugin may run (at most once per run, from the repo root) to (re)generate storage-state files when they are missing or the app rejects the session |
| `testSheet` | no | Path to the repository's committed test case sheet (CSV). When set, the sheet becomes the test plan and outranks comment-scraping and auto-generation. When unset, the plugin still discovers a sheet by convention — see below. |

## Repository Test Case Sheet

When a repo commits a test case sheet, that sheet is the source of truth for what gets tested — the plugin executes it instead of scraping a plan from comments or generating a happy-path substitute.

Discovery order (first match wins):

1. `testSheet` from this config
2. `test-cases.csv` at the repo root
3. `docs/test-cases.csv`
4. Any `*test-case*.csv` at the repo root or in `docs/`

Required columns: `ID`, `Feature`, `Title`, `Type`, `Steps`, `Expected Result`, `Notes`, `Status`, `Added In`.

**Every row whose `Status` is `Active` runs, every time.** The plugin does not subset by feature or by relevance to the diff — a new feature is validated against the whole suite, not just its own cases. `Draft` and `Retired` rows never run.

After the run, the plugin compares the PR's changes against the sheet's coverage and proposes ready-to-paste rows for any uncovered user-facing behaviour. **It never edits the sheet** — proposals go in the report comment and a human appends them.

`PLAN_SOURCE` reads `sourced from repository test case sheet (<path>)` when this path was taken. Any other value means the sheet was not picked up.

## URL Resolution Precedence (all modes)

First match wins:

1. `--url` argument
2. `--env` argument → that environment's `baseUrl`
3. `defaultEnvironment` → that environment's `baseUrl`
4. Comment-scraping (`Preview URL:`, `Staging URL:`, `Deploy preview:`, … — the 1.0 behaviour)
5. Stop with "no testable URL"

**Read-only determination:** when the URL came from config (paths 2–3), `mutationsAllowed` is authoritative and the URL substring heuristic (`staging`, `preview`, `dev`, `test`, `localhost`) is skipped. The heuristic still applies to scraped and `--url` URLs.

## Storage States

A storage state is Playwright's serialized session (cookies + local storage) for a signed-in user, produced by `context.storage_state(path="...")` (Python) / `context.storageState({ path })` (Node.js). The natural generator is your repo's existing E2E global-setup — the script that logs each test role in and saves its session — pointed at the same file paths this config references.

Roles are free-form names (`admin`, `user`, `super`, …). A run picks its role via `--role <role>`, falling back to the environment's `defaultRole`. The role's storage-state path is handed to the browser context; the plugin never reads or prints the file's contents.

## `authSetupCommand` Contract

- Run from the repo root.
- Must be **non-interactive** — no prompts; credentials come from the environment/secret store your E2E setup already uses.
- Must exit `0` on success, having (re)generated every storage-state file the config references.
- The plugin runs it **at most once per run**, only when a storage state is missing or the app rejects the session. If the gate persists after one attempt, steps are BLOCKED with `Auth session rejected — storage state stale and setup command did not recover it`.

## Security Notes

- **Gitignore every storage-state file** the config points at — they contain live session cookies/tokens.
- No credentials, secrets, or tokens in `.web-app-tester.json` — paths and URLs only.
- The plugin never prints storage-state contents (cookie or token values) to logs, comments, generated scripts, or output. File *paths* may appear; contents may not.
- Generated scripts receive the storage-state path via `browser.new_context(storage_state=...)` — they never inline the session data.
