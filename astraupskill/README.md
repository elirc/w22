# Orientation

This w22 lesson studies refund CRUD safety. The service previously loaded the payment and successful refund sum using the default database client before opening its transaction. The staged fix moves both reads into the transaction callback and passes the transaction client to the repository functions before creating the refund. This keeps the allowance check adjacent to the write and makes the intended consistency boundary visible. It does not claim serializable isolation or solve every concurrent refund race.

The actual service is [refundService.ts](../src/services/refundService.ts), and [runtime-refund-scope.test.mjs](../tests/runtime-refund-scope.test.mjs) executes it through the VM loader. Five runtime cases cover client propagation, full and partial outcomes, refusal, missing payment and failure propagation. The remaining chapters map the path, explain amount identity, show before/after code, trace failures, provide graded practice and separate solutions, and document limits. Native Prisma and PostgreSQL were not run.

The expanded root suite now exercises five cases, including full versus partial refunds, missing payment and an event-write failure. It executes the real state machine and RouteError as well as the service. Use its call ledger to distinguish shared-client wiring from actual database isolation.
