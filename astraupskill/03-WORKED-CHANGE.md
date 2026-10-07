# Worked change

## Before (snapshot)

`astraupskill/snapshots/src/services/refundService.ts` read outside the transaction:

```ts
const payment = await findPaymentById(input.paymentId);          // default client
if (!payment) { throw new RouteError(404, "PAYMENT_NOT_FOUND", "Payment not found."); }
const refundedAmount = await sumSuccessfulRefundsForPayment(payment.id); // default client
const remainingRefundableAmount = payment.amountMinor - refundedAmount;
assertRefundAllowed(payment.status, input.amountMinor, remainingRefundableAmount);

return prisma.$transaction(async (tx) => {
  const refund = await createRefund({ ... }, tx);
  // events, status transitions ...
});
```

## After

`src/services/refundService.ts:25-34`:

```ts
return prisma.$transaction(async (tx) => {
  // Read the payment and successful refund total on the same transaction
  // client as the write. This keeps the allowance check with the mutation.
  const payment = await findPaymentById(input.paymentId, tx);
  if (!payment) {
    throw new RouteError(404, "PAYMENT_NOT_FOUND", "Payment not found.");
  }
  const refundedAmount = await sumSuccessfulRefundsForPayment(payment.id, tx);
  const remainingRefundableAmount = payment.amountMinor - refundedAmount;
  assertRefundAllowed(payment.status, input.amountMinor, remainingRefundableAmount);
  // createRefund(..., tx) and the rest are unchanged
```

`git diff --no-index` between the snapshot and the current file shows exactly that hunk: 11 lines
removed from before the transaction, 10 added inside it. Nothing after `createRefund` changed.

## Why it was a small change

Both repository functions already accepted an optional client
(`src/repositories/paymentRepository.ts:55`, `src/repositories/refundRepository.ts:28`). The fix
changes call placement and arguments, not SQL.

## Behavioural differences

| Situation | Before | After |
| --- | --- | --- |
| Missing payment | 404 thrown before any transaction | 404 thrown inside the callback; transaction rolls back (nothing was written) |
| Over-refund / non-SUCCEEDED | 409 before the transaction | 409 inside the callback |
| Legal refund | reads on one connection, writes on another | all on `tx` |
| Concurrent legal refunds | can over-refund | can still over-refund (see 02, section 2) |

The legal path still writes, in order: refund row, `REFUND_REQUESTED`, refund -> SUCCEEDED, payment
-> REFUNDED (full refunds only), `REFUND_SUCCEEDED` (lines 36-91).

## Review notes

- The comment at lines 26-27 is accurate and should stay; it explains why `tx` is passed.
- The `⚠️ SENIOR NOTE` at line 77 sits *after* the full-refund branch it describes.
- The `fullyRefunded` test at line 64 uses `===`. It cannot be `>` because the allowance check
  already prevents exceeding the amount in a single request.

## Exercise

**Goal:** confirm the change is exactly as described.
**Check:** run `git diff --no-index astraupskill/snapshots/src/services/refundService.ts src/services/refundService.ts`
from the repo root and confirm the only hunk starts at line 22 of the snapshot, and that no line
after `createRefund(` differs.
