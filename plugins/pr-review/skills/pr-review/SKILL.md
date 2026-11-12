---
name: pr-review
description: Senior-dev PR review — plain-language summary of what the PR does, real bugs / breaking changes / standards violations as JSON with criticality, and a final merge verdict. Use when the user says "/pr-review", "review this PR", or passes a PR number, branch, or URL. `/pr-review --comment #1 #3` drafts short inline comments for those findings as a pending (unsubmitted) review.
---

# PR review (senior dev, no noise)

Review the target like a senior engineer on this team who has to live with the
code afterwards. Be real: report only things that are actually wrong or risky.
Do NOT pad the review with style nits, "consider adding a comment", speculative
"might be nice" refactors, or anything you can't point to in the diff. An empty
bug list is a perfectly good outcome.

## 1. Figure out the target

First find the default branch (`gh repo view --json defaultBranchRef --jq
.defaultBranchRef.name`, usually `main`). Below it's written as `<base>`.

- Argument is a PR number or URL → `gh pr view <n>` + `gh pr diff <n>`.
- Argument is a branch → `git diff <base>...<branch>`.
- No argument → the current branch vs `<base>` (`git diff <base>...HEAD`), plus
  any uncommitted changes (`git diff HEAD`).

Read the full diff. For every changed file, open enough of the surrounding code
to understand the change in context — don't judge a hunk in isolation.

## 2. Load the standards

Before judging anything, read the repo's own rules so "goes against our
standards" means *this repo's* standards, not generic ones. Read whichever of
these exist:

- The root `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`, and any such file in
  the directories the diff touches.
- Standards or convention docs relevant to the change (API, language style,
  testing, git workflow, …).
- Any architecture decision records (ADRs) the change touches or contradicts.

If the repo has none of these, judge by the conventions the existing code
clearly follows.

## 3. What to look for

In priority order:

1. **Actual bugs** — wrong logic, off-by-one, None/null/empty handling, wrong
   status codes, race conditions, broken error paths, wrong query results.
2. **Breaking changes** — changed API response shapes / field names / status
   codes, removed endpoints, migrations that break existing data or can't run
   on a live DB, changed contracts that other clients or services depend on.
3. **Security** — missing auth or permission checks, data leaking across
   users / tenants / accounts, injection, unsafe input handling, auth errors
   that reveal whether an account exists, secrets in code.
4. **Standards violations that matter** — breaking a hard rule the repo's docs
   state (module boundaries, required feature flags, required tests,
   idempotency or other contracts, etc.).
5. **Missing tests** only when the repo's rules require them or the untested
   path is genuinely risky.

For each candidate finding, verify it before reporting: trace the code path
and confirm it really breaks. If you can't describe a concrete scenario where
it goes wrong, drop it.

## 4. Output format (exactly these three sections)

### 1. What this PR does

A short, plain-English explanation (3–6 sentences or a few bullets) that a
non-author could understand: what problem it solves, what changed, and what a
user/caller will notice. No jargon dumps, no file-by-file list.

### 2. Bugs found

**If the `ReportFindings` tool is available** (Claude desktop app, Code tab):
call it once with the findings, most critical first, so the app shows the
"Walk through in Changes" view and the "Apply fixes" button. ALSO print the
full JSON block (format below) under this heading, with the same findings in
the same order, so the ids match. Do both every time: the panel and the JSON.
With nothing found, call it with `[]` and print `[]`. Map each finding:

- `file` → repo-relative path; `line` → line on the new side of the diff
- `short_summary` → the title (≤60 chars)
- `summary` → `"<Criticality>: <description> Fix: <fix>"`, e.g.
  `"High: loadLive has no stale-response guard. Fix: drop responses older than the latest request."`
- `failure_scenario` → the impact as a concrete scenario (inputs/state → what breaks)
- `category` → the type (`bug`, `breaking-change`, `security`, `standards`, `missing-test`)
- `verdict` → `"CONFIRMED"` (you only report findings you traced)

A finding's id is its 1-based position in that list (used by `--comment`).

**JSON block** (always printed; it's the only output when `ReportFindings` isn't
available): a single JSON code block — an array, most critical first. Empty array `[]` if nothing real was found. Each
item:

