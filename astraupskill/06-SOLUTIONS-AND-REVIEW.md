# Solutions and review

## Level 1

| # | Outcome | Payment status after |
| --- | --- | --- |
| 1 | allowed (remaining 40) | REFUNDED (60 + 40 = 100) |
| 2 | 409 `REFUND_AMOUNT_EXCEEDS_CAPTURE` (`paymentStateMachine.ts:60-62`) | SUCCEEDED |
| 3 | allowed | SUCCEEDED (10 + 40 = 50) |
| 4 | allowed | REFUNDED |
| 5 | 409 `PAYMENT_NOT_REFUNDABLE` (`paymentStateMachine.ts:55-57`) | PENDING |
| 6 | 409 `PAYMENT_NOT_REFUNDABLE` | REFUNDED |

## Level 2

`findPaymentById(input.paymentId, tx)` (line 28), `sumSuccessfulRefundsForPayment(payment.id, tx)`
(32), `createRefund({...}, tx)` (36-44), `createPaymentEvent({... REFUND_REQUESTED}, tx)` (46-59),
`transitionRefundStatus(refund.id, SUCCEEDED, providerRefundId, tx)` (62),
`transitionPaymentStatus({...}, tx)` (67-74, full refunds only), `createPaymentEvent({...
REFUND_SUCCEEDED}, tx)` (78-91).

## Level 3

(a) transaction, payment — then 404. (b) transaction, payment, sum — then 409. (c) transaction,
payment, sum, create, REFUND_REQUESTED — then the original error; no refund transition, no payment
transition, no second event. Against a real database the refund insert from (c) would roll back with
the transaction; the stub cannot show that.

## Level 4

Both transactions read 60 and both pass `40 <= 40`. Both insert and commit; the SUCCEEDED total is
140. A `SELECT id FROM "Payment" WHERE id = $1 FOR UPDATE` via `tx.$queryRaw` immediately after line
25 serializes the two: B blocks until A commits. B's `findPaymentById` then sees A's committed
status change to REFUNDED, so B fails with `PAYMENT_NOT_REFUNDABLE` (the status check runs before
the amount check). With a smaller pair — say two 20-unit refunds against a remaining 30 — B would
instead see a refunded total of 90 and fail with `REFUND_AMOUNT_EXCEEDS_CAPTURE`.

## Level 5

A good plan uses a disposable database, seeds a SUCCEEDED payment of 100 with a SUCCEEDED refund of
60, wraps `sumSuccessfulRefundsForPayment` to await a barrier both calls must reach, releases it, and
asserts the refunded sum. Today it observes 140; with a lock or serializable retry, 100.

## Level 6

`STATUS_MISMATCH`. The refund path sets `Payment.status = REFUNDED` and
`Payment.lastKnownProvider = REFUNDED` (`refundService.ts:67-74`, writing through
`src/repositories/paymentRepository.ts:105-124`) but never updates `ProviderPayment.status`, which
stays SUCCEEDED. Reconciliation maps SUCCEEDED -> SUCCEEDED (`reconciliationService.ts:58`), compares
with REFUNDED (line 60), and — because a SUCCEED-mode payment has a webhook delivery — records a
`STATUS_MISMATCH` (lines 61-81) on every run.

## Level 7

No. `withRouteAuth`'s returned function calls `requirePermission` at `guards.ts:32` before invoking
the handler; the handler's `try`/`catch` (route lines 10-22) has not started yet. The `RouteError`
rejects the route function itself and is handled by Next.js as an uncaught error, not by `jsonError`.

## Senior-review findings

Found by reading the code; none executed.

1. **Auth errors bypass `jsonError`.** `src/lib/auth/guards.ts:32` throws 401/403 outside every
   handler's `try`. All `withRouteAuth` routes (`src/app/api/payments/route.ts:7`,
   `src/app/api/admin/**`) are affected; the documented `{ error: { code: "FORBIDDEN" } }` body is
   not what callers receive. Fix: wrap lines 32-34 in `try`/`catch` and return `jsonError(error)`.
2. **Unauthenticated webhook.** `src/app/api/webhooks/provider/route.ts:4-14` accepts any JSON and
   passes it to `processProviderWebhook`, which casts it without validation
   (`src/services/webhookService.ts:29`). Anyone who learns a `providerPaymentId` can post
   `status: "SUCCEEDED"` and move a PENDING payment to SUCCEEDED. The route's own comment (line 8)
   acknowledges the missing signature check.
3. **Concurrent refunds over-refund.** See Level 4. `src/services/refundService.ts:28-34`.
4. **Webhook rejection mark is rolled back.** `webhookService.ts:123-126` marks the event REJECTED
   with `tx` and then rethrows, which rolls back the whole transaction — including the
   `createWebhookEvent` at lines 54-65 and the REJECTED mark itself. Failed webhooks leave no
   `WebhookEvent` row; only the outbound `WebhookDelivery` records the failure
   (`src/services/providerSimulatorService.ts:170-178`).
5. **Duplicate-check outside the transaction.** `webhookService.ts:30-37` reads the event before
   the transaction; two concurrent deliveries of one event both proceed, and the second
   `createWebhookEvent` hits the `providerEventId` unique constraint (`prisma/schema.prisma:216`) and
   surfaces as a 500.
6. **Reconciliation false positive on refunds.** See Level 6.
7. **Idempotency key not scoped to user.** `src/services/paymentService.ts:66-68` looks up
   `(route, key)` only, and the fingerprint covers just the body (line 67). A second user who reuses
   a key with the same body receives the first user's payment id in the replay.
8. **UI defeats idempotency.** `src/hooks/usePaymentActions.ts:32` generates a fresh
   `Idempotency-Key` on every mutation call, so a double-submitted form creates two payments.
9. **Broken import in the guard.** `src/lib/auth/guards.ts:10` imports `RouteError` from
   `../api/route-helpers`, but `src/lib/api/route-helpers.ts:10` only imports it and never
   re-exports it. TypeScript should reject this import during type-checking; if the code is bundled
   anyway, `RouteError` is `undefined` and `error instanceof RouteError` at `guards.ts:48` throws a
   `TypeError` whenever `requirePagePermission` sees a 403 — so a SUPPORT user opening
   `src/app/admin/reconciliation/page.tsx` gets an error instead of the intended redirect to `/`.
   Fix: import from `../errors`.
10. **Unbounded reconciliation.** `listPaymentsForReconciliation` (`src/repositories/paymentRepository.ts:126-134`)
   loads every payment with all webhook deliveries in one query.

## Review rubric for the refund patch

Accept: both reads use `tx`; not-found and policy checks precede `createRefund`; original errors
propagate; full vs partial branches unchanged; documentation says plainly that isolation is not
solved. Reject: any claim that moving reads into the callback prevents concurrent over-refunds, or
that the VM test proves rollback.
