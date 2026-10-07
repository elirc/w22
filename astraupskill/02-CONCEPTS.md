# Concepts

## 1. Refund identity is more than an amount

Whether a refund of N is allowed depends on four facts: the payment id, the payment's current
status, the original amount (`Payment.amountMinor`, `prisma/schema.prisma:141`), and the sum of
refunds already SUCCEEDED (`src/repositories/refundRepository.ts:28-40`). The remaining refundable
amount is `amountMinor - refunded` (`src/services/refundService.ts:33`). The parameter name in
`assertRefundAllowed` is misleading — it is called `capturedAmountMinor`
(`src/lib/domain/paymentStateMachine.ts:53`) but the service passes the *remaining* amount.

The state machine checks status first, then amount (lines 55-62). A FAILED payment therefore gets
`PAYMENT_NOT_REFUNDABLE` even if the amount is also too large.

## 2. A transaction client is a scope, not a lock

Passing `tx` means every read and write runs on the same connection inside one `BEGIN ... COMMIT`.
If a later write fails, earlier writes in the callback roll back. That is what the fix buys.

It does not serialize concurrent transactions. Under READ COMMITTED, transaction A and transaction
B can both run the aggregate at line 32, both see `refunded = 60` on a 100 payment, both pass a
40-unit check, and both commit — 140 refunded against a 100 payment. Three standard fixes:

1. Lock the payment row first (`SELECT ... FOR UPDATE` through `$queryRaw`) so B waits for A.
2. Run the transaction at `Serializable` isolation and retry on serialization failure.
3. Store a running `refundedMinor` on `Payment` and update it with a guarded atomic statement
   (`UPDATE ... SET refunded = refunded + n WHERE refunded + n <= amount`), treating zero rows as 409.

## 3. Pending vs succeeded

The aggregate counts only SUCCEEDED refunds. A PENDING refund does not reserve money. In this code
the window is tiny, because the refund is created PENDING (`refundRepository.ts:20-25`) and flipped
to SUCCEEDED in the same transaction (`refundService.ts:62`). In a real integration where the
provider confirms asynchronously, a PENDING refund would sit outside the sum for minutes, and two
operators could each refund the full balance.

## 4. Idempotency is a different problem

A transaction makes one request atomic; idempotency makes a *repeated* request harmless. The refund
endpoint has no idempotency key, so a double-submitted refund form creates two refunds if the
balance allows. Payment creation does have one (`src/services/paymentService.ts:62-84`), keyed on
`(route, key)` with a request fingerprint (`prisma/schema.prisma:135`). Note that the key is not
scoped to the user: two users sending the same key and identical body share one record.

## 5. The simulated provider

`providerRefundId` is a locally generated `prov_ref_<uuid>` (`refundService.ts:61`) and the refund
is marked SUCCEEDED without any external call. Moving reads into the transaction does not solve the
real coordination problem — a provider refund that succeeds after the database transaction rolls
back, or vice versa. That needs an outbox or a two-phase "request, then confirm via webhook" design.

## 6. Errors as contract

The service throws `RouteError` (`src/lib/errors.ts:7-16`) for domain refusals and lets everything
else propagate. `jsonError` (`src/lib/api/route-helpers.ts:16-54`) turns `RouteError` into
`{ error: { code, message, details } }` with its status, `ZodError` into 400, and anything else into
a logged 500. The service catches nothing, so infrastructure failures stay distinguishable from
domain refusals.

## Exercise

**Goal:** separate atomicity from isolation.
**Check:** write two timelines for two concurrent 40-unit refunds on a 100 payment with 60 already
refunded — one with the current code, one with a `FOR UPDATE` lock on the payment row before line
32. State the final refunded total in each (140 vs 100) and which error the second request receives
under the lock (`PAYMENT_NOT_REFUNDABLE`: the first request made the payment REFUNDED, and status is
checked before amount).
