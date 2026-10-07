# Verification

## What was executed (historical record)

One command, recorded in [evidence/results.json](evidence/results.json):

```
node --experimental-vm-modules --test tests/runtime-refund-scope.test.mjs
```

Recorded result: `"passed": true`, `"executedTests": 5`, `"exitCode": 0`, 9.63 s, run on
2026-09-22 in a staging copy (the `cwd` field points at a staging directory, not this clone). The
log the JSON names (`crud-b04-astra-runtime.log`) is not committed here; only the JSON is. Nothing
was re-run while this documentation was refined.

## What the five cases execute

The real `src/services/refundService.ts`, `src/lib/domain/paymentStateMachine.ts` and
`src/lib/errors.ts`, with stubbed Prisma enums, transaction, repositories and UUID
(`tests/runtime-refund-scope.test.mjs:10-26`). Each stub asserts it received the shared `tx`
(line 9). The cases are tabulated in 04.

## Limits (from results.json, still accurate)

- No native Prisma, PostgreSQL, HTTP, or provider call was executed.
- Passing the transaction client keeps reads and writes in one transaction, but default isolation
  does not prevent simultaneous refunds from exceeding the allowance. No locking, serializable
  retry, or durable refund idempotency was added.
- The service simulates provider success with a generated id; the tests do not establish real
  provider refunds, real rollback, or request authorization.

## What was source-reviewed but not executed

The route guard (`src/lib/auth/guards.ts`), the refund schema
(`src/lib/validation/paymentSchemas.ts:24-37`), and the repository defaults were read, not run. The
findings in 06 come from reading and are not test results.

## How to reproduce

From the repo root on a Node version whose `node:module` exports `stripTypeScriptTypes` (imported
by `tests/helpers/load-typescript.mjs`), run the command above. It needs no installed packages.
**Check:** five passing tests. An `Unstubbed external dependency` error means the service gained an
import that the fixture at lines 10-26 does not stub.
