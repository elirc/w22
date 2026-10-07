# Testing Rationale

## `support role cannot run reconciliation` in `tests/auth/authorization.test.ts`
This is worth a test because reconciliation is a privileged operational action that can expose system-wide financial state. If a support user can trigger it accidentally, your RBAC model is not trustworthy.

The test calls `requirePermission()` directly. It does not go through `withRouteAuth()` in `src/lib/auth/guards.ts`, which is why it cannot notice that the guard throws its `RouteError` at line 32 outside the handler's `try`/`catch`. A more thorough production suite would hit `src/app/api/admin/reconciliation/run/route.ts` through an HTTP test harness, verify the structured `403` body, and cover unauthenticated requests separately — that test would currently fail.

## `cannot move a failed payment back to succeeded` in `tests/services/paymentStateMachine.test.ts`
This is worth a test because terminal-state regressions are exactly the kind of subtle correctness bug that creates false revenue or refund eligibility.

A more thorough production suite would add webhook-driven transition cases through `src/services/webhookService.ts`, including duplicate-event handling, the rejected-event path (whose `REJECTED` mark is currently rolled back with the transaction), and legal `PENDING -> FAILED` / `PENDING -> SUCCEEDED` transitions.

## `customer payment queries are scoped to their own user id` in `tests/services/paymentAccessPolicy.test.ts`
This is worth a test because list endpoints are one of the easiest places to leak multi-record data accidentally. The ownership filter in `buildPaymentListScope()` is small but high-risk.

A more thorough production suite would drive `src/services/paymentQueryService.ts` against a test database and prove that customer sessions never receive rows for other owners. Note that the only HTTP list route today, `src/app/api/admin/payments/route.ts`, requires `PAYMENT_READ_ANY`, so a CUSTOMER never reaches the scoped branch through the API.

## `createPaymentSchema rejects non-positive payment amounts` in `tests/validation/paymentSchemas.test.ts`
This is worth a test because amount validation belongs at the API boundary. Letting impossible money values into the service layer makes failures harder to reason about and easier to mishandle.

A more thorough production suite would add boundary cases for currency normalization, refund-reason refinement, and malformed payload shapes against the real `POST /api/payments` route.

This test imports `zod`, so it needs installed dependencies; the other three tests import only local source and `node:` modules.

## `tests/runtime-refund-scope.test.mjs` (five cases)
Added with the refund-scope fix. It loads the real `src/services/refundService.ts`, state machine and `RouteError` through `tests/helpers/load-typescript.mjs`, stubs Prisma and the repositories, and asserts that both refund reads and every write receive the same transaction client, plus full/partial outcomes, policy refusals, missing payment, and event-failure propagation. It is not matched by the `npm test` glob in `package.json:10` (`tests/**/*.test.ts`); run it with `node --experimental-vm-modules --test tests/runtime-refund-scope.test.mjs`. It does not prove database rollback or concurrency behavior.

## Not tested anywhere
- Idempotent replay, payload-mismatch 409, and the P2002 collision branch in `src/services/paymentService.ts:168-188`.
- Every reconciliation issue type in `src/services/reconciliationService.ts`.
- Concurrent refunds (`src/services/refundService.ts:25-34`) — the main remaining risk after the fix.
