---
name: layered-review
description: Layered code review modeled on Anthropic's Claude Code team process. Runs pre-PR verification, parallel bug-hunting agents, a separate security pass, majority-vote verification of every finding, Important/Nit/Pre-existing tagging, risk-tier classification (Low/Medium/High → 0/1/2 human approvals), and round capping so reviews converge. Use when the user asks to review a diff, branch, or PR "the Anthropic way", asks for a layered/multi-agent review, asks what risk tier or how many approvals a change needs, or invokes /layered-review. Arguments - [quick|standard|deep] [target - PR number, branch, or path; default is the working-tree diff against the base branch] [--comment to post inline PR comments] [--full to force a full re-review].
---

# Layered review

This skill applies the review process from "Solving code review at Anthropic"
(claude.dev/blog/solving-code-review-at-anthropic). The goal is the **minimum human
review time that does not increase reverts or incidents**. The skill does the
machine layers. It tells the human exactly what is left for them.

Follow the phases in order. Do not skip a phase unless this file says so.

## Arguments

- **Effort**: `quick`, `standard` (default), or `deep`. Pick from the cost of a
  missed bug. If the repo's `REVIEW.md` sets a default effort, use it unless the
  user gave one.
- **Target**: a PR number, a branch name, or a path. Default: the diff of the
  current branch plus uncommitted changes against the base branch (`main`, else
  `master`, else the remote default branch).
- `--comment`: post findings as inline PR comments with `gh` (PR targets only).
  Confirm with the user before posting the first time in a session.
- `--full`: ignore round state and review the whole diff again.

| Effort   | Finder agents | Verifiers per finding | Keep a finding when | Security pass |
|----------|---------------|-----------------------|---------------------|---------------|
| quick    | 0 (you review inline) | 0 (you re-check each finding against the code) | you can cite file:line evidence | inline checklist |
| standard | 3 | 3 | ≥ 2 of 3 confirm | 1 agent |
| deep     | 5 | 5 | ≥ 3 of 5 confirm | 1 agent + inline checklist |

## Phase 0 – Gather context

Collect all of this before any review work:

1. **Diff and intent.**
   - PR: `gh pr view <n> --json title,body,baseRefName,headRefOid,files` and `gh pr diff <n>`.
   - Branch/local: `git diff <base>...HEAD` plus `git diff HEAD`. Use the last
     commit messages as the stated intent.
   - If the change claims to fix a bug, note that. Finders must check that the
     fix addresses the **root cause**, not only the symptom.
2. **Repo guidance.** Read `CLAUDE.md` and `REVIEW.md` at the repo root and in
   each directory that the diff touches. `REVIEW.md` rules override the defaults
   in this file (what failure looks like, nit cap, paths to skip, sensitive paths,
   default effort). If there is no `REVIEW.md`, say so in the report and offer
   the template in `references/REVIEW.template.md`.
3. **Round state.** Read `$(git rev-parse --git-dir)/layered-review/<branch>.json`
   if it exists (schema below). It tells you the round number, the last reviewed
   commit, and the findings already posted.
4. **Size.** Count changed lines. Note that large diffs (>1,000 lines) usually
   have findings and small ones usually come back clean. Do not invent findings
   to fill a report.

Drop files that `REVIEW.md` says to skip, plus generated files, lockfiles, and
vendored code, unless the user asks for them.

## Phase 1 – Verification evidence (pre-PR layer)

Evidence beats claims. Find the project's runnable checks in `CLAUDE.md`,
`package.json`, `Makefile`, `pyproject.toml`, CI workflow files, and similar.
Run the cheap ones that apply to the changed files: build, type check, lint,
and the tests nearest to the change. Time-box this phase (about 5 minutes for
`standard`; skip long suites in `quick`).

Record for the report: each command, pass/fail, and the relevant output lines.
If a check cannot run, say why. Never report a check as passed without output.

If checks fail, keep going with the review, but list the failures first in
the report. They block merge.

## Phase 2 – Risk tier

Classify the **change**, not the author. Use `references/risk-tiers.md`.

- **Low** – docs, copy, internal-only features, additive changes behind a flag.
  Needs 0 human approvals once automated checks and security review are green.
- **Medium** – ordinary product changes, and sensitive areas such as caching,
  auto-update, prompts, and config defaults. Needs 1 human approval.
- **High** – billing, auth, data migrations, permissions, crypto, deletion
  paths, and any path the owners list as two-person. Needs 2 human approvals.

Overrides that always require a human, whatever the tier:
- Any CI/CD workflow file (`.github/workflows/**`, `.gitlab-ci.yml`, `Jenkinsfile`, and similar).
- Any path listed as sensitive in `REVIEW.md` or `CODEOWNERS`.
- Any change that this review cannot vouch for (for example, a binary blob or an
  unreviewable generated diff).

State the tier, the reason in one sentence, and the approvals required. Remind
the user: an agent's approval never counts as a human approval, and the person
who prompted an agent-written PR is a required reviewer.

## Phase 3 – Bug hunt (parallel finders)

Correctness only. **Do not report** style, formatting, naming, missing
comments, compiler/linter diagnostics (Phase 1 covers those), or cleanup
suggestions. Those filters cut noise.

For `standard` and `deep`, launch the finder agents **in one message** with the
Agent tool (`subagent_type: general-purpose`) so they run in parallel. Give each
one a different lens. Use the first N of this list:

