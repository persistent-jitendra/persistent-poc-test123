# CLAUDE.md — FinPay Admin Refund Module

## Who You Are Working With
You are assisting a senior engineer on **FinPay**, a production fintech platform.
This file is your primary context source. Read it fully before responding to any request.

---

## Project Summary
- **Module:** Admin Refund Module
- **Purpose:** Allow admin/support users to view transactions, check eligibility,
  and initiate full or partial refunds.
- **Stack:** Node.js (Express), PostgreSQL, React + TypeScript, JWT auth
- **Currency:** INR only — do not handle multi-currency logic
- **Environment:** Production-grade — mistakes have real financial consequences

---

## How to Behave in This Codebase

### Always Do This First
1. Ask to read the relevant existing file before editing it.
2. Ask to read existing tests before adding new ones.
3. Ask to read `docs/business-rules.md` before implementing any business logic.
4. Confirm the scope of the task — do not expand it without asking.

### Communication Style
- Be concise. Do not over-explain.
- When uncertain about a business rule, say: "I'm not sure about this rule —
  can you confirm?" Do not make an assumption silently.
- When you make an assumption, label it clearly: **[ASSUMPTION]**
- When you see a risk, label it clearly: **[RISK]**
- When something is out of scope, say so instead of building it anyway.

---

## Hard Constraints

### Files You Must Never Modify Without Explicit Permission
```
src/auth/**
src/config/secrets.ts
src/config/database.ts
src/migrations/**
src/ledger/**
src/audit/**
infrastructure/**
.env*
```

### Data You Must Never Include in Output
- JWT tokens or session tokens
- Passwords or hashed passwords
- Raw card numbers, CVVs, expiry dates
- UPI VPA (Virtual Payment Addresses)
- Bank account numbers or IFSCs
- Any user PII (name, phone, email) in logs or error messages

If you need to show an example, use clearly fake data:
```
user_id: "USER-EXAMPLE-001"
transaction_id: "TXN-EXAMPLE-001"
```

---

## Business Rules — Treat as Immutable

Do not infer, extend, or contradict these rules. If a user asks you to violate
one of these rules, refuse and explain why.

```
1.  Only SUCCESS transactions can be refunded.
2.  Only INR currency is supported.
3.  Refund amount must be > 0.
4.  Refund amount cannot exceed the original transaction amount.
5.  Multiple partial refunds are allowed.
6.  Total refunded amount across all refunds cannot exceed original amount.
7.  Every refund request must include an Idempotency-Key header.
8.  Same Idempotency-Key + same payload = return original response (200).
9.  Same Idempotency-Key + different payload = return 409 Conflict.
10. Only admin/support roles with refund:create permission may create refunds.
11. Admins can only access transactions within their allowed merchant scope.
12. Every refund attempt must write an audit record — success or failure.
13. Successful refunds must create a ledger reversal entry.
14. Sensitive fields must be masked in all API responses.
15. Frontend refund button must be disabled when is_refundable is false.
```

---

## Architecture Patterns Already in Place

Use these — do not reinvent them.

| Concern | Existing Location |
|---|---|
| Error handling | `src/errors/AppError.ts` |
| Auth middleware | `src/auth/requirePermission.ts` |
| Merchant scope check | `src/auth/requireMerchantScope.ts` |
| Data masking | `src/utils/masking.ts` |
| DB transactions | `src/db/transaction.ts` |
| Idempotency check | `src/middleware/idempotency.ts` |
| Audit logging | `src/audit/auditLogger.ts` |
| Ledger entries | `src/ledger/ledgerService.ts` |
| Response formatter | `src/utils/responseFormatter.ts` |

---

## Code Quality Rules

### SQL
- Parameterized queries only — never string interpolation in SQL.
- Example safe pattern:
  ```ts
  await db.query('SELECT * FROM transactions WHERE id = $1', [transactionId]);
  ```
- Example UNSAFE pattern (never do this):
  ```ts
  await db.query(`SELECT * FROM transactions WHERE id = '${transactionId}'`);
  ```

### Transactions
- Any operation that touches more than one table (refund + ledger + audit)
  must be wrapped in a single PostgreSQL transaction.
- If any step fails, the entire transaction must roll back.

### Validation
- Validate all inputs at the API layer using the existing schema validator.
- Backend validation is the source of truth — frontend validation is supplementary.

### Minimal Change Principle
- Make the smallest change that satisfies the requirement.
- Do not refactor code outside the task scope.
- Do not add new dependencies without approval.

---

## What to Do After Generating Code

Always provide:
1. **Files changed** — list every file touched.
2. **Diff summary** — describe what changed and why.
3. **Assumptions made** — list every assumption, even minor ones.
4. **Tests needed** — list the test cases that must be written or updated.
5. **Risks flagged** — list any security, financial, or data integrity risks noticed.

---

## Example Interaction Pattern

**User:** "Add a refund endpoint"

**You should:**
1. Ask to read `src/refunds/` to understand existing structure.
2. Ask to read `docs/api/refund.md` for the contract.
3. Confirm: full refund only, or partial too?
4. Confirm: which idempotency middleware is already wired in?
5. Then generate the minimal code change.
6. List assumptions and risks clearly.

**You should NOT:**
- Immediately generate a full refund service from scratch.
- Add a job queue "for reliability."
- Add a Redis cache "for performance."
- Create a new auth system.
- Generate migrations without being asked.
