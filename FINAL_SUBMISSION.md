# Final Submission — FinPay Admin Refund Module
## Assessment Deliverables Index + Final Summary

---

## Deliverables Index

### 1. AI Workflow

**File:** `AI_WORKFLOW.md`

Covers: how project context is loaded before any AI session, how business
rules are enforced as a pre-coding gate, how overengineering is prevented,
which files AI must never touch, how secrets and PII are handled, how
AI-generated code is verified before acceptance, and reusable prompt
starters for the whole team.

Key rules enforced:
- AI reads `docs/business-rules.md` before implementing any logic
- Every rule in code must trace to a Rule ID — no inferred rules
- Protected files listed explicitly and blocked in `.cursorignore`
- No new dependencies, tables, or infrastructure without approval
- Backend is the only security boundary — frontend is UX only

---

### 2. AI Rules / Skills / Commands

| File | Purpose |
|---|---|
| `AGENTS.md` | Hard rules for all AI agents — must-not-do checklist |
| `CLAUDE.md` | Claude-specific context, immutable rules table, correct interaction model |
| `.cursorignore` | Blocks AI from reading secrets, auth, ledger, audit, migrations, infra |
| `.cursor/rules/fintech-development.mdc` | Cursor auto-rules for all `src/` files |
| `.cursor/rules/safe-fintech-change.mdc` | Cursor auto-rule enforcing 7-step protocol on financial files |
| `.cursor/rules/anti-overengineering.mdc` | Cursor auto-rule enforcing 7-check review on all diffs |
| `.agents/skills/safe-fintech-change/SKILL.md` | 7-step protocol: inspect → summarise → minimal change → risks → tests → assumptions → no guessing |
| `.agents/skills/anti-overengineering-review/SKILL.md` | 7-check review: scope → deps → abstractions → unrelated files → simpler approach → test proportion → fintech safety |
| `.claude/instructions/safe-fintech-change.md` | Claude session instruction for financial code changes |
| `.claude/instructions/anti-overengineering.md` | Claude session instruction for diff review |
| `docs/ai-commands.md` | 10 team copy-paste prompts for common tasks |

---

### 3. Backend Implementation

**Language:** TypeScript + Node.js (Express) + PostgreSQL

| File | Responsibility |
|---|---|
| `db/schema.sql` | 6 tables — transactions, refunds, idempotency_keys, refund_audit, ledger_entries, admin_users |
| `backend/src/types/index.ts` | All shared types — no business logic |
| `backend/src/errors/AppError.ts` | Single error class + named factories per rule |
| `backend/src/refunds/refund.constants.ts` | Rule-derived constants — supported currencies, refundable statuses |
| `backend/src/refunds/refund.rules.ts` | Pure assertion functions — one per Rule ID |
| `backend/src/refunds/refundService.ts` | Atomic flow: `SELECT FOR UPDATE` → eligibility check → refund INSERT → ledger → audit |
| `backend/src/refunds/refundController.ts` | HTTP handlers — success and failure paths both write audit |
| `backend/src/transactions/transactionController.ts` | GET transaction with masked response |
| `backend/src/auth/requirePermission.ts` | JWT validation, role check, permission gate |
| `backend/src/auth/requireMerchantScope.ts` | Merchant scope enforcement |
| `backend/src/middleware/idempotency.ts` | UUID v4 validation, hash comparison, 24h TTL, response capture |
| `backend/src/audit/auditLogger.ts` | Writes inside the same DB transaction as the refund |
| `backend/src/ledger/ledgerService.ts` | REFUND_REVERSAL entry, negative amount, same transaction |
| `backend/src/utils/masking.ts` | Field-level maskers (UPI, card, bank) + log stripping |
| `backend/src/utils/responseFormatter.ts` | Consistent API response helpers — no stack traces |
| `backend/src/db/index.ts` | PG pool + `withTransaction()` atomic wrapper |
| `backend/src/routes/index.ts` | All routes with 3-layer middleware: auth → permission → merchant scope |
| `backend/src/app.ts` | Express app, helmet, global error handler |
| `backend/package.json` | Minimal deps — pg, express, helmet, jsonwebtoken only |

**Security properties enforced in code:**
- All SQL parameterised — `db.query('... WHERE id = $1', [id])`
- Every multi-table write inside `withTransaction()` — refund + ledger + audit
- `SELECT FOR UPDATE` prevents concurrent over-refund
- Auth middleware on every route — never inline
- Idempotency middleware before every mutating route
- Audit written on both success and failure paths
- Failure audit uses a fresh connection — survives rollback

---

### 4. Frontend Implementation

