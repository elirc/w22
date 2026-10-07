# Auth And Permissions

Session creation happens in `src/app/api/auth/login/route.ts`, which parses input with `loginSchema.parse()` from `src/lib/validation/authSchemas.ts` and then calls `login()` in `src/services/authService.ts`.

Session verification happens in `src/lib/auth/session.ts`. `getSessionForRequest()` reads the cookie name from `src/lib/auth/cookies.ts`, loads the session row through `findSessionByToken()` in `src/repositories/sessionRepository.ts`, and rejects expired sessions before routes see user data.

Route-level authorization is centralized in `src/lib/auth/guards.ts`. `withRouteAuth()` calls `requirePermission()` from `src/lib/auth/authorization.ts` before the handler body runs. That means `src/app/api/payments/route.ts`, `src/app/api/admin/webhooks/deliver/route.ts`, and `src/app/api/admin/reconciliation/run/route.ts` all share the same authentication and RBAC path.

The RBAC policy itself is defined in `src/lib/permissions.ts`. The mapping is explicit:
- `CUSTOMER` can create payments and read only their own payments.
- `SUPPORT` can read any payment but cannot create, refund, deliver webhooks, or run reconciliation.
- `OPS` can create payments, read any payment, refund, deliver webhooks, and run reconciliation.
- `ADMIN` has the same operational permissions as `OPS` in this codebase.

Per-record access checks are separate from route-level permission checks. `src/services/paymentQueryService.ts` calls `assertCanReadPaymentRecord()` from `src/services/paymentAccessPolicy.ts` after loading the payment, which in turn uses `canReadPayment()` from `src/lib/auth/authorization.ts`. This is the second line of defense if a broader read route ever exists later.

Ownership scoping for list queries is centralized in `src/services/paymentAccessPolicy.ts`. `buildPaymentListScope()` returns `{ ownerUserId: session.userId }` for `CUSTOMER` and `{}` for staff roles. That function is the code path a stronger reviewer should protect most aggressively because removing it would leak customer data from list views.

Page-level authorization is also centralized. `src/app/admin/layout.tsx` calls `requireStaffPaymentsPage()` from `src/lib/auth/guards.ts`, and `src/app/admin/reconciliation/page.tsx` calls `requirePagePermission(Permission.RECONCILIATION_RUN)` so support users can inspect payments but cannot access reconciliation operations.

Gaps a reviewer should know about:
- `src/lib/auth/guards.ts:10` imports `RouteError` from `../api/route-helpers`, which does not export it (it only imports it at `src/lib/api/route-helpers.ts:10`). Type-checking should flag this; at runtime the `instanceof RouteError` check in `requirePagePermission()` (line 48) cannot work, so the "403 redirects to `/`" path for pages is broken. Import it from `../errors` instead.
- `withRouteAuth()` throws the `RouteError` from `requirePermission()` at `src/lib/auth/guards.ts:32`, before the wrapped handler and outside its `try`/`catch`. Unauthenticated (401) and forbidden (403) API calls therefore do not go through `jsonError()` in `src/lib/api/route-helpers.ts`.
- `src/app/api/webhooks/provider/route.ts` is not wrapped and verifies no signature (its own comment at line 8 says so). `processProviderWebhook()` casts the body without validation (`src/services/webhookService.ts:29`), so any caller who knows a `providerPaymentId` can drive a legal payment transition such as `PENDING -> SUCCEEDED`.

Sessions are revocable server-side: logout deletes the session row (`src/services/authService.ts:37-39`), and every request re-reads the row and the user's current role (`src/lib/auth/session.ts:30-43`).

This project is not multi-tenant. To add multi-tenancy later without a rewrite:
- Add `tenantId` to tenant-owned models in `prisma/schema.prisma`, especially `Payment`, `ProviderPayment`, `WebhookDelivery`, `WebhookEvent`, `Refund`, and `ReconciliationIssue`.
- Extend `AuthenticatedSession` in `src/lib/auth/session.ts` to include `tenantId`.
- Change `buildPaymentListScope()` in `src/services/paymentAccessPolicy.ts` to return both `tenantId` and ownership filters.
- Update every repository function in `src/repositories/**` that reads tenant-owned data so the `where` clause scopes by `tenantId` before business filters are applied.
- Add a tenant-aware permission layer alongside `src/lib/permissions.ts` so platform admins and org-level operators are separated explicitly.
