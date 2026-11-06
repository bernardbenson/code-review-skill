# REVIEW.md

<!-- Code owners write this file. The layered-review skill reads it from the
repo root and from every directory a diff touches. Keep it short. -->

## Default effort
standard   <!-- quick | standard | deep: choose by the cost of a missed bug -->

## What failure looks like here
<!-- The bugs that actually hurt this repo. Be concrete. -->
- Example: a cache key that omits the user ID leaks data between accounts.
- Example: a migration that is not backward compatible breaks the rolling deploy.

## Sensitive paths (always need a human; two-person where marked)
- `billing/**` (two-person)
- `auth/**` (two-person)
- `.github/workflows/**`

## Paths to skip
- `**/generated/**`
- `**/*.snap`
- `vendor/**`

## Rules for findings
- Every finding must cite file and line and give a concrete failure scenario.
- Max 3 nits per review.
- Do not comment on style. The formatter and linter own it.
- Start the summary with a count. Say "Nothing needs fixing" when that is true.

## Trusted authors (lighter path for Low tier)
- @owning-team