```json
[
  {
    "id": 1,
    "title": "Short name of the problem",
    "criticality": "critical | high | medium | low",
    "type": "bug | breaking-change | security | standards | missing-test",
    "file": "path/to/file.py",
    "line": 42,
    "description": "Plain simple language: what is wrong and when it happens.",
    "impact": "What actually goes wrong for users / data / other clients.",
    "fix": "The simplest way to fix it."
  }
]
```

Criticality guide:
- **critical** — data loss/corruption, security hole or data leak, prod outage,
  breaks existing clients. Must fix before merge.
- **high** — real bug users will hit in normal use. Should fix before merge.
- **medium** — bug in an edge case, or a hard-rule standards violation.
- **low** — minor but real issue; fine to fix in a follow-up.

Keep `description`, `impact`, and `fix` short and in simple, everyday
language — write it the way you'd explain it to a teammate over chat.

### 3. Good to merge?

One clear verdict line — **Yes**, **Yes, after fixing #…**, or **No** —
followed by one or two sentences on why.

## 5. `--comment` mode — draft inline comments as a pending review

`/pr-review --comment #1 #2 #5` means: take findings 1, 2 and 5 from the
review you already reported in this conversation and put each one on the PR
as an inline comment, inside a **pending (unsubmitted) review**. The user
submits it themselves as "Request changes" — you never publish it.

- The `#n` ids are the `id`s from the "Bugs found" JSON you reported earlier,
  or the 1-based positions in the `ReportFindings` list if you used that.
  If there's no earlier review in this conversation, run the review first
  (sections 1–4), show it, and stop — don't guess which findings were meant.
- No ids given (`/pr-review --comment`) → ask which ones; don't comment on all.
- The target must be a PR (number/URL, or the current branch's PR via
  `gh pr view --json number`). No PR → say so and stop.

### Comment style

Short, human, plain. One to two sentences: what's wrong, then what to do.
No headings, no bullets, no "Impact:" / "Fix:" labels, no criticality tags, no
essays, no "Great work but…". Write it like a teammate typing a quick review.
Examples of the right tone:

- "This only checks that the date isn't in the future, so a past date gets
  around the expiry. Reject anything earlier than the creation time."
- "Don't overwrite `created_at`, keep it as server time. Put the client's time
  in a new field like `recorded_at`."
- "The retry path skips the lock, so two workers can process the same job.
  Take the lock before the retry too."

Use backticks for identifiers. Don't mention Claude, AI, or the finding id.

### Creating the pending review

1. Get the head commit and repo: `gh pr view <n> --json headRefOid,number`
   and `gh repo view --json nameWithOwner`.
2. Anchor each comment on a line that is **in the PR diff**, on the new side
   (`side: "RIGHT"`). If the finding's line isn't part of a hunk, pick the
   closest changed line in that file that the comment is really about. If the
   file has no relevant changed line at all, tell the user that finding
   couldn't be anchored instead of forcing it somewhere wrong.
3. Check for an existing pending review by the user (GitHub allows only one
   per user per PR):
   `gh api repos/{owner}/{repo}/pulls/{n}/reviews --jq '.[] | select(.state=="PENDING") | .id'`
   - **None** → create one with all comments in a single call, and **omit
     `event`** so it stays PENDING (never pass `REQUEST_CHANGES`, `COMMENT`
     or `APPROVE` — that publishes it). Leave the review body empty:
     ```bash
     gh api repos/{owner}/{repo}/pulls/{n}/reviews --method POST --input payload.json
     ```
     with `payload.json` (written to a temp/scratch directory):
     `{"commit_id": "<headRefOid>", "comments": [{"path": "...", "line": 20, "side": "RIGHT", "body": "..."}]}`
     For a multi-line anchor add `start_line` + `start_side: "RIGHT"`.
   - **Exists** → add each comment to it via GraphQL
     `addPullRequestReviewThread` (input: `pullRequestReviewId` = the review's
     `node_id`, `path`, `line`, `side: RIGHT`, `body`) so the user's own
     pending comments are kept.
4. Report back briefly: which findings were added (file:line + the comment
   text), any that couldn't be anchored, and a reminder that the review is
   pending — open the PR's "Files changed", hit **Submit review**, choose
   **Request changes**.
