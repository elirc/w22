# Codebase map

## 1. The refund path, hop by hop

| # | Hop | File:line | What happens |
| --- | --- | --- | --- |
| 1 | Form + hook | `src/hooks/usePaymentActions.ts:42-58` | `useRefundPayment(paymentId)` POSTs `{ amountMinor, reason }`; on success invalidates the list and detail queries. |
| 2 | Guard | `src/lib/auth/guards.ts:28-35` | Resolves the session from the cookie, then `requirePermission(..., PAYMENT_REFUND)` (line 32). Only OPS and ADMIN hold it (`src/lib/permissions.ts:23-36`). |
| 3 | Handler | `src/app/api/payments/[paymentId]/refund/route.ts:9-23` | Parses the body with `refundPaymentSchema` (line 11), calls the service, returns `202`. Its `try`/`catch` maps errors with `jsonError`. |
| 4 | Schema | `src/lib/validation/paymentSchemas.ts:24-37` | Positive integer up to 1,000,000; reason 5-200 chars and not literally "test". |
| 5 | Transaction | `src/services/refundService.ts:25` | `prisma.$transaction(async (tx) => ...)` opens before any read. |
| 6 | Payment read | `refundService.ts:28-31` -> `src/repositories/paymentRepository.ts:55-60` | `findUnique` with the full `paymentDetailInclude` (owner, refunds, events, webhooks — lines 11-29). Missing -> 404 `PAYMENT_NOT_FOUND`. |
| 7 | Refunded total | `refundService.ts:32` -> `src/repositories/refundRepository.ts:28-40` | `aggregate` of `amountMinor` over refunds with status SUCCEEDED only; `null` sum becomes 0. |
| 8 | Allowance | `refundService.ts:33-34` -> `src/lib/domain/paymentStateMachine.ts:50-63` | Status must be SUCCEEDED (409 `PAYMENT_NOT_REFUNDABLE`), then amount must not exceed `amountMinor - refunded` (409 `REFUND_AMOUNT_EXCEEDS_CAPTURE`). |
| 9 | Create | `refundService.ts:36-44` -> `refundRepository.ts:11-26` | Inserts a PENDING refund. |
| 10 | Event | `refundService.ts:46-59` | `REFUND_REQUESTED` payment event. |
| 11 | "Provider" | `refundService.ts:61-62` | Generates `prov_ref_<uuid>` locally and marks the refund SUCCEEDED. No provider call happens. |
| 12 | Full refund | `refundService.ts:64-75` | If `refunded + amount === payment.amountMinor`, payment -> REFUNDED with `lastKnownProvider: "REFUNDED"`. |
| 13 | Event | `refundService.ts:78-91` | `REFUND_SUCCEEDED` with `fullyRefunded`. |
| 14 | Result | `refundService.ts:93-99` | `{ refundId, paymentId, refundStatus, paymentStatus, providerRefundId }`. |

Every repository function used here takes an optional `db: DatabaseClient = prisma` as its last
argument (for example `refundRepository.ts:28`). That default is why the original bug was silent:
omitting `tx` still compiles and still works, it just runs on a different connection.

## 2. The rest of the system

- **Auth.** Database-backed sessions: login creates a random 24-byte token row
  (`src/services/authService.ts:28-34`); every request looks it up and checks `expiresAt`
  (`src/lib/auth/session.ts:23-44`). Passwords use scrypt with a random salt
  (`src/lib/auth/password.ts:11-26`).
- **RBAC.** One table, `rolePermissions` (`src/lib/permissions.ts:20-37`). CUSTOMER: create + read
  own. SUPPORT: read any. OPS and ADMIN: everything.
- **Payment creation.** `createPaymentWithIdempotency` (`src/services/paymentService.ts:55-192`)
  requires an `Idempotency-Key` header, fingerprints the body with a key-sorted SHA-256
  (`src/lib/idempotency/requestFingerprint.ts:9-29`), replays completed responses, and creates the
  payment, event, provider mirror and optional webhook delivery in one transaction (lines 87-167).
- **Provider simulator.** `resolveProviderSimulation` (`src/services/providerSimulatorService.ts:24-52`)
  maps SUCCEED / FAIL / PARTIAL_SUCCESS / NETWORK_UNKNOWN to a provider status and whether a webhook
  is queued (PARTIAL_SUCCESS queues none).
- **Webhooks.** `processProviderWebhook` (`src/services/webhookService.ts:28-129`) dedupes on
  `providerEventId`, updates the provider mirror, and applies a legal local transition.
- **Reconciliation.** `runReconciliation` (`src/services/reconciliationService.ts:23-144`) compares
  each payment with its provider mirror and records MISSING_PROVIDER_PAYMENT, STATUS_MISMATCH,
  MISSING_WEBHOOK_DELIVERY, or MISSING_LOCAL_PAYMENT issues.

## 3. Contrast: the payment-creation path, which already did this right

Payment creation is the model the refund fix copies. Every write in it takes `tx`:

| # | File:line | Step |
| --- | --- | --- |
| 1 | `src/services/paymentService.ts:62-64` | No `Idempotency-Key` header -> 400 `MISSING_IDEMPOTENCY_KEY`. |
| 2 | `paymentService.ts:66-68` | Fingerprint the parsed body; look up `(route, key)` with the default client (outside any transaction). |
| 3 | `paymentService.ts:70-84` | Existing record: different hash -> 409; COMPLETED -> replay with `replayed: true` (`buildCompletedReplay`, lines 42-53); otherwise 409 in progress. |
| 4 | `paymentService.ts:87-95` | Inside `$transaction`: insert the IN_PROGRESS record with `tx`. |
| 5 | `paymentService.ts:97-102` | Resolve the owner (`src/services/paymentAccessPolicy.ts:12-33`) and load it with `tx`. |
| 6 | `paymentService.ts:104-132` | Create the PENDING payment and the `PAYMENT_CREATED` event with `tx`. |
| 7 | `paymentService.ts:134-140` | `dispatchPaymentToProvider(..., tx)` writes attempt, provider mirror, payment view, event and optional webhook delivery. |
| 8 | `paymentService.ts:152-161` | Mark the idempotency record COMPLETED with the response snapshot. |
| 9 | `paymentService.ts:168-188` | A concurrent request that lost the race on the `(route, key)` unique index gets P2002 and re-reads the winner's record. |

Step 2's read sits outside the transaction, just as the refund reads used to — but here that is
safe, because step 4's insert is protected by a unique constraint (`prisma/schema.prisma:135`) and
step 9 handles the collision. The refund path has no equivalent constraint, which is why moving its
reads into the transaction matters and why it still is not enough on its own.

## Exercise

**Goal:** see why the default client hid the bug.
**Check:** open `astraupskill/snapshots/src/services/refundService.ts` and list every repository
call made *before* `prisma.$transaction`. For each, say which client it used (answer: the module-level
`prisma`, via the `db = prisma` default) and why TypeScript did not complain.
