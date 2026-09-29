# Worked change

Before, the service read `findPaymentById(input.paymentId)` and `sumSuccessfulRefundsForPayment(payment.id)` before `prisma.$transaction`. After, the transaction callback begins with `findPaymentById(input.paymentId, tx)`, checks not found, computes `sumSuccessfulRefundsForPayment(payment.id, tx)`, and calls `assertRefundAllowed` before `createRefund(..., tx)`. The resulting trace keeps allowance calculation with the mutation. The test records the exact client object for payment, sum, and create, then rejects a 30-unit refund when 80 of 100 is already successful. This is a bounded persistence-scope change, not a provider integration or full concurrency proof.

Compare the exact [snapshot](snapshots/src/services/refundService.ts) with the service. The repository signatures already accepted an optional client, so this correction changes call placement and arguments without changing their SQL definitions. Missing payments now fail inside the callback before aggregation. A legal full refund still creates the row, writes REFUND_REQUESTED, transitions refund status, transitions payment status and writes REFUND_SUCCEEDED. A partial refund omits the payment-status transition. The expanded tests check every collaborator's transaction handle, rather than only checking that a transaction callback exists.

```ts
return prisma.$transaction(async (tx) => {
  const payment = await findPaymentById(input.paymentId, tx);
  // Existing not-found guard remains here.
  const refundedAmount = await sumSuccessfulRefundsForPayment(payment.id, tx);
  // Existing allowance validation and writes follow.
});
```
