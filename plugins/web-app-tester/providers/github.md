# Provider: GitHub

Use this provider when `git remote get-url origin` contains `github.com`.

## How This Fits with the Rest of the Plugin

- **Reading** — Use `gh` to fetch PR/issue metadata, comments, linked issues, and commit messages.
- **Posting** — Use `gh pr comment` or `gh issue comment` to post the test execution report and any interim notices (no URL found, auto-generated plan).

GitHub does not support file attachments on issue/PR comments, so screenshots are described inline as "Screenshot captured at point of failure" rather than embedded as files.

## Prerequisites

The `gh` CLI must be installed and authenticated. Verify with:

```bash
gh auth status
```

If not authenticated, run `gh auth login` or set the `GITHUB_TOKEN` environment variable.

### Token Permissions

| Permission | Access | Why it's needed |
|---|---|---|
| **Metadata** | Read | Resolve repository owner and name |
| **Issues** | Read & Write | Fetch issue body and comments; post result comment |
| **Pull requests** | Read & Write | Fetch PR body, commits, and comments; post result comment |

---

## Resolving Owner and Repo

```bash
REMOTE=$(git remote get-url origin)
OWNER=$(echo "$REMOTE" | sed 's|https://github.com/||;s|git@github.com:||' | cut -d'/' -f1)
REPO=$(echo "$REMOTE"  | sed 's|https://github.com/||;s|git@github.com:||' | cut -d'/' -f2 | sed 's|\.git$||')
```

---

## Fetching PR Content

```bash
gh pr view ${PR_NUMBER} --json number,title,body,state,headRefName,baseRefName,url,author,labels,commits,closingIssuesReferences,comments
```

Fetching comments separately if needed:
```bash
gh api "repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments" \
  --jq '.[].body'
```

Linked issues:
```bash
gh pr view ${PR_NUMBER} --json closingIssuesReferences --jq '.closingIssuesReferences[].number'
```

---

## Fetching Issue Content

```bash
gh issue view ${ISSUE_NUMBER} --json number,title,body,state,labels,assignees,comments,projectItems
```

Linked PRs from an issue:
```bash
gh api "repos/${OWNER}/${REPO}/issues/${ISSUE_NUMBER}/timeline" --paginate \
  --jq '.[] | select(.event=="cross-referenced" or .event=="closed") | .source.issue.number // empty'

gh pr list --search "${ISSUE_NUMBER} in:body" --state all \
  --json number,title,state,headRefName,url,body --limit 20
```

---

## Posting the "Test in Progress" Comment

Post a single starting comment on the entry artefact immediately after platform detection so the author knows the web app test run has started and that it can take several minutes (Playwright install + browser session) to complete.

**PR:**
```bash
gh pr comment ${PR_NUMBER} --body "$(cat <<'EOF'
🤖 **Web app test in progress**

I'm installing Playwright if needed, launching a browser session, and executing the test plan against the deployed app. The full test execution report will be posted as a comment when complete — this may take a few minutes.
EOF
)"
```

**Issue:**
```bash
gh issue comment ${ISSUE_NUMBER} --body "$(cat <<'EOF'
🤖 **Web app test in progress**

I'm installing Playwright if needed, launching a browser session, and executing the test plan against the deployed app. The full test execution report will be posted as a comment when complete — this may take a few minutes.
EOF
)"
```

## Progress Comments

A run goes 60–120 seconds at a time with no visible output. From the outside that is indistinguishable from a hang, so the run must show where it is.

**One status comment, edited through the run.** The starting comment *is* the status comment — capture its id and PATCH it at every state change rather than posting again. Editing keeps the thread to two comments (status + final report) instead of six, and the reader watches one place.

```bash
gh api --method PATCH "repos/${REPO}/issues/comments/${STATUS_ID}" \
  -f body="<new body>" >/dev/null
```

### The states

**The first line never changes.** Every state keeps the exact heading `🤖 **Web app test in progress**` — that heading is the key `find_comment` matches on, so altering it makes the next update post a new comment instead of editing this one. The state goes on the *second* line.

Body template — fill the placeholders, change nothing else:

```
🤖 **Web app test in progress**

<STATE LINE>

**Plan:** `<sheet>` (<n> active cases)
**Target:** <url>
<optional: **Workspace:** re-pointed to the PR head (`<short sha>`)>

_Updated <ISO 8601 UTC>._
```

`<STATE LINE>` is exactly one of:

