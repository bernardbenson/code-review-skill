# Adoption checklist

The layers from the source process that sit outside a single review run.
Use this when the user asks how to adopt the full process.

## Before the PR
- [ ] A short `CLAUDE.md` with runnable checks (build, test, lint), conventions, goals, non-goals.
- [ ] Hooks that run formatters and linters automatically.
- [ ] The agent shows check output instead of claiming success.
- [ ] A fresh-context review of the diff before opening the PR (this skill, `/code-review`, `/security-review`).
- [ ] A `REVIEW.md` written by code owners (see `REVIEW.template.md`).

## On every PR
- [ ] Required CI: lint, type check, tests on all supported platforms, build.
- [ ] PR template with **Testing**, **Risk**, and **How to revert** sections (see `pr-template.md`).
- [ ] An AI reviewer on every PR, plus a separate security review.
- [ ] All review threads must be resolved (fixed, or closed with a reason) before merge.
- [ ] Review effort matched to risk; review rounds capped (about 3).

## Approval
- [ ] Risk tiers written down (see `risk-tiers.md`). Start strict.
- [ ] `CODEOWNERS` and branch protection on sensitive paths.
- [ ] Dismiss stale approvals when new commits are pushed.
- [ ] Agent approvals do not count as human approvals.

## Merge and after
- [ ] A merge queue that re-runs checks on the latest main with the change applied.
- [ ] Feature flags that turn off without a deploy; internal users first.
- [ ] Staged releases (for example, `latest` channel before `stable`).
- [ ] One-step rollback.
- [ ] Incident reviews that ask which layer should have caught the issue.
- [ ] Past incident PRs turned into evals for the reviewer; keep reviewer changes only if the score improves.
- [ ] Track human time (approvals per PR, comments, wait time) with revert and incident rates as guardrails.
- [ ] Classify each reviewer comment by outcome: fixed, wrong, intentional, does not matter.
