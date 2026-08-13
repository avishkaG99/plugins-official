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

## Progress Heartbeats

A run regularly goes 60–120 seconds with no visible output — resolving the sheet, exploring selectors, and executing a long plan all happen silently. From the outside that is indistinguishable from a hung job, so the run must say where it is.

**Capture the starting comment's ID** so later updates edit it in place instead of posting a new comment each time. One comment that changes is readable; six comments are noise.

```bash
STATUS_COMMENT_ID=$(gh api "repos/${REPO}/issues/${PR_NUMBER}/comments" \
  --jq 'map(select(.user.login=="github-actions[bot]" or (.body|startswith("🤖 **Web app test in progress**")))) | last | .id')
```

**Update it** at each phase boundary, and whenever a single step is expected to exceed ~60 seconds:

```bash
gh api --method PATCH "repos/${REPO}/issues/comments/${STATUS_COMMENT_ID}" \
  -f body="$(cat <<'EOF'
🤖 **Web app test in progress**

**Now:** <current activity>
**Done:** <phases completed>
**Next:** <what follows>

_Updated <UTC timestamp>. The full report replaces this comment when the run completes._
EOF
)" >/dev/null
```

Post an update at each of these points — they bracket the longest silences:

| Point | `Now:` reads |
|---|---|
| Worktree corrected (Phase 0.5) | `Re-pointed the workspace at the PR head (<sha>)` |
| Sheet resolved (Phase 1) | `Loaded <n> active cases from <sheet path>` |
| Chromium ready (Phase 2 step 1) | `Browser ready — exploring the app for stable selectors` |
| Execution starting | `Executing <n> test cases against <url>` |
| Every ~10 cases, or 90s, whichever first | `Executed <k>/<n> cases — <p> passed, <f> failed, <b> blocked` |
| Report composing (Phase 3) | `Composing the report` |

The periodic execution update is the important one: a 52-case plan spends most of its wall clock inside a single script invocation. When the plan is long enough that intermediate progress cannot be reported from inside the script, say so explicitly in the "Executing" update — `this step runs to completion before the next update` — so a long silence is expected rather than alarming.

If any heartbeat call fails, continue the run — a failed status update must never abort a test run. Never post heartbeat content as a new comment when `STATUS_COMMENT_ID` is known.

**Verify mode (`MODE=verify`) — post the verification variant on the issue:**

```bash
gh issue comment ${ISSUE_NUMBER} --body "$(cat <<'EOF'
🤖 **Bug verification in progress**

I'm replaying the repro steps against the deployed environment and running a decisive check against the expected result. A STILL REPRODUCIBLE / NOT REPRODUCIBLE / INCONCLUSIVE verdict will be posted here when complete — this may take a few minutes.
EOF
)"
```

If posting fails, output a single warning line and continue — do not stop the run.

---

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
