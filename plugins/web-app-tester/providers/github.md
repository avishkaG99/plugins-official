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

A run regularly goes 60–120 seconds with no visible output — resolving the sheet, exploring selectors, and executing a long plan all happen silently. From the outside that is indistinguishable from a hung job.

**Each milestone is its own comment.** The starting comment stays exactly as posted — never edit it. Subsequent milestones post separate comments, so the PR thread reads as a chronological record of what the run established and when. Only the periodic *execution tally* edits itself in place, because it fires repeatedly and would otherwise flood the thread.

### 1. Starting comment — post once, never edit

Already covered above. It says the run has begun and nothing more.

### 2. Test plan resolved — post after Phase 1

Post as soon as the sheet is located and parsed, before any browser work. This is the comment that tells the author what will actually be tested.

```bash
gh pr comment ${PR_NUMBER} --body "$(cat <<'EOF'
🤖 **Test plan resolved**

**Source:** `<TEST_SHEET_PATH>` (repository test case sheet)
**Active cases:** <n>
**Target:** <TEST_URL>
<if WORKTREE_CORRECTED: "**Workspace:** re-pointed to the PR head (`<sha>`) — the prepared checkout was on the default branch.">

Executing all <n> active cases now.
EOF
)"
```

When no sheet was found and the plan was auto-generated or scraped, say so plainly here instead — that is the moment the author can still intervene.

### 3. Proposed cases — post only when coverage is missing

Post when the PR adds user-facing behaviour no `Active` case covers. This is a **separate comment** from the plan and from the report, because it is the one that asks the author for a decision.

**Ask a direct question and give a clickable answer.** GitHub comments cannot render a dropdown, but task-list checkboxes are interactive — a reader ticks one in the rendered comment without editing markdown. Use one checkbox per option and state plainly that ticking is the whole action:

```bash
gh pr comment ${PR_NUMBER} --body "$(cat <<'EOF'
🤖 **New feature detected — add test cases?**

This PR adds **<feature>**, which no active case in `<sheet path>` covers.
The suite currently holds **<n> active cases**; these would make it **<n+k>**.

<one-line summary per proposed case, e.g.:>
- `TC-053` — create a saved view (Happy)
- `TC-054` — delete a saved view (Happy)

<details>
<summary>Show the exact rows that would be appended</summary>

```csv
<the full CSV rows>
```
</details>

---

**Do you want these added to `<sheet path>` and committed to this PR?**

- [ ] ✅ **Yes** — append them, commit to this branch, then run all <n+k> cases
- [ ] ❌ **No** — leave the sheet alone and test the existing <n> cases

_Tick a box, or apply the `ai-dlc/pr/test-cases-approved` label — either works.
Ignoring this is fine too: the next ordinary run tests the <n> existing cases._
EOF
)"
```

**Reading the answer.** On the next run, fetch this comment and check which box is ticked (`- [x]`). Treat the ticked "Yes" as approval, a ticked "No" as a decline, and neither ticked as no decision — which is the same as a decline for behaviour, but is reported differently (`no response` vs `declined`). If **both** are ticked, do not guess: post one line asking for a single choice and stop.

An edited CSV block in a human reply still wins over the agent's original rows.

Post nothing when coverage is complete — silence is the correct signal there; the final report states that no gap was found.

### 4. Execution tally — one comment, edited in place

This is the exception to the one-comment-per-milestone rule. It fires every 5 cases or ~45 seconds, so it edits itself rather than posting repeatedly. Create it once when execution starts, capture its ID, then PATCH it:

```bash
TALLY_ID=$(gh pr comment ${PR_NUMBER} --body "🤖 **Executing** — 0/<n> cases" 2>/dev/null \
  && gh api "repos/${REPO}/issues/${PR_NUMBER}/comments" --jq 'last | .id')

gh api --method PATCH "repos/${REPO}/issues/comments/${TALLY_ID}" \
  -f body="🤖 **Executing** — <k>/<n> cases · <p> passed · <f> failed · <b> blocked

_Updated <UTC timestamp>._" >/dev/null
```

When the run finishes, leave the tally at its final state; the report comment follows it.

### 5. Final report — post once

The full test execution report, per `styles/report-template.md`.

---

If any progress comment fails to post, log one warning line and continue. Status reporting must never abort a test run.


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
