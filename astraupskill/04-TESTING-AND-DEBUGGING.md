# Testing and debugging

## The runtime harness

`tests/runtime-refund-scope.test.mjs` loads the production `src/services/refundService.ts` through
`tests/helpers/load-typescript.mjs`. Relative imports load real source, so `src/lib/errors.ts` and
`src/lib/domain/paymentStateMachine.ts` are the production modules. It stubs five specifiers
(lines 10-26): `@prisma/client`, `../lib/db` (a `$transaction` that records `'transaction'` and calls
the callback with a frozen `tx`), and the payment, refund and event repositories. It also injects a
`crypto.randomUUID` that returns `'fixed-uuid'` (line 10), which is why the provider id is
predictable.

Every stub calls `effect(name, client)` (line 9), which asserts `client === tx` and appends to a
call ledger. The ledger is what the cases compare.

| Case | Lines | Fixture | Expected ledger |
| --- | --- | --- | --- |
| Full remaining refund | 29-33 | amount 100, refunded 60, refund 40 | transaction, payment, sum, create, REFUND_REQUESTED, refund-status, payment-status, REFUND_SUCCEEDED; result `paymentStatus: "REFUNDED"` |
| Partial refund | 34-37 | refunded 10, refund 40 | same minus `payment-status`; `paymentStatus: "SUCCEEDED"` |
| Policy refusals | 38-42 | refunded 80 (remaining 20) and status FAILED | stops after `sum`; 409 with `REFUND_AMOUNT_EXCEEDS_CAPTURE` / `PAYMENT_NOT_REFUNDABLE` |
| Missing payment | 43-45 | `missing: true` | stops after `payment`; 404 `PAYMENT_NOT_FOUND` |
| Event failure | 46-49 | first `createPaymentEvent` throws | stops after `REFUND_REQUESTED`; the same `Error` object reaches the caller |

The FAILED case still runs `sum` because the service computes the remaining amount before calling
`assertRefundAllowed` (lines 32-34), even though the status check alone would reject.

## What the ledger proves and what it does not

It proves call order, argument wiring, and that one client object is passed everywhere. It does not
prove that PostgreSQL rolls back the refund row in the event-failure case — the stub `$transaction`
has no rollback. "Create was called, then the event threw" is a partial workflow in the ledger, not
evidence about persisted rows.

## The other tests

`npm test` runs `node --experimental-strip-types ... --test tests/**/*.test.ts` (`package.json:10`),
which picks up four TypeScript files and **not** the `.mjs` runtime test:

- `tests/auth/authorization.test.ts:8-26` — SUPPORT lacks `reconciliation:run` -> `RouteError` 403.
- `tests/services/paymentStateMachine.test.ts:7-18` — FAILED -> SUCCEEDED is rejected with 409.
- `tests/services/paymentAccessPolicy.test.ts:6-20` — CUSTOMER list scope is `{ ownerUserId }`.
- `tests/validation/paymentSchemas.test.ts:6-17` — `amountMinor: 0` fails `createPaymentSchema`.

Whether the `tests/**` glob expands as intended depends on the shell running the script. These were
not executed for this documentation pass.

## Untested surfaces

- `src/lib/auth/guards.ts` — nothing calls `withRouteAuth`, so the fact that its 401/403 is thrown
  outside the handler's `try` (line 32) is invisible to the suite.
- `src/services/webhookService.ts` — no duplicate, rejection or rollback test.
- `src/services/paymentService.ts` — no replay, fingerprint-mismatch or P2002 race test.
- `src/services/reconciliationService.ts` — no test of any issue type.

## Debugging a refund that "should have worked"

1. 403/500 before the handler: check the user's role against `src/lib/permissions.ts:20-37`.
2. 400 `VALIDATION_ERROR`: the reason is under 5 characters or equals "test"
   (`src/lib/validation/paymentSchemas.ts:24-37`).
3. 409 `PAYMENT_NOT_REFUNDABLE`: the payment is PENDING, FAILED, or already REFUNDED.
4. 409 `REFUND_AMOUNT_EXCEEDS_CAPTURE`: compare the request with `amountMinor` minus the SUCCEEDED
   refunds shown on the payment detail (`paymentDetailInclude` loads refunds,
   `src/repositories/paymentRepository.ts:15-18`).
5. 500: read the server log — `jsonError` logs unknown errors with `console.error`
   (`src/lib/api/route-helpers.ts:43`).

## Exercise

**Goal:** turn a ledger into a prediction.
**Check:** for a payment of 100 with 0 refunded and a refund of 100, write the expected ledger and
`paymentStatus` before reading case 1 (answer: the full eight-entry ledger and `REFUNDED`, because
`0 + 100 === 100` at line 64).
