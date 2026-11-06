# Risk tiers

Classify the change by what it touches and what a mistake would cost. Do not
classify by who wrote it. Trusted authors (usually the owning team) may get a
lighter path only if the repo's `REVIEW.md` says so.

| Tier | Typical changes | Human approvals |
|------|-----------------|-----------------|
| Low | Docs, comments, copy/strings, tests only, internal-only tools, additive code behind a flag that defaults off | 0, once automated checks and security review are green |
| Medium | Ordinary product logic, refactors with behavior risk, dependency bumps, caching, auto-update, prompts/system prompts, config defaults, public API additions | 1 |
| High | Billing and payments, authn/authz, permissions, secrets handling, crypto, data migrations and deletion, public API removals or breaking changes, release/update infrastructure, owner-listed two-person paths | 2 |

## Always require a human

Whatever the tier:

- CI/CD workflow files and build pipelines.
- Paths listed as sensitive in `REVIEW.md` or covered by `CODEOWNERS` rules for a sensitive team.
- Changes that the review cannot vouch for: binaries, very large generated diffs, obfuscated code.

## Tie-breaking

- If a change spans tiers, use the highest tier that any file reaches.
- If unsure between two tiers, choose the higher one and say why.
- A flag only lowers the tier if the flag defaults off and turning it off needs no deploy.

## Approval rules

- An agent's approval never counts as a human approval.
- For an agent-written PR, the person who prompted the agent is a required reviewer.
- Approvals cover one commit. New commits that the automated layers cannot vouch for need re-approval.
