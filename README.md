# FinPay Admin Refund Module

A production-grade full stack refund management system for the FinPay
fintech platform. Allows admin and support users to view transactions,
check refund eligibility, and initiate full or partial refunds safely.

---

## Folder Structure

```
finpay-admin-refund-module/
│
├── README.md                          # This file
├── AGENTS.md                          # Hard rules for all AI agents
├── CLAUDE.md                          # Claude-specific project context
├── AI_WORKFLOW.md                     # Master AI development workflow
├── AI_PROMPT_LOG.md                   # AI prompt log — accepted/rejected output
├── FINAL_SUBMISSION.md                # Full assessment deliverables + final summary
├── .cursorignore                      # Files AI must never read or modify
│
├── .cursor/
│   └── rules/
│       ├── fintech-development.mdc    # Auto-applied Cursor rules for src/**
│       ├── safe-fintech-change.mdc    # 7-step protocol for financial code
│       └── anti-overengineering.mdc  # 7-check review for all diffs
│
├── .agents/
│   └── skills/
│       ├── safe-fintech-change/
│       │   └── SKILL.md              # Skill: inspect → minimal change → risks → tests
│       └── anti-overengineering-review/
│           └── SKILL.md              # Skill: scope → deps → abstractions → safety
│
├── .claude/
│   └── instructions/
│       ├── safe-fintech-change.md    # Claude session instruction — financial code
│       └── anti-overengineering.md  # Claude session instruction — diff review
│
├── docs/
│   ├── business-rules.md             # Canonical business rules — all Rule IDs
│   ├── validation-boundary.md        # Frontend vs backend validation split
│   ├── idempotency-and-concurrency.md # Design: FOR UPDATE, idempotency, 4 layers
│   ├── code-review-task7.md          # Review of flawed AI-generated code
│   ├── test-plan.md                  # Full test plan — all 18 test cases
│   └── ai-commands.md                # 10 team copy-paste AI prompts
│
├── db/
│   └── schema.sql                    # PostgreSQL schema — 6 tables with constraints
│
├── backend/
│   ├── package.json                  # Dependencies: pg, express, helmet, jsonwebtoken
│   ├── jest.config.ts                # Jest config — 80% coverage threshold
│   │
│   ├── src/
│   │   ├── app.ts                    # Express app — helmet, routes, global error handler
│   │   │
│   │   ├── types/
│   │   │   └── index.ts              # Shared TypeScript types — no business logic
│   │   │
│   │   ├── errors/
│   │   │   └── AppError.ts           # AppError class + named error factories per rule
│   │   │
│   │   ├── db/
│   │   │   └── index.ts              # PG pool + withTransaction() atomic wrapper
│   │   │
│   │   ├── auth/
│   │   │   ├── requirePermission.ts  # JWT validation, role check, permission gate
│   │   │   └── requireMerchantScope.ts # Merchant scope enforcement (AZ-004)
│   │   │
│   │   ├── middleware/
│   │   │   └── idempotency.ts        # UUID v4, SHA-256 hash, 24h TTL, response capture
│   │   │
│   │   ├── audit/
│   │   │   └── auditLogger.ts        # Writes inside DB transaction — every attempt
│   │   │
│   │   ├── ledger/
│   │   │   └── ledgerService.ts      # REFUND_REVERSAL entry, negative amount
│   │   │
│   │   ├── refunds/
│   │   │   ├── refund.constants.ts   # Rule-derived constants — currencies, statuses
│   │   │   ├── refund.rules.ts       # Pure assertion functions — one per Rule ID
│   │   │   ├── refundService.ts      # Atomic flow: lock → check → insert → ledger → audit
│   │   │   └── refundController.ts   # HTTP handlers — success and failure audit paths
│   │   │
│   │   ├── transactions/
│   │   │   └── transactionController.ts # GET transaction with masked response
│   │   │
│   │   ├── routes/
│   │   │   └── index.ts              # All routes — 3-layer middleware on every route
│   │   │
│   │   └── utils/
│   │       ├── masking.ts            # UPI, card, bank masking + log stripping
│   │       └── responseFormatter.ts  # Consistent API responses — no stack traces
│   │
│   └── tests/
│       ├── refunds/
│       │   ├── refund.rules.test.ts              # 18 unit tests — all rule assertions
│       │   ├── refund.auth.test.ts               # 12 tests — auth, permission, scope
│       │   ├── refund.business.test.ts           # 14 tests — status, amount, over-refund
│       │   ├── refund.idempotency.test.ts        # 10 tests — replay, conflict, missing key
│       │   ├── refund.concurrency.test.ts        # 12 tests — concurrent, audit, ledger
│       │   └── refundController.integration.test.ts # Integration tests — full stack
│       │
│       └── utils/
│           └── masking.test.ts       # 15 unit tests — all masking rules + edge cases
│
└── frontend/
    └── src/
        ├── types/
        │   └── index.ts              # Frontend types — mirrors backend response shapes
        │
        ├── api/
        │   └── refundApi.ts          # All API calls — JWT in memory, idempotency key
        │
        ├── hooks/
        │   ├── useTransaction.ts     # Fetch + refetch, cancellation-safe, error state
        │   └── useRefund.ts          # State machine: idle → submitting → success/error
        │
        ├── components/
        │   ├── TransactionDetail.tsx # Transaction data, masked ref, eligibility button
        │   ├── RefundModal.tsx       # Amount input, reason, success/error/loading states
        │   ├── RefundHistory.tsx     # Reverse chronological list, empty state
        │   └── ui/
        │       ├── Badge.tsx         # Status badge — driven by type
        │       ├── Spinner.tsx       # Accessible loading with aria-live
        │       └── Alert.tsx        # Error/success/warning with role="alert"
        │
        ├── utils/
        │   └── format.ts             # Paise → rupee, date formatter, badge class
        │
        ├── styles/
        │   └── globals.css           # Design tokens, card, modal, form, masked-value
        │
        └── tests/
            └── TransactionDetail.test.tsx  # 24 frontend tests — RTL + Jest
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend runtime | Node.js + TypeScript |
| Backend framework | Express |
| Database | PostgreSQL |
| Authentication | JWT (jsonwebtoken) |
| Security headers | Helmet |
| Frontend | React + TypeScript |
| Frontend testing | React Testing Library + Jest |
| Backend testing | Jest + Supertest + ts-jest |

---

## Key Architecture Properties

| Property | Implementation |
|---|---|
| Atomic financial writes | `withTransaction()` — refund + ledger + audit in one `BEGIN/COMMIT` |
| Concurrent over-refund prevention | `SELECT ... FOR UPDATE` row lock on the transaction row |
| Duplicate refund prevention | Idempotency middleware — SHA-256 payload hash, 24h TTL |
| Authorization | 3-layer middleware: `authenticate` → `requirePermission` → `requireMerchantScope` |
| Data masking | `maskTransaction()` called before every API response — raw `payment_ref` never leaves backend |
| Audit completeness | Every refund attempt writes to `refund_audit` — success and failure paths |
| Ledger integrity | `REFUND_REVERSAL` entry written inside same transaction as refund |
| SQL safety | Parameterised queries everywhere — no string interpolation |
| Frontend safety | Backend is the only security boundary — frontend validation is UX only |

---

## Running Tests

```bash
# Backend — all tests
cd backend && npm test