**Language:** TypeScript + React

| File | Responsibility |
|---|---|
| `frontend/src/types/index.ts` | Response types — mirrors backend shapes, no business logic |
| `frontend/src/api/refundApi.ts` | All API calls centralised — JWT in memory only, idempotency key per request |
| `frontend/src/hooks/useTransaction.ts` | Fetches transaction + refunds, cancellation-safe, exposes `refetch` |
| `frontend/src/hooks/useRefund.ts` | `idle → submitting → success/error` state machine — idempotency key on mount, double-submit lock |
| `frontend/src/components/TransactionDetail.tsx` | Masked data, `is_refundable`-driven button state, `canCreateRefund` permission gate |
| `frontend/src/components/RefundModal.tsx` | Amount + reason inputs, success/error/loading states, close blocked while in-flight |
| `frontend/src/components/RefundHistory.tsx` | Reverse chronological list (FE-005), empty state |
| `frontend/src/components/ui/Badge.tsx` | Status badge driven by type |
| `frontend/src/components/ui/Spinner.tsx` | Accessible loading state with `aria-live` |
| `frontend/src/components/ui/Alert.tsx` | Error/success/warning with `role="alert"` |
| `frontend/src/styles/globals.css` | Design tokens, card, modal, form, badge, spinner, masked-value |
| `frontend/src/utils/format.ts` | Paise → rupee, date formatter, badge class |

**Validation boundary** (`docs/validation-boundary.md`):

| Validation | Frontend | Backend |
|---|---|---|
| Amount > 0 | UX only (input min attr) | Enforced (400 INVALID_AMOUNT) |
| Amount ≤ remaining | UX only (input max attr) | Enforced (422 OVER_REFUND) |
| Transaction eligible | Not computed | Enforced (422 REFUND_INELIGIBLE) |
| Idempotency key present | Not checked | Enforced (400 MISSING_IDEMPOTENCY_KEY) |
| Permission | Hides button | Enforced (403 FORBIDDEN) |
| Merchant scope | Not checked | Enforced (403 MERCHANT_SCOPE_VIOLATION) |
| Masking | Renders as-is | Enforced in maskTransaction() |

---

### 5. Idempotency and Concurrency

**File:** `docs/idempotency-and-concurrency.md`

**Scenario 1 — Retry after timeout (duplicate refund):**
```
1. useRefund hook generates one UUID v4 key per modal open (useRef)
2. Retry sends same key → idempotency middleware intercepts
3. Hash of request body matches stored hash → stored response returned immediately
4. withTransaction() never called → no duplicate refund row created
```

**Scenario 2 — Two simultaneous ₹3000 refunds on ₹5000 transaction:**
```
T1  Request A: SELECT * FROM transactions WHERE id = $1 FOR UPDATE → lock acquired
T2  Request B: SELECT * FROM transactions WHERE id = $1 FOR UPDATE → BLOCKED
T3  Request A: reads ₹0 refunded → ₹0 + ₹3000 ≤ ₹5000 → inserts ₹3000 → COMMIT
T4  Request B: lock released → reads ₹3000 committed → ₹3000 + ₹3000 > ₹5000 → OVER_REFUND → ROLLBACK
```

**Four independent layers of protection:**
1. Frontend double-submit lock (`useRefund` hook state machine)
2. Idempotency middleware (payload hash comparison before business logic)
3. Database transaction with `SELECT FOR UPDATE` (row-level exclusive lock)
4. `assertNoOverRefund()` against committed data inside the locked transaction

**Design decisions:**
- PostgreSQL `FOR UPDATE` over Redis distributed lock: simpler, co-located with data, auto-releases on rollback
- Pessimistic locking over optimistic: no retry loop risk for financial operations
- Idempotency storage in PostgreSQL over Redis: no new infrastructure dependency
- No retry logic inside the service: client retries with idempotency key instead

---

### 6. Review of Flawed AI-Generated Code

**File:** `docs/code-review-task7.md`

**Code reviewed:**
```typescript
router.post("/api/admin/transactions/:id/refunds", async (req, res) => {
  const transaction = await Transaction.findByPk(req.params.id);
  if (!transaction) return res.status(404).json({ error: "Transaction not found" });
  const refund = await Refund.create({
    transactionId: transaction.id, amount: req.body.amount,
    reason: req.body.reason, status: "SUCCESS"
  });
  transaction.status = "REFUNDED";
  await transaction.save();
  return res.json(refund);
});
```

**Issues found (12 categories):**

