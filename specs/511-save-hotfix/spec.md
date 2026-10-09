# Spec Kit: v5.1.1 emergency save hotfix

Date: 2026-10-09. Status: LOCAL VERIFIED; production not deployed or verified.

## Reproduced error
Production employee cleanup POST /api/state returned HTTP 503 / Cloudflare 1102, Ray a47d1fc158a6dc4b-FRA, 15:12:50 Tehran (11:42:50 UTC), operation hotfix511-1791546125663-cleanup-test-employees, baseRevision 278. HTML response: Worker exceeded resource limits. CPU versus memory is UNKNOWN without invocation logs.

## Root cause / confirmed hotspot
Inspection covered the Admin save authorization, employee-deletion integrity checks, credential cleanup, state read/write and operation receipts. hasAuthorizedEvaluationChanges unnecessarily serialized the complete server-owned audit history on every Admin save, even when pe_audit_logs was absent from the request. The existing representative dataset contains 626 employees, 624 evaluations and 10,000 audit entries: the discarded audit JSON is 8,002,629 characters (8,519,127 UTF-8 bytes). It also built the entire evaluation lookup before returning for an employee-only write. These are proven redundant allocations/processing, not proof of the precise production 1102 resource. The required persisted-state serialization remains.

## Exact files changed
functions/api/state.ts: short-circuit audit serialization behind the existing client-audit-field check; construct the evaluation lookup only for evaluation writes. Audit comparison, actor/permission checks, protected references, revision checks and idempotent operation receipts are unchanged. Only this existing record is updated besides that source file. No configuration changes or production deployment. Accepted v5.1 archives remain unchanged.

## One real backend acceptance scenario
Actual Wrangler Pages Functions and isolated CHALAK_DB KV, seeded with the existing dataset. Admin create two disposable employees: POST /api/state 200, revision 270; backend read confirmed both. Real Employees UI selected those two and confirmed bulk deletion once: POST /api/auth/password 200, POST /api/state 200, revision 271. Reloaded the browser and independently GET /api/state: 200; both test IDs absent; all 626 original employees and 624 evaluations deeply equal to baseline. Only audit history changed, with two creation and two deletion audit entries under the existing retention policy. No 503 and no POST retry storm. Typecheck and production build each ran once and passed. This verifies local persistence only; production 1102 elimination and CPU/memory before/after remain unverified.