# Backend — with coverage report
cd backend && npm run test:coverage

# Backend — single module
cd backend && npm test -- --testPathPattern=refunds

# Frontend — all tests
cd frontend && npm test -- --watchAll=false

# Type check
cd backend && npm run type-check
cd frontend && npm run type-check

# Lint
cd backend && npm run lint
```

---

## Business Rules Reference

All business logic traces to a Rule ID in `docs/business-rules.md`.

| Area | Rule ID | Rule |
|---|---|---|
| Eligibility | RE-001 | Only `SUCCESS` transactions can be refunded |
| Currency | RE-002 | Only `INR` is supported |
| Amount | RA-001 | Refund amount must be > 0 |
| Amount | RA-002 | Cannot exceed original transaction amount |
| Partial refund | RA-003 | Multiple partial refunds are allowed |
| Over-refund | RA-004 | Total refunded cannot exceed original amount |
| Idempotency | ID-001 | Every refund request must include `Idempotency-Key` header |
| Idempotency | ID-003 | Same key + same payload → return original response |
| Idempotency | ID-004 | Same key + different payload → 409 Conflict |
| Authorization | AZ-001 | Only `admin` or `support` roles may access refund endpoints |
| Authorization | AZ-002 | Only users with `refund:create` permission may create refunds |
| Merchant scope | AZ-004 | Admin may only access transactions in their `allowed_merchant_ids` |
| Audit | AU-001 | Every refund attempt must create an audit record |
| Ledger | LE-001 | Successful refund must create a reversal entry in `ledger_entries` |
| Masking | DP-001 | UPI VPA masked: `ab****@okaxis` |
| Masking | DP-003 | Card number masked: `****1111` |
| Frontend | FE-001 | Refund button disabled when `is_refundable: false` |
| Frontend | FE-002 | Eligibility computed by backend only — never frontend |

---

## AI Workflow Quick Start

Before any AI-assisted change on this codebase:

```
1. Load AGENTS.md and docs/business-rules.md into the AI context.
2. Run the Safe Fintech Change skill:
   .agents/skills/safe-fintech-change/SKILL.md
3. After AI generates code, run the Anti-Overengineering Review:
   .agents/skills/anti-overengineering-review/SKILL.md
4. Run git diff and verify every changed file was in the stated scope.
5. Run npm test — all tests must pass before merging.
```

See `docs/ai-commands.md` for 10 ready-to-use team prompts.