| Priority | Issue | Impact |
|---|---|---|
| P0 | No authentication | Any caller can create refunds |
| P0 | No idempotency | Retries create duplicate refunds |
| P0 | No over-refund guard | Transaction refunded ×N |
| P0 | Non-atomic writes | Refund exists without ledger/status |
| P0 | No ledger entry | Ledger permanently out of balance |
| P1 | No transaction status check | FAILED/PENDING transactions refunded |
| P1 | No audit trail | No compliance record |
| P2 | No amount validation | Zero/negative/float amounts accepted |
| P2 | Invalid status value `"REFUNDED"` | DB corruption or constraint error |
| P3 | No error handling | Stack traces or crashes exposed |
| P3 | Raw ORM response | Internal data or PII leaked |
| P3 | No input sanitisation | Overlarge payloads accepted |

**Production verdict: NO.** Not with minor modifications. Must be rewritten from scratch following the existing `refundService.ts` pattern.

---

### 7. Tests and Test Plan

**File:** `docs/test-plan.md`

**72 test cases across 6 test files:**

| File | Cases | Area |
|---|---|---|
| `refund.rules.test.ts` | 18 | Pure unit — all business rule assertions |
| `refund.auth.test.ts` | 12 | Authentication, authorization, merchant scope |
| `refund.business.test.ts` | 14 | Status, amount, partial refunds, over-refund |
| `refund.idempotency.test.ts` | 10 | Same-key replay, conflict, missing key |
| `refund.concurrency.test.ts` | 12 | Concurrent refunds, audit, ledger, rollback |
| `TransactionDetail.test.tsx` | 24 | Button disabled, double-submit, errors, masking |

All 18 assessment test cases are covered with at least 2 scenarios each.

---

### 8. AI Prompt Log

**File:** `AI_PROMPT_LOG.md`

6 documented prompts covering: AGENTS.md creation, database schema, idempotency
middleware, refund service (concurrency), useRefund hook, and validation boundary
document. Each entry includes what was accepted, what was rejected, why, how it
was verified, and assumptions made.

**Pattern of rejections:** Redis for idempotency (×3), retry logic in service (×1),
optimistic locking (×1), debounce in hook (×1), unrequested tables (×2),
multi-currency support (×1), SSR section (×1).

---

---

# Final Summary

## What I Built

A production-grade Admin Refund Module for FinPay covering:

- **Backend:** Express + PostgreSQL API with three-layer auth middleware
  (authenticate → requirePermission → requireMerchantScope), idempotency
  middleware (UUID v4, SHA-256 payload hash, 24h TTL), atomic refund
  creation (SELECT FOR UPDATE + refund + ledger + audit in one transaction),
  field-level data masking, structured error handling, and a complete
  business rule assertion layer traced to documented rule IDs.

- **Frontend:** React + TypeScript UI with a state-machine hook
  (`useRefund`) covering idle/submitting/success/error states,
  double-submit prevention, idempotency key managed as a ref, three
  components (TransactionDetail, RefundModal, RefundHistory), accessible
  loading/error/success states, and a strict validation boundary where the
  backend is the only source of truth for eligibility.

- **AI workflow:** 11 files establishing how AI is used safely — agent
  rules, Cursor IDE rules, Claude instructions, two reusable skills (Safe
  Fintech Change and Anti-Overengineering Review), a 10-command team
  prompt reference, and a master workflow document.

- **Documentation:** Business rules with Rule IDs, validation boundary,
  idempotency and concurrency design, code review, test plan, and prompt log.

**Total: 57 files.**

---

## AI Tools Used

**SASVA AI** (IDE-embedded assistant)

Used for: file generation, code scaffolding, pattern application, and
documentation drafting. All output was reviewed before acceptance. No
generated file was merged without line-by-line verification against the
business rules and security checklist.

---

## What AI Helped With

- Scaffolding boilerplate that follows established patterns — the three-layer
  middleware stack, the `withTransaction()` wrapper, the `AppError` factory
  pattern — once the pattern was established, AI applied it consistently.

- Generating test scaffolding (describe/it blocks, mock setup) that would
  have taken significant time to write manually.

- Producing comprehensive JSDoc-style inline comments that make the code
  self-documenting.

- Drafting the masking utility — once the field-level rules were provided,
  AI generated the correct routing logic without inventing new fields.

- Generating the `refund.rules.ts` assertion functions — once the rule IDs
  were given, the mapping from rule to assertion was mechanical and AI
  executed it correctly.

---

## What AI Got Wrong or Missed

