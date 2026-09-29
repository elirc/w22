# Verification

Executed command: `node --experimental-vm-modules --test tests/runtime-refund-scope.test.mjs`; accepted expanded result 5/5 passed. The first case proves payment and prior-refund reads use the transaction client before create. The second proves remaining allowance rejects an over-refund without creating a row. The test loads the actual service source with explicit doubles. Native Prisma, PostgreSQL isolation, HTTP auth, provider refund dispatch, and concurrent real transactions were not executed. The exact original service is in `astraupskill/snapshots/src/services/refundService.ts`. The evidence supports transaction-client propagation and local allowance ordering only.

The accepted expanded suite is captured as crud-b04-astra-runtime and has five passing cases. It executes actual service, state machine and RouteError, with explicit repository, transaction and UUID adapters. All read and write collaborator calls assert the shared transaction handle. No real payment provider is called. The failed-event case verifies propagation and stopping order, not durable rollback. The repository defaults and route validation were source-reviewed; route authorization and input-schema execution were not part of this command.

## Astra execution evidence

The captured focused production-source suite executed 5 tests: all passed, zero failed and zero skipped. Read the [captured output](evidence/crud-b04-astra-runtime.log) and [exact command and limits](evidence/results.json). Earlier check records remain preserved in the workspace.
