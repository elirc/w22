# Trace lab

Fill each table from `src/services/refundService.ts` and `src/lib/domain/paymentStateMachine.ts`.

## Lab 1 — valid vs over-limit (payment 100, refunded 60)

| Step | Line | Valid 40 | Over 41 |
| --- | --- | --- | --- |
| transaction opens | 25 | yes | yes |
| payment read (tx) | 28 | SUCCEEDED, 100 | SUCCEEDED, 100 |
| successful sum (tx) | 32 | 60 | 60 |
| remaining | 33 | 40 | 40 |
| assertion | 34 | passes | 409 `REFUND_AMOUNT_EXCEEDS_CAPTURE` |
| create row (tx) | 36-44 | called | never called |
| REFUND_REQUESTED | 46-59 | written | — |
| refund -> SUCCEEDED | 62 | written | — |
| fullyRefunded | 64 | true | — |
| payment -> REFUNDED | 66-75 | written | — |
| REFUND_SUCCEEDED | 78-91 | written | — |
| HTTP | route `:19` / `:21` | 202 | 409 via `jsonError` |

## Lab 2 — failure boundaries

| Case | First thrown value | Last call made | Runtime test |
| --- | --- | --- | --- |
| missing payment | `RouteError(404, "PAYMENT_NOT_FOUND")` | payment read | `tests/runtime-refund-scope.test.mjs:43-45` |
| FAILED payment | `RouteError(409, "PAYMENT_NOT_REFUNDABLE")` | sum | `:38-42` |
| over-limit | `RouteError(409, "REFUND_AMOUNT_EXCEEDS_CAPTURE")` | sum | `:38-42` |
| event write fails | the original `Error` | `REFUND_REQUESTED` attempt | `:46-49` |

The first three are domain refusals; the last is an infrastructure failure that leaves a partial
ledger. Against PostgreSQL the transaction would roll the refund row back; the stub cannot show it.

## Lab 3 — concurrency worksheet

Draw two columns, A and B, each refunding 40 on payment 100 with 60 refunded. Mark when each runs
line 28, line 32, line 36 and commit. Then redraw with a row lock taken before line 28.
**Check:** the first drawing ends with 140 refunded; the second with 100 and B failing
`PAYMENT_NOT_REFUNDABLE`.

## Lab 4 — webhook path (seed or create a SUCCEED payment first)

Trace `POST /api/admin/webhooks/deliver` for one pending delivery.

| Step | File:line | Expected |
| --- | --- | --- |
| guard | `src/lib/auth/guards.ts:32` | OPS/ADMIN hold `webhook:deliver` |
| body | `src/app/api/admin/webhooks/deliver/route.ts:11-12` | empty body -> `{ limit: 10 }` |
| load | `src/services/providerSimulatorService.ts:158` | oldest PENDING deliveries first (`src/repositories/webhookRepository.ts:29-37`) |
| dedupe | `src/services/webhookService.ts:30-37` | no PROCESSED event yet |
| mirror | `webhookService.ts:39-49` | provider payment and local payment found |
| map | `webhookService.ts:51` | SUCCEEDED -> SUCCEEDED |
| transaction | `webhookService.ts:53-128` | event row, mirror update, WEBHOOK_RECEIVED, PENDING -> SUCCEEDED, STATUS_CHANGED, PROCESSED |
| delivery | `providerSimulatorService.ts:164` | DELIVERED, attempts + 1 |

**Check:** explain what happens to the `WebhookEvent` row if `transitionPaymentStatus` throws
inside the transaction (answer: the REJECTED mark at line 125 and the event row itself roll back
with the rethrow at 126; only `markWebhookDeliveryFailed` at `providerSimulatorService.ts:172`
persists, outside any transaction).

## Lab 5b — idempotent replay of payment creation

Send the same `POST /api/payments` body twice with the same `Idempotency-Key` header (for example
with `curl`, since the UI generates a new key per click — `src/hooks/usePaymentActions.ts:32`).

| Request | Lookup (`paymentService.ts:68`) | Branch | Response |
| --- | --- | --- | --- |
| first | no record | transaction, lines 87-167 | 202, `replayed: false` |
| second, same body | COMPLETED, same hash | `buildCompletedReplay` (line 80) | 202, same `paymentId`, `replayed: true` |
| third, changed `amountMinor` | COMPLETED, different hash | line 71-77 | 409 `IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_PAYLOAD` |
| two first-requests at once | both see no record | loser hits P2002, lines 168-188 | winner 202; loser replay or 409 in progress, depending on timing |

**Check:** explain why the replayed status code is 202 (`responseStatusCode` stored at line 157 and
read back at line 47) and why a refund request has no equivalent row.

## Lab 5 — reconciliation after a full refund

Using Lab 1's valid case, run reconciliation and fill: provider status (SUCCEEDED, unchanged),
expected local status (`resolveLocalStatusFromProviderStatus`, `paymentStateMachine.ts:34-48`),
actual local status (REFUNDED), webhook deliveries (one), issue type (`STATUS_MISMATCH`,
`src/services/reconciliationService.ts:60-81`).