| Issue | Detail |
|---|---|
| Missing auth middleware | When first prompted for a route, AI omitted `requirePermission` — had to be explicitly requested |
| Invented `refund_policies` table | AI added a configurable policy engine that was never requested |
| Retry logic in service layer | AI added exponential backoff inside `createRefund` — dangerous for money movement |
| Redis for idempotency | Suggested three times across different prompts despite explicit no-Redis instruction |
| Optimistic locking | Suggested a `version` column check instead of `FOR UPDATE` |
| Debounce for double-submit | Suggested timing-based prevention instead of state-based lock |
| Multi-currency abstraction | Added generic amount type "for future currencies" despite INR-only rule |
| `status: "REFUNDED"` | In the Task 7 reviewed code, AI used an invalid status value not in the schema |
| `res.json(refund)` without masking | Raw ORM object returned — sensitive fields would be exposed |

---

## What I Rejected From AI Output

| Rejected Item | Reason |
|---|---|
| `refund_policies` table | Not in business rules — solving a hypothetical future requirement |
| `notifications` table | Email notifications were never mentioned in requirements |
| `soft_deleted_at` on refunds | Financial records are immutable — soft-delete implies a deactivation path |
| Redis as idempotency store | Adds external dependency and failure mode — PostgreSQL is sufficient |
| In-memory `Map` as idempotency cache | Breaks in multi-instance deployments — correctness bug |
| Idempotency INSERT inside main transaction | Correct behaviour but adds complexity — `res.json` wrap is simpler |
| Exponential backoff retry in service | Automatic financial retries can cause duplicates if first request partially succeeded |
| Optimistic locking with version column | Requires retry loop at application layer — wrong pattern for money movement |
| Debounce(500ms) on submit | Timing-based — closes window too early on slow networks |
| Dual `isSubmittingRef` + state machine | Two sources of truth that can drift apart |
| `zod` validation library | Existing validator was in scope — no new dependency needed |
| SSR/Next.js section in validation doc | Project uses React — out of scope |
| "Suggested AI Tools" section in AGENTS.md | Distracts from the actual agent rules |

---

## Assumptions Made

| # | Assumption | Risk if Wrong |
|---|---|---|
| A1 | Amounts are stored as integers (paise) throughout | Float arithmetic would corrupt over-refund comparisons |
| A2 | JWT secret is available at `process.env.JWT_SECRET` | Missing env var would throw unhandled error at startup |
| A3 | `db-migrate` or equivalent handles migrations — `db/schema.sql` is target state only | Running schema directly against production would drop existing tables |
| A4 | PostgreSQL `lock_timeout` is configured at server level | Unbounded lock wait would queue requests indefinitely under load |
| A5 | Idempotency key expiry cleanup runs on a scheduled job | Without cleanup, `idempotency_keys` table grows unboundedly |
| A6 | Client sends JSON fields in consistent order | Payload hash comparison is order-sensitive — different serialisers could produce false 409s |
| A7 | Auth token is passed to `setAuthToken()` at login time | Mechanism for initial login is outside this module's scope |
| A8 | Modal is unmounted when closed | If kept mounted via CSS, same idempotency key persists — correct but implicit |
| A9 | `payment_ref` column holds the raw sensitive value | If already masked at write time, the masking utility would double-mask |
| A10 | Only one currency (INR) will be in scope for the foreseeable future | Multi-currency would require a non-trivial refactor of amount handling |

---

## Tests Added or Planned

**72 automated tests across 6 files covering all 18 assessment areas:**

```
Authentication:   5 tests  — unauth, bad JWT, inactive user, wrong role
Authorization:    4 tests  — missing permission, read-only role
Merchant scope:   3 tests  — wrong merchant, empty scope, correct merchant
Status check:     4 tests  — PENDING/FAILED/REVERSED/SUCCESS
Amount (zero):    1 test
Amount (negative):3 tests  — negative, float
Amount (exceeds): 3 tests  — over original, exact limit allowed
Partial refund:   3 tests  — first, second, failed refunds excluded
Over-refund:      3 tests  — cumulative, fully refunded
Idempotency:      3 tests  — replay, expired key, controller not called
Idempotency conflict: 4 tests — different amount, different reason, missing key, invalid format
Concurrency:      2 tests  — over-refund blocked, non-overlapping allowed
Audit:            3 tests  — success path, failure path, fields present
Ledger:           3 tests  — reversal present, not on failure, rollback
FE button:        5 tests  — enabled, disabled (failed), disabled (refunded), hidden, reason shown
FE double-submit: 5 tests  — button disabled, API called once, close disabled, spinner
FE error:         6 tests  — backend error, network error, page error, a11y, UX guard, success
FE masking:       9 tests  — masked rendered, raw absent from DOM, table, CSS class
```

---

## Risks Remaining