1. **Logic and edge cases** – off-by-one, null/empty, wrong conditions, unhandled errors, wrong return values.
2. **Intent and root cause** – does the diff do what the PR says? For bug fixes, is the root cause fixed, and are other instances of the same bug class still present?
3. **Cross-file contracts** – callers and callees outside the diff, changed signatures, API/schema compatibility, config and migration ordering.
4. **State and concurrency** – races, caching and invalidation, retries, idempotency, resource leaks, ordering.
5. **Failure and rollback** – behavior under partial failure, flags off/on, upgrade and downgrade paths, can the change be reverted safely.

Each finder prompt must contain: the diff (or the path to a file that holds it),
the stated intent, the relevant `CLAUDE.md`/`REVIEW.md` text, its lens, the
list of already-posted findings (so it does not repeat them), and the output
contract below. Tell each finder to read surrounding code in the repo, not only
the diff.

**Finder output contract** (one JSON array):

```json
[{"file": "src/x.ts", "line": 42, "title": "≤ 80 chars",
  "failure_scenario": "concrete input/state → wrong output/crash",
  "evidence": "the exact code lines and why they fail",
  "bug_class": "short slug, e.g. unchecked-null-from-cache",
  "introduced_by_diff": true}]
```

A finding without file:line evidence and a concrete failure scenario is
invalid. Discard it.

For `quick`, do this phase yourself with the same lenses and contract.

## Phase 4 – Security pass (separate, can block)

Run security as its own pass, separate from the bug hunt. For `standard` and
`deep`, launch one agent with the security lens in the same message as the
finders. Check: injection (SQL, shell, template, path), authn/authz gaps,
secrets in code or logs, unsafe deserialization, SSRF, unsafe file handling,
dependency changes, permission widening in CI, and data exposure. Use the same
output contract and add `"security": true`.

Open security findings block merge, the same as Important bug findings.

## Phase 5 – Verify by vote

Merge all candidate findings. Deduplicate findings that share file, nearby line,
and root cause.

For `standard` and `deep`, start the verifiers for **all** findings in one
message. Each verifier gets one finding and the code. It must **not** get the
finder's reasoning beyond the claim itself, so its read stays independent. Ask
it to try to refute the finding and return:

```json
{"verdict": "confirmed" | "refuted" | "unsure", "reason": "one or two sentences",
 "introduced_by_diff": true | false}
```

Keep a finding only if confirmed votes reach the threshold in the effort table.
Use the verifiers' majority on `introduced_by_diff`. If verifier count would
exceed about 40 agents, verify the top findings by severity and drop the rest,
and say so.

For `quick`, re-read the code for each finding yourself and drop any you
cannot defend.

## Phase 6 – Tag, group, and cap

Give each surviving finding exactly one tag:

- **Important** – a real bug or security issue that this diff introduces. Must be fixed or resolved with a reason before merge.
- **Nit** – correct but minor (a small edge case with low impact). Cap at 3 per review unless `REVIEW.md` says otherwise. Drop the rest and say how many you dropped.
- **Pre-existing** – a real bug that the diff did not introduce. Do not blame the author. Does not block merge. Suggest a separate issue.

When several findings share a `bug_class`, report them as one group. Name the
class and list every instance, so that the author fixes all of them at once.

## Phase 7 – Rounds and convergence

Write the round state after each review:

```json
{"branch": "feat/x", "round": 2, "last_reviewed_sha": "abc123",
 "findings": [{"fingerprint": "src/x.ts:unchecked-null-from-cache", "tag": "Important",
               "status": "open" | "fixed" | "resolved", "reason": ""}]}
```

Store it at `$(git rev-parse --git-dir)/layered-review/<branch>.json` so that
it is never committed.

Rules:
- **Never repeat** a finding whose fingerprint is already in state. If it is
  still present, mark it "still open" in one line instead.
- Mark a finding **fixed** when the new code no longer has the problem. It clears
  without human action.
- From **round 3** on (unless `--full`), review only `git diff <last_reviewed_sha>..HEAD`.
- After **round 3** with Important findings still open, stop hunting. Tell the
  author to reconsider the approach and say what the remaining findings have in
  common.
- If most findings are Pre-existing, say so. Further rounds will mostly surface
  old bugs. Recommend a separate sweep of existing code instead.

## Phase 8 – Report

Start with the count. If nothing needs fixing, say so plainly in the first line.

```
## Layered review — <target> (round <n>, <effort>)

**<k> Important, <m> Nit, <p> Pre-existing.** <or: "Nothing needs fixing.">
Risk tier: <Low|Medium|High> — <reason>. Human approvals required: <0|1|2>.
Merge blocked: <yes/no> — <blocking items: failed checks, open Important or security findings>

### Verification evidence
- `<command>` — pass/fail — <key output line>

### Important
1. **<title>** — `file:line` <(security)>
   Failure: <scenario>
   Evidence: <code and reason>   Votes: <c>/<n>

### Nit
...

### Pre-existing (not introduced by this PR)
...

### For the human reviewer
- <what only a person can judge: direction and scope, architecture others will
  copy, sensitive paths, reading the evidence above>
- How to revert: <one line, or "not stated — ask the author">
```

Keep each finding to 2–4 lines. Do not pad the report.

With `--comment` on a PR target, post each Important and Nit finding as an
inline review comment, and prefix it with its tag (`[Important]`, `[Nit]`,
`[Pre-existing]`). Use `gh api repos/{owner}/{repo}/pulls/<n>/comments` with
`commit_id` set to the head SHA. Post the summary as one PR comment. Never
approve the PR. An agent approval does not count.

## What this skill does not do

These layers exist in the source process but sit outside a review run. Mention
them in the report only when relevant, with a pointer to `references/adoption-checklist.md`:
blocking CI and branch protection, merge queues, feature flags and staged
releases, one-step rollback, incident-driven evals, and metrics (human
approvals per PR, wait time, revert and incident rates).
