---
name: apply-proposed-cases
description: Coverage-first flow for web-app-tester. Handles --propose-only (detect uncovered behaviour from the PR's changed files, post reviewable CSV rows with a clickable approval, and stop without opening a browser) and --apply-cases (read the approval, append the approved rows to the repository test case sheet, commit them to the PR head branch as a single-file commit, then run the full enlarged suite). Only loaded when one of those two flags is present.
disable-model-invocation: true
---

# Coverage-First Flow

This skill is invoked by the **orchestrator** agent, and only when `PROPOSE_ONLY` or `APPLY_CASES` is set. A default run never reads it.

By default a run tests the sheet as it stands and proposes uncovered cases at the end, leaving the sheet untouched. The coverage-first flow instead settles coverage **before** testing, so the full suite that runs already includes the PR's new behaviour. It is two runs with a human decision between them.

### Run 1 — `--propose-only`

Establish coverage, propose, and stop. Do **not** open a browser and do **not** execute any test case; this run exists to produce a reviewable list cheaply.

1. Complete Phase 0.5 (worktree check) and Phase 1 up to sheet resolution — enough to know the sheet's contents and the next unused ID.
2. Derive uncovered behaviour from `CHANGED_FILES` exactly as `skills/post-test-report/SKILL.md` §2f describes, **except** that you may not confirm behaviour in a running application — no browser runs in this mode. Base proposals on the changed files and the sheet alone, and say so.
3. Post one comment titled `## 🤖 Proposed Test Cases — awaiting approval` containing: what the PR adds, why existing coverage is insufficient, the ready-to-paste CSV rows, and an explicit instruction for how to approve.
4. Stop. Report `PLAN_SOURCE`, the sheet path, the active-case count, and the proposed IDs. Emit no verdict — nothing was tested.

If no uncovered behaviour is found, post that finding plainly and stop; there is nothing to approve.

**Approval is optional, and declining costs nothing.** These are two independent triggers, not a blocking gate:

- **Approve** → the `--apply-cases` run appends the rows, commits them, and tests the enlarged suite.
- **Do nothing, or decline** → the ordinary label runs whenever you choose, tests the sheet exactly as it stands, and posts a normal report. The proposal comment simply remains on the PR as an unactioned suggestion.

Never treat a missing approval as a failure, never re-post the proposal to chase a decision, and never block a plain run because a proposal is outstanding. A plain run whose sheet still lacks the proposed cases notes that in the report — it does not refuse to run or mark the PR incomplete.

The approval instruction in the comment must name the exact mechanism the repo uses. Default wording:

```
To approve: apply the `ai-dlc/pr/test-cases-approved` label to this PR.
The next run will append these rows to <sheet path>, commit them to this
branch, and execute the full suite including them.

To reject or amend: edit the rows in a reply comment before applying the
label — the run reads the most recent proposal comment.
```

### Run 2 — `--apply-cases`

Apply the approved rows, then test everything.

1. Complete Phase 0.5 so the worktree is at the PR head — the commit must land on the PR's branch, not the default branch.
2. **Claim the proposal before doing anything else.** A ticked box triggers this run by *editing* the proposal comment, so that comment is both the instruction and the trigger. Every later edit — including the approval echo in step 3 — would re-match the rule and start another run. Write a consumed marker first, and the rule's `comment.body!~'<!-- wat:consumed -->'` clause stops that.

   ```bash
   PROPOSAL_ID=$(gh api "repos/${REPO}/issues/${ENTRY_ID}/comments" \
     --jq '[.[] | select(.body | startswith("🤖 **New feature detected"))] | first | .id // empty')
   BODY=$(gh api "repos/${REPO}/issues/comments/${PROPOSAL_ID}" --jq '.body')

   case "$BODY" in
     *"<!-- wat:consumed -->"*)
       echo "Proposal already consumed — another run is handling it. Stopping."; exit 0 ;;
   esac

   # 1. record which box was ticked
   case "$BODY" in
     *"[x] ✅"*) ANSWER=yes ;;
     *"[x] ❌"*) ANSWER=no  ;;
     *)          ANSWER=none ;;   # reached via label or /approve-cases
   esac

   # 2. delete BOTH checkbox lines and replace them with a plain result line
   RESOLVED=$(printf '%s' "$BODY" \
     | sed '/^- \[[ x]\] ✅/d; /^- \[[ x]\] ❌/d' \
     | sed '/^\*\*Do you want these added/d; /^_Tick a box/,/^$/d' \
     | sed '/^---$/d' \
     | cat -s)
   gh api --method PATCH "repos/${REPO}/issues/comments/${PROPOSAL_ID}" \
     -f body="${RESOLVED}

**Answered: <✅ Yes | ❌ No> — by @<login> at <UTC time>.**
<what happens next, one line>
<!-- wat:consumed -->" >/dev/null
   ```

   **Strip the whole question, not just the boxes.** The prompt line ("Do you want these added…"), the instruction line ("Tick a box above…") and the `---` divider all become false once an answer is in — they read as though a decision is still open. Remove them along with the boxes and collapse the blank runs, so the resolved comment states what was proposed and what was decided, and nothing else.

   **Delete the checkboxes — do not merely untick them.** Once an answer is in, no checkbox may remain in the body. A box that is still rendered invites another click: the reader expects something to happen, the `wat:consumed` marker silently swallows it, and the comment now lies about its own state. Removing the lines makes a second click impossible rather than ignored.

   `ANSWER` is what the rest of this skill reads. After this step the comment body is a record, not an input.

   **Stop if there is no proposal comment at all.** An empty `PROPOSAL_ID` means `--apply-cases` was triggered without anything to apply — the label was added before a run proposed cases, or the comment was deleted. Post one line — `🤖 No proposed test cases found on this PR. Run the plain test label first.` — and exit. Never guess rows to append.

   **Stop the run if the marker is already present.** That means a concurrent or earlier dispatch has claimed this proposal; continuing would append the rows twice. This check is the only thing preventing a tick from looping, so it runs before the rows are read, before the commit, and before any browser opens.

3. **Locate the approved rows and confirm the answer.** Read the proposal comment fetched above and inspect its checkboxes:

   ```bash
   gh api "repos/${REPO}/issues/${ENTRY_ID}/comments" \
     --jq 'map(select(.body|contains("New feature detected"))) | last | .body'
   ```

   | State | Meaning | Action |
   |---|---|---|
   | `- [x] ✅ **Yes**` | Approved | Append, commit, run the enlarged suite |
   | `- [x] ❌ **No**` | Declined | Do not touch the sheet; run the existing cases and note the decline |
   | Neither ticked | No response | Same as declined, but reported as `no response` |
   | Both ticked | Ambiguous | Post one line asking for a single choice and stop |

   Reaching this step via the approval **label** counts as a Yes even when no box is ticked — the label is the alternative approval path, not a second gate.

   If a later human reply contains an edited CSV block, that reply wins — the human's version is authoritative over the agent's original.

4. **Echo the decision back before acting on it.** Edit the proposal comment to record what was read, so the choice is visible and auditable rather than inferred from a later commit. Append a resolution block to the existing body (keep the original question and rows intact, so the thread still shows what was asked):

   ```bash
   gh api --method PATCH "repos/${REPO}/issues/comments/${PROPOSAL_COMMENT_ID}" \
     -f body="${ORIGINAL_BODY}

---

**✅ Approved** by @<login> at <UTC timestamp> — via <ticked checkbox | \`ai-dlc/pr/test-cases-approved\` label>.

Appending \`<TC-0NN>\`, \`<TC-0NN>\` to \`<sheet path>\` and running all <n+k> cases. The sheet change lands as its own commit on this branch." >/dev/null
   ```

   Use the matching wording for the other outcomes — `**❌ Declined**`, `**➖ No response**` — each naming what happens next (`testing the existing <n> cases; the sheet is unchanged`). Post this **before** the commit, so a run that dies mid-way still leaves a record of what it decided.

   This edit is the one permitted change to the proposal comment; never rewrite the question or the proposed rows themselves.
5. **Validate every row before writing.** Each must have all sheet columns, an ID that is unused and continues the sequence, `Status=Active`, and `Added In=PR#<n>`. Reject the batch and stop if any row is malformed, duplicates an existing ID, or changes an existing row — appending is the only permitted edit.
6. **Append and commit** to the PR head branch:

   ```bash
   git checkout -q "${PR_HEAD_BRANCH}"
   # append validated rows to the sheet, preserving its exact CSV dialect and trailing newline
   git add "${TEST_SHEET_PATH}"
   git commit -m "$(cat <<'MSG'
test: add cases for <feature> (PR #<n>)

Appends <TC-0NN>..<TC-0NN> covering <feature>, proposed by web-app-tester
and approved on the PR. Existing rows are untouched.
MSG
)"
   git push origin "${PR_HEAD_BRANCH}"
   ```

   **Stage the sheet and nothing else.** `git add` names the sheet path explicitly — never `git add -A` or `.`. The commit must contain exactly one changed file, so the sheet update is reviewable on its own and can be reverted without disturbing the PR's feature work. Verify before pushing:

   ```bash
   git show --stat --oneline HEAD | tail -n +2   # must list only the sheet
   ```

   If anything else appears — a stray `_wat_run/`, a lockfile, a generated route tree — reset and re-stage just the sheet. A test-case commit that carries unrelated changes is a defect even when the test run succeeds.

   Never rewrite, reorder, or renumber existing rows. Never force-push. If the push is rejected because the branch moved, re-fetch, re-apply the append on top, and retry once; if it still fails, stop and report rather than forcing.
7. **Re-read the sheet from disk** after the commit, so the plan reflects the appended rows.
8. **Run the full suite** — every `Active` case including the new ones — through Phases 1–3 as normal.
9. Phase 3 reports `CASES_APPLIED` (the IDs committed) and the commit SHA in Notes, so the report states plainly that the sheet grew during this run.

**This is the one circumstance in which the plugin writes to the repository.** It requires an explicit `--apply-cases` invocation; without that flag the never-modify rule in `skills/post-test-report/SKILL.md` applies unchanged. The token must have write access to the PR branch — if the push fails with a permissions error, report that specifically rather than silently continuing to the test run, since the suite would then omit the approved cases.