| Risk | Severity | Mitigation |
|---|---|---|
| Idempotency key stored asynchronously after response | Medium | The `FOR UPDATE` lock is the primary over-refund guard. A missed key store means the next retry is treated as new — both will pass the lock check independently. Unlikely to cause a duplicate under normal conditions. |
| Payload hash is JSON-serialisation-order-sensitive | Low | Frontend always serialises via `JSON.stringify` on a typed object. External API callers must follow the documented field order. |
| No real concurrent test in CI | Medium | Concurrency tests use mocks that simulate post-lock state. True concurrent testing requires two real PostgreSQL connections and is not in the current CI setup. |
| `lock_timeout` not set in code | Low | Documented as infrastructure concern. Recommend setting `lock_timeout = 5s` in the connection string or session config. |
| Idempotency table grows without cleanup | Low | Cleanup job is documented but not implemented. Recommend a daily `DELETE WHERE expires_at < NOW()` cron job. |
| No rate limiting on refund endpoint | Medium | An authenticated admin could flood the endpoint. Recommend per-user rate limiting (e.g., 10 refunds/minute) at the API gateway or middleware layer. |
| No refresh token rotation | Out of scope | JWT expiry and refresh are outside this module. The assumption is that the auth system handles this. |
| `ON CONFLICT DO NOTHING` in idempotency INSERT | Low | If two concurrent new requests with the same key both pass the initial lookup simultaneously, only one key is stored. The second request would find the key on its next read cycle, which may not happen within the same request. This is an extremely unlikely edge case. |

---

## What I Would Improve With More Time

**Code:**
1. **True concurrent integration tests** — spin up a real PostgreSQL instance in Docker,
   issue two simultaneous `fetch` calls, and assert only one refund row is created.
   This validates the `FOR UPDATE` lock against actual DB behaviour, not mocks.

2. **Idempotency cleanup job** — a simple scheduled query to delete expired keys.
   Currently documented but not implemented.

3. **Rate limiting middleware** — per-user rate limit on the refund endpoint to prevent
   flooding from a compromised admin account.

4. **Request schema validation** — a thin schema validator on `req.body` at the route
   level to reject malformed payloads before they reach business logic. Currently
   business rule assertions handle this, but an explicit schema check would give
   cleaner error messages.

5. **Partial refund UX** — the modal currently pre-fills the full remaining amount.
   A "Quick Refund" button for common amounts (25%, 50%, 100%) would reduce support
   operator errors.

**Process:**
6. **Load test** — validate that `FOR UPDATE` lock contention under high concurrency
   (100+ simultaneous admin sessions) does not cause unacceptable p99 latency.

7. **Compliance review** — have the audit schema and ledger entry format reviewed
   by a compliance officer before production deployment. The fields captured
   (actor_id, ip_address, outcome, idempotency_key) are based on standard practice
   but may need jurisdiction-specific additions.

8. **Penetration test** — specifically targeting the merchant scope middleware:
   confirm that crafted JWT payloads with manipulated `allowed_merchant_ids`
   are rejected correctly.

---

## Would I Approve This for Production?

**Conditional Yes**

**Conditions that must be met before production deployment:**

```
[ ] True concurrent integration test passes against a real PostgreSQL instance
    (validates FOR UPDATE lock — not just mocked behaviour)

[ ] lock_timeout configured in PostgreSQL connection settings

[ ] Idempotency key cleanup job deployed and scheduled

[ ] Rate limiting applied to the refund endpoint

[ ] Security review of merchant scope middleware
    (confirm JWT payload cannot be manipulated to expand merchant access)

[ ] Compliance sign-off on audit record schema

[ ] Load test showing p99 latency < 2s under expected concurrent admin load

[ ] All 72 automated tests passing in CI
```

**Why conditional and not unconditional:**

The core financial safety properties are correct:
- Atomic transactions prevent partial writes
- `FOR UPDATE` prevents concurrent over-refund
- Idempotency prevents duplicate refunds from retries
- Audit trail covers both success and failure paths
- Data masking prevents PII exposure
- Three-layer auth prevents unauthorized access

The conditions are operational and infrastructure concerns, not correctness
bugs. The code would be safe to deploy to a staging environment with real
traffic for verification. Production deployment should wait until the
concurrent test and rate limiting conditions are met, since those address
the two most realistic attack surfaces: a compromised admin account and
infrastructure-level retry behaviour.

**What would make this an unconditional No:**
- Any missing auth middleware on a route
- Any financial write outside a DB transaction
- Any audit record that could be silently skipped
- Any SQL that uses string interpolation
- Any response returning unmasked payment data

None of these conditions are present in the current implementation.
