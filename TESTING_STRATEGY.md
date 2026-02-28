# Sweep Testing Strategy

## Required Test Ladder

Every PR (human or agent) must satisfy:

1. Unit tests
- Pure logic and invariant tests.
- Includes edge cases and invalid input paths.

2. Contract tests
- Request/response validation against `packages/shared/src/contracts/**`.
- Enum/state compatibility checks.

3. Integration tests
- DB behavior (state transitions, idempotency collisions, transaction safety).
- External side effects mocked and asserted.

4. Critical E2E tests
- Cleaner onboarding
- Customer search
- Hold -> payment intent -> booking confirm
- Cancellation/refund

## Mandatory Edge-Case Coverage

- Duplicate idempotency key with same payload returns cached result.
- Duplicate idempotency key with different payload is rejected.
- Invalid status transition is rejected.
- Pricing math rounding for decimal hours.
- Expired hold cannot be confirmed.
- Unauthorized and cross-user access attempts return 401/403.

## CI Minimum Gates

- `bun run test` in `backend`
- Prisma schema validation (`bun run db:generate`)
- Lint/typecheck (to be added as repo-wide gate)

## Failure Triage Rules

- Any flaky test blocks merge until stabilized.
- Production bug regressions must add a test before fix is merged.
- Every incident should produce at least one new automated test.
