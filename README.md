# layered-review

A Claude Code skill based on
[Solving code review at Anthropic](https://claude.dev/blog/solving-code-review-at-anthropic/).

It reviews a diff, branch, or PR in layers:

1. **Verification evidence** – runs build, lint, type check, and nearby tests, and reports the output.
2. **Risk tier** – classifies the change as Low, Medium, or High (0, 1, or 2 human approvals).
3. **Bug hunt** – parallel finder agents, each with a different lens. Correctness only, no style.
4. **Security pass** – a separate pass. Its findings can block merge.
5. **Verify by vote** – independent verifiers try to refute each finding. Only majority-confirmed findings survive.
6. **Tag and cap** – Important, Nit (max 3), or Pre-existing. Findings of the same bug class are grouped.
7. **Convergence** – round state, no repeated findings, diff-only re-review from round 3, and a "reconsider the approach" stop after round 3.
8. **Report** – starts with the count, says "Nothing needs fixing" when true, and lists what is left for the human.

## Install

Project-level (already active in this repo):

```
.claude/skills/layered-review/
```

User-level (all projects):

```sh
cp -r .claude/skills/layered-review ~/.claude/skills/
```

## Use

```
/layered-review                 # standard effort, current branch vs base
/layered-review quick
/layered-review deep 123        # PR #123
/layered-review standard 123 --comment
/layered-review --full          # ignore round state
```

| Effort | Finders | Verifiers per finding | Threshold |
|--------|---------|-----------------------|-----------|
| quick | inline | inline | file:line evidence |
| standard | 3 | 3 | 2/3 |
| deep | 5 | 5 | 3/5 |

Add a `REVIEW.md` to your repo to set the default effort, sensitive paths, paths to skip,
and the nit cap. See `.claude/skills/layered-review/references/REVIEW.template.md`.

## Files

- `SKILL.md` – the review procedure.
- `references/risk-tiers.md` – tier rules and approval rules.
- `references/REVIEW.template.md` – a starter `REVIEW.md` for code owners.
- `references/pr-template.md` – PR template with Testing, Risk, and How to revert.
- `references/adoption-checklist.md` – the layers outside a review run (CI, merge queue, flags, rollback, evals, metrics).