| State | Line |
|---|---|
| Starting | `⏳ Starting — installing Playwright and launching a browser.` |
| Resolving | `⚙️ Resolving the test plan…` |
| Ready | `📋 Ready — executing <n> cases.` |
| Executing | `▶️ **<k>/<n> cases** · ✅ <p> passed · ❌ <f> failed · ⚪ <b> blocked` |
| Reporting | `📝 Composing the report…` |
| Done | `✅ Complete — <p>/<n> passed. See the report below.` |

**Rules for the Executing line, which repeats:**

- Keep the field order `passed · failed · blocked` every time. Do not reorder.
- Emit counts only — no parentheticals, no commentary, no "effective" or "auto-corrected" notes. Those belong in the final report, not a status line.
- Counts must be monotonic: `<k>`, `<p>`, `<f>`, `<b>` never decrease between updates. A number going backwards means the tally is being recomputed rather than accumulated — recount from the log before posting.
- Keep `_Updated …_` as the last line in every state; it is how a stalled run is spotted.

The final report is a **separate** comment — the status comment is left at **Done** rather than being overwritten with it, so the run's trace and its result both remain readable.

**Proposed cases** are also a separate comment, since they ask the reader for a decision and must not be overwritten by a later state.

**Every comment in this section is post-once.** Before posting any of them, query for an existing one and edit that instead:

```bash
find_comment() {  # $1 = unique prefix, e.g. "🤖 **Test plan resolved**"
  gh api "repos/${REPO}/issues/${ENTRY_ID}/comments" \
    --jq --arg p "$1" '[.[] | select(.body | startswith($p))] | last | .id // empty'
}
```

If it returns an id, PATCH that comment; if it returns nothing, post a new one.

The prefixes are fixed — use these exact strings, and never let a body's first line drift from them:

| Comment | Prefix |
|---|---|
| Status (all states) | `🤖 **Web app test in progress**` |
| Test plan resolved | `🤖 **Test plan resolved**` |
| Proposed cases | `🤖 **New feature detected` |

A body whose first line no longer matches its prefix is unfindable, and the next update posts a duplicate instead of editing. That is the single most common cause of a thread filling with near-identical comments.

If any edit fails, log one warning and continue. Status reporting never aborts a run.


## Posting the "No URL Found" Comment

**PR:**
```bash
gh pr comment ${PR_NUMBER} --body "🤖 web-app-tester could not run — no testable URL was found.
Add a comment with the URL (e.g. Preview URL: https://...) and re-trigger."
```

**Issue:**
```bash
gh issue comment ${ISSUE_NUMBER} --body "🤖 web-app-tester could not run — no testable URL was found.
Add a comment with the URL (e.g. Preview URL: https://...) and re-trigger."
```

---

## Posting the Auto-Generated Plan Comment

**PR:**
```bash
gh pr comment ${PR_NUMBER} --body "$(cat <<'EOF'
🤖 web-app-tester — No test plan found. Auto-generated plan, executing now:

${AUTO_GENERATED_STEPS}
EOF
)"
```

**Issue:**
```bash
gh issue comment ${ISSUE_NUMBER} --body "$(cat <<'EOF'
🤖 web-app-tester — No test plan found. Auto-generated plan, executing now:

${AUTO_GENERATED_STEPS}
EOF
)"
```

---

## Posting the Test Execution Report

Construct the full report body following `styles/report-template.md`, then post it.

**PR:**
```bash
gh pr comment ${PR_NUMBER} --body "$(cat <<'EOF'
${REPORT_BODY}
EOF
)"
```

**Issue:**
```bash
gh issue comment ${ISSUE_NUMBER} --body "$(cat <<'EOF'
${REPORT_BODY}
EOF
)"
```

---

## Posting the Verdict Comment (verify mode)

Post the rendered verdict template (see `styles/verdict-template.md`) **on the issue itself**:

```bash
gh issue comment ${ISSUE_NUMBER} --body "$(cat <<'EOF'
${VERDICT_BODY}
EOF
)"
```

GitHub comments do not support file attachments via the CLI, so the decisive screenshot is **not** uploaded — the decisive observation sentence in the verdict body carries the evidence inline (same convention as test-mode failure screenshots).

For the interactive-only state-transition offer (see `skills/post-verdict-report/SKILL.md`), the equivalent commands are `gh issue reopen ${ISSUE_NUMBER}` (STILL REPRODUCIBLE) and `gh issue close ${ISSUE_NUMBER}` (NOT REPRODUCIBLE) — apply only on a second explicit user confirmation, never autonomously.

---

## Output

On completion:
```
MODE=test:   web-app-tester complete for {ENTRY_TYPE} #{ENTRY_ID}: {OVERALL_RESULT} — {PASSED}/{TOTAL} test cases passed
MODE=verify: verify-bug complete for {ENTRY_TYPE} #{ENTRY_ID}: {VERDICT}
```
