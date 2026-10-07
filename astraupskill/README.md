# Orientation

This w22 lesson studies refund safety in an idempotent payments lab (package
`idempotent-payments-lab`, `package.json:2`): a Next.js 15 App Router app with Prisma on PostgreSQL
(`prisma/schema.prisma:5-8`), a simulated payment provider, webhook processing, refunds, and
reconciliation.

## The change this lesson is about

`createRefundForPayment` used to load the payment and the successful-refund total with the default
Prisma client **before** opening its transaction, then write the refund inside the transaction. The
staged fix moves both reads into the transaction callback and passes `tx` to the repository
functions (`src/services/refundService.ts:25-34`). The allowance check now sits next to the write it
guards, and every read and write in the workflow uses one client.

What the fix does **not** do: it does not make concurrent refunds safe. Prisma interactive
transactions run at the database's default isolation level (READ COMMITTED on PostgreSQL), so two
simultaneous refund requests can still both read the same refunded total and both commit. No row
lock, serializable retry, or idempotency key was added for refunds.

Files:

- Service: [refundService.ts](../src/services/refundService.ts) (lines 17-101)
- Pre-fix copy: [snapshots/src/services/refundService.ts](snapshots/src/services/refundService.ts)
- Runtime test: [runtime-refund-scope.test.mjs](../tests/runtime-refund-scope.test.mjs), five cases,
  loading the real service through `tests/helpers/load-typescript.mjs`

## How to use this pack

| Chapter | What you get |
| --- | --- |
| [01-CODEBASE-MAP](01-CODEBASE-MAP.md) | The refund path hop by hop with file:line, plus a map of the rest of the app |
| [02-CONCEPTS](02-CONCEPTS.md) | Transaction client vs isolation vs idempotency; amount identity; the provider gap |
| [03-WORKED-CHANGE](03-WORKED-CHANGE.md) | Before/after code and why the repository signatures made the fix small |
| [04-TESTING-AND-DEBUGGING](04-TESTING-AND-DEBUGGING.md) | What the five cases assert, the other test files, and what nothing tests |
| [05-PRACTICE](05-PRACTICE.md) | Graded exercises with Goal / Check |
| [06-SOLUTIONS-AND-REVIEW](06-SOLUTIONS-AND-REVIEW.md) | Answers, review rubric, and senior-review findings across the app |
| [07-TRACE-LAB](07-TRACE-LAB.md) | Tables to fill from the code |
| [VERIFICATION](VERIFICATION.md) | What was executed, when, and what was not |

## Headline findings outside the patch

While preparing this pack, reading the code turned up issues the refund fix does not touch (details
and line numbers in 06):

- `withRouteAuth` throws its 401/403 `RouteError` **outside** every handler's `try`/`catch`
  (`src/lib/auth/guards.ts:32`), so those errors are not converted by `jsonError`.
- The provider webhook endpoint has no authentication or signature check
  (`src/app/api/webhooks/provider/route.ts:4-14`), so any caller can move a payment's status.
- A fully refunded payment is flagged as a reconciliation mismatch on every run, because refunds
  never update the provider mirror (`src/services/refundService.ts:66-75` vs
  `src/services/reconciliationService.ts:58-60`).
