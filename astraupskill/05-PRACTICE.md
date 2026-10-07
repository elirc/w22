# Practice

Answers are in 06. Keep `src/services/refundService.ts` and
`src/lib/domain/paymentStateMachine.ts` open.

## Level 1 — arithmetic and branches

**Goal:** predict the outcome of single requests.
Payment amount 100. For each case give: allowed or code, payment status afterwards.

1. refunded 60, request 40
2. refunded 60, request 41
3. refunded 10, request 40
4. refunded 0, request 100
5. payment status PENDING, refunded 0, request 10
6. payment status REFUNDED, request 1

**Check:** six rows; each refusal names its error code and the line that throws it.

## Level 2 — client tracing

**Goal:** prove every database operation shares one client.
List every repository call in `createRefundForPayment` with the argument that carries the client.
**Check:** seven operations inside the transaction (two reads, one create, one refund transition,
up to one payment transition, two events), each passing `tx` as its last argument.

## Level 3 — failure boundaries

**Goal:** know where each failure stops the workflow.
For (a) missing payment, (b) FAILED payment, (c) the `REFUND_REQUESTED` event write throwing,
say which calls ran and which did not.
**Check:** compare with the ledgers in `tests/runtime-refund-scope.test.mjs:40, 44, 48`.

## Level 4 — the concurrency gap

**Goal:** explain what the transaction does not protect.
Two OPS users each refund 40 on a 100 payment with 60 refunded, at the same moment.
**Check:** a two-column timeline showing both aggregates returning 60 and both commits succeeding;
then pick one fix from 02, section 2, and show where in `refundService.ts` it goes.

## Level 5 — integration test design

**Goal:** design a test that would prove the concurrency claim against PostgreSQL.
**Check:** your plan seeds one payment, opens two `createRefundForPayment` calls with a barrier
after the aggregate (for example a wrapped repository that awaits a shared promise), asserts the
total of SUCCEEDED refunds afterwards, and states that today it would observe 140.

## Level 6 — review beyond the patch

**Goal:** find the reconciliation false positive.
Fully refund a SUCCEED-mode payment, then run reconciliation. Using
`src/services/refundService.ts:66-75` and `src/services/reconciliationService.ts:58-82`, predict
the issue (if any) recorded for that payment.
**Check:** name the issue type and explain which table the refund path forgot to update.

## Level 7 — auth wrapper

**Goal:** apply "where is the try?" to the guard.
Read `src/lib/auth/guards.ts:20-36` and `src/app/api/payments/[paymentId]/refund/route.ts:9-23`.
**Check:** say whether a SUPPORT user's refund request reaches `jsonError`, and why.

Score two points each for arithmetic, client identity, stopping point, isolation limits, and
separating the simulated provider id from a real refund.
