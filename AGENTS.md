# AGENTS.md — FinPay Admin Refund Module

## Purpose
This file instructs all AI agents (Copilot, Claude, Cursor, GPT, etc.) on how to
behave safely inside this fintech codebase. Read this file completely before
generating any code, suggestion, or refactor.

---

## 1. Project Context

This is **FinPay** — a fintech payment platform.
This module is the **Admin Refund Module**: it allows admin and support users to
view transactions, check refund eligibility, and initiate full or partial refunds.

**Stack:**
- Backend: Node.js (Express) + PostgreSQL
- Frontend: React + TypeScript
- Auth: JWT with role-based permissions
- Currency: INR only

---

## 2. Absolute Rules — Do Not Violate

### 2.1 Never Guess Business Logic
- Do NOT invent refund rules, eligibility criteria, amount limits, or status flows.
- All business rules are defined in `docs/business-rules.md` and `src/refunds/refund.rules.ts`.
- If a business rule is ambiguous or missing, STOP and ask. Do not assume.

### 2.2 Never Touch These Files Without Explicit Instruction
- `src/config/database.ts` — database connection config
- `src/config/secrets.ts` — secrets and env variable loader
- `src/migrations/` — database migrations
- `src/auth/` — authentication and authorization middleware
- `.env`, `.env.production`, `.env.staging` — environment files
- `src/ledger/` — financial ledger entries
- `src/audit/` — audit trail logic
- `infrastructure/` — deployment, Terraform, CI/CD configs

### 2.3 Never Expose Sensitive Data
- Do NOT log, print, or return: passwords, JWT tokens, raw card numbers, UPI VPAs,
  bank account numbers, or any PII.
- Payment data must be masked before sending to the frontend.
- Use the existing `maskSensitiveData()` utility in `src/utils/masking.ts`.

### 2.4 Backend Enforces All Rules
- Frontend validation is for UX only — never a security boundary.
- Every business rule (eligibility, amount limits, authorization, idempotency)
  MUST be enforced on the backend.
- Do not skip backend validation because the frontend "already checks it."

### 2.5 Do Not Overengineer
- Do not add new npm packages, services, queues, caches, or abstractions
  unless explicitly requested.
- Do not introduce Redis, Kafka, RabbitMQ, or any event bus unless instructed.
- Use the smallest safe change that satisfies the requirement.
- If you think a new abstraction is needed, explain why and wait for approval.

---

## 3. Business Rules Reference (Do Not Override)

| Area | Rule |
|---|---|
| Refund eligibility | Only `SUCCESS` transactions can be refunded |
| Currency | Only `INR` is supported |
| Refund amount | Must be > 0 |
| Refund limit | Cannot exceed original transaction amount |
| Partial refunds | Multiple partial refunds are allowed |
| Over-refund guard | Total refunded amount cannot exceed original amount |
| Idempotency | Every refund request must carry an `Idempotency-Key` header |
| Duplicate request | Same key + same payload → return original response (200) |
| Idempotency conflict | Same key + different payload → return 409 Conflict |
| Authorization | Only `admin`/`support` roles with `refund:create` permission |
| Merchant scope | Admin can only access transactions for their allowed merchants |
| Audit trail | Every refund attempt (success or failure) must be logged in `refund_audit` |
| Ledger | Successful refund must create a reversal entry in `ledger_entries` |
| Data masking | Sensitive payment fields must be masked in all API responses |
| Frontend gate | Refund button disabled when `is_refundable: false` |

---

## 4. Before Writing Any Code

1. Read the relevant existing source file(s) first.
2. Read the existing tests for the area you are changing.
3. Read the API contract in `docs/api/` if it exists.
4. Identify the smallest change needed.
5. Do not create new files or folders unless absolutely necessary.

---

## 5. Code Generation Standards

### 5.1 Error Handling
- Use the existing `AppError` class in `src/errors/AppError.ts`.
- Return structured errors: `{ error: { code, message, details } }`.
- Never expose stack traces or internal error messages to the client.

### 5.2 Authorization Pattern
- Always call `requirePermission('refund:create')` middleware before refund routes.
- Always call `requireMerchantScope()` middleware to validate merchant access.
- Never inline auth logic in controllers — use the existing middleware.

### 5.3 Database
- Use parameterized queries only. Never concatenate user input into SQL.
- Wrap multi-step financial operations (refund + ledger + audit) in a single
  database transaction.
- Do not alter existing column names or table schemas without a migration file.

### 5.4 Idempotency Pattern
- Check `idempotency_keys` table before processing.
- If key exists and payload hash matches → return stored response.
- If key exists and payload hash differs → return 409.
- Store key + payload hash + response after successful processing.

---

## 6. After Generating Code

1. Show a diff of what changed — do not silently rewrite files.
2. List every file modified.
3. Flag any assumption made (even small ones).
4. Identify any place where a business rule was unclear.
5. Do NOT run migrations, deploys, or seed scripts automatically.

---

## 7. Testing Requirements

- Every new backend function must have a corresponding unit test.
- Refund endpoint must have integration tests covering:
  - Happy path (full refund)
  - Happy path (partial refund)
  - Duplicate idempotency key (same payload)
  - Idempotency conflict (different payload)
  - Over-refund attempt
  - Unauthorized role
  - Out-of-scope merchant
  - Non-SUCCESS transaction
- Do not delete or modify existing tests unless explicitly asked.

---

## 8. What AI Must Not Do — Summary Checklist

- [ ] Must not guess or invent business rules
- [ ] Must not touch auth, ledger, audit, migration, or secrets files
- [ ] Must not add unrequested dependencies or infrastructure
- [ ] Must not log or return PII, tokens, or raw payment data
- [ ] Must not skip backend validation
- [ ] Must not run destructive commands (DROP, DELETE, migrate, deploy)
- [ ] Must not silently modify files outside the stated task scope
- [ ] Must not bypass idempotency or audit logging
