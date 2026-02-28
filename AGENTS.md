# AGENTS.md

If you are reading this file, also read `CLAUDE.md` in this same directory before taking actions.

If both files contain instructions, follow both and use the stricter rule when they conflict.

## Non-Negotiable Testing Gate

- Every new function, behavior change, and bug fix must include corresponding automated tests in the same PR.
- Before declaring work complete, run the full existing automated test suite for each affected repo (not just tests added or touched in the change).
- Do not mark work as done if any pre-existing test fails.

## Branch and PR Workflow (Required)

- For every code change, create a new branch from `main` and open a new PR to merge into `main` unless explicitly instructed otherwise.
- Never commit or push directly to `main` unless the user explicitly asks for that exception.
- Before merging a PR, run code review using a fresh reviewer sub-agent.
- If review finds any blocking issue (bugs, regressions, failing tests, security risks, or unclear logic), fix it and run review again with a new fresh reviewer sub-agent.
- Repeat the fix and fresh-review cycle until the latest sub-agent PR review is clean, then merge.

## Production Guardrails (Required)

- Require pull requests to `main` with required status checks and required approvals before merge.
- Require Code Owner approval for critical paths (`backend`, `auth`, `payments`, `infra`, and workflow/security configs).
- Treat AI-generated output as draft input only; human review and accountability are mandatory before merge.
- CI on every PR must run compile/build validation, the full automated test suite, and static analysis.
- Security gates are mandatory: CodeQL/code scanning and dependency review must pass.
- Release artifacts must have supply-chain provenance attestation (SLSA-aligned build provenance).
- For AI-related changes, run evals continuously on every change; fail PRs when evals are required but missing.
- Follow secure SDLC baseline controls: NIST SSDF (SP 800-218) and OWASP ASVS 5.0.0.

