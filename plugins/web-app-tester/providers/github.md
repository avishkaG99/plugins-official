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

Move through these in order, editing the same comment each time:

| State | Body |
|---|---|
| **Starting** | `🤖 **Web app test in progress**` + the original wording |
| **Resolving** | `⚙️ Resolving the test plan…` — after the worktree check, before Phase 1 finishes |
| **Ready** | `📋 **Testing <n> cases** from \`<sheet>\` against <url>` — plan resolved, browser starting |
| **Executing** | `▶️ **<k>/<n> cases** · ✅ <p> · ❌ <f> · ⚪ <b>` — refreshed every 5 cases |
| **Reporting** | `📝 Composing the report…` |
| **Done** | `✅ **Complete** — <p>/<n> passed. See the report below.` |

Include a trailing `_Updated <UTC time>._` on every state so a stalled run is visibly stalled — an unchanged timestamp is the signal.

Worked example at the Executing state:

```
🤖 **Web app test in progress**

▶️ **25/52 cases** · ✅ 25 · ❌ 0 · ⚪ 0

**Plan:** `test-cases.csv` (52 active cases)
**Target:** https://test-runner-git-feature-product-tags-….vercel.app
**Workspace:** re-pointed to the PR head (`42d1dd7`)

_Updated 18:42:11 UTC._
```

The final report is a **separate** comment — the status comment is left at **Done** rather than being overwritten with it, so the run's trace and its result both remain readable.

**Proposed cases** are also a separate comment, since they ask the reader for a decision and must not be overwritten by a later state.

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
