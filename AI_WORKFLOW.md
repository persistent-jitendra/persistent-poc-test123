# AI_WORKFLOW.md — FinPay Admin Refund Module
## Safe AI Development Workflow

---

## Overview

This document is the master guide for how AI tools are used safely on this
project. It covers every phase of development: context setup, prompting,
code review, testing, and what AI is never allowed to do.

Every developer on this project must read this before using any AI tool.

---

## 1. How We Give Project Context to AI

AI tools produce safe output only when given accurate, bounded context.
We never let AI assume the shape of this codebase.

### 1.1 Always Load These Files First

Before starting any AI-assisted task, ensure the AI has read:

```
AGENTS.md               ← hard rules and protected files
CLAUDE.md               ← project summary and architecture patterns
docs/business-rules.md  ← source of truth for all business logic
docs/api/<endpoint>.md  ← API contract for the area being changed
src/<module>/           ← the actual code being modified
tests/<module>/         ← existing tests for that module
```

### 1.2 Context Prompt Template

Use this template to open every AI session:

```
You are working on FinPay, a production fintech platform.
Read AGENTS.md and CLAUDE.md before doing anything else.
The task is: [specific task description].
The files involved are: [list files].
Do not modify any file outside this list.
Do not add new dependencies.
Ask me before implementing any business rule that is not in docs/business-rules.md.
```

### 1.3 Scope Bounding

Every prompt must state:
- **What to change** — specific files or functions
- **What not to change** — explicitly name out-of-scope files
- **What the output should look like** — endpoint, function, type, test

Never use open-ended prompts like *"build the refund feature"* without
bounding them first.

---

## 2. How We Stop AI From Guessing Business Rules

### The Rule
> If a business rule is not written down, AI must not implement it.

### The Process

```
Step 1 — Before any AI prompt that involves business logic:
          Open docs/business-rules.md.
          Find the exact rule that applies.
          Paste it into the prompt verbatim.

Step 2 — In the prompt, add:
          "Only implement what is stated in these rules.
           If anything is ambiguous, ask me — do not guess."

Step 3 — After AI generates code:
          Read every conditional, every validation, every error message.
          Verify each one maps to a documented rule.
          Delete or correct anything that does not.
```

### Business Rules Are Located Here — Nowhere Else

```
docs/business-rules.md          ← canonical source
src/refunds/refund.rules.ts     ← TypeScript constants derived from docs
src/refunds/refund.constants.ts ← status codes, limits, currency list
```

AI must read these files. AI must not invent rules from:
- Variable names
- Database column names
- Similar systems it was trained on
- "Common sense" about how refunds work

---

## 3. How We Stop AI From Overengineering

### The Principle
> Solve the stated problem. Not a future version of it.

### Guardrails We Apply

**In the prompt:**
```
Use the smallest change that satisfies this requirement.
Do not add new services, queues, caches, or packages.
Do not refactor code outside the files listed.
Do not add abstractions that do not already exist in this codebase.
```

**After code is generated — run the Anti-Overengineering Review:**
See `.agents/skills/anti-overengineering-review/SKILL.md`

The review checks for:
- New npm packages that were not requested
- New infrastructure (Redis, Bull, Kafka) that was not requested
- New abstraction layers (Repository, Factory, Strategy) not present elsewhere
- Code volume exceeding what the task requires
- Justifications that use words like "scalability", "extensibility",
  "future-proofing" — these are rejected unless the future is now

**Hard rejections — never accept AI output that introduces:**
- A message queue for a synchronous operation
- A distributed cache to replace a simple DB lookup
- A saga or CQRS pattern for a basic CRUD endpoint
- Multi-currency support when only INR is in scope
- Generic base classes with a single implementation

---

## 4. How We Prevent AI From Touching Risky Files

### 4.1 File-Level Protection

The following files are protected by `.cursorignore` and listed explicitly
in `AGENTS.md`. AI must not read or modify them:

```
Category              Files
──────────────────    ──────────────────────────────────────────
Secrets               .env, .env.*, *.pem, *.key, secrets/
Auth & Permissions    src/auth/**, src/middleware/auth.ts
Financial Core        src/ledger/**, src/audit/**
DB Migrations         src/migrations/**
App Config            src/config/secrets.ts, src/config/database.ts
Infrastructure        infrastructure/**, terraform/**, .github/workflows/**
Deployment Scripts    scripts/deploy.sh, scripts/migrate-production.sh
```

### 4.2 Prompt-Level Protection

Every prompt for a bounded task must include:

```
Do not modify any file outside: [explicit list].
Do not create migration files.
Do not modify auth middleware.
Do not modify ledger or audit services.
```

### 4.3 Review-Level Protection

Before accepting any AI-generated change:
- Run `git diff --stat` to see every file touched.
- If any protected file appears in the diff → reject the entire change
  and re-prompt with tighter scope.

```bash
# Run after every AI-assisted change
git diff --stat
git diff src/        # review every line before accepting
```

---

## 5. How We Handle Secrets, PII, Payment Data, Tokens

### 5.1 What Must Never Appear in AI Context

Do not paste the following into any AI prompt, chat, or code suggestion:

```
❌ Real .env values (database URLs, API keys, JWT secrets)
❌ Real JWT tokens from any environment
❌ Real user data (names, emails, phone numbers)
❌ Real transaction IDs from production
❌ Real card numbers, UPI VPAs, bank account numbers
❌ Real merchant credentials
```

### 5.2 Use Fake Data in Prompts and Examples

When you need to show AI a data shape, use clearly fake values:

```json
{
  "transaction_id": "TXN-EXAMPLE-001",
  "user_id": "USER-EXAMPLE-001",
  "merchant_id": "MER-EXAMPLE-001",
  "amount": 1000,
  "currency": "INR"
}
```

### 5.3 Masking in Code

All API responses that include payment data must pass through:
```typescript
import { maskSensitiveData } from 'src/utils/masking';
return res.json(maskSensitiveData(data));
```

AI must use this utility — it must never implement its own masking logic.

### 5.4 Environment Variable Rules

- AI may reference `process.env.VARIABLE_NAME` in code.
- AI must never suggest hardcoding a value that belongs in an env variable.
- AI must never print, log, or return env variable values.
- New env variables introduced by AI must be documented in `.env.example`
  with a placeholder value only.

---

## 6. How We Verify AI-Generated Code

### 6.1 The Review Process (Non-Negotiable)

No AI-generated code merges without completing these steps:

```
Step 1 — Diff Review
  git diff
  Read every changed line. Not a summary — every line.
  Reject anything you do not fully understand.

Step 2 — Business Rule Verification
  For every conditional in the generated code:
    → Find the corresponding rule in docs/business-rules.md.
    → If no rule exists for it → delete it and ask the developer.

Step 3 — Security Check
  Run the security checklist from AGENTS.md Section 5.
  Check for: SQL injection, missing auth, unmasked PII, missing transactions.

Step 4 — Run Existing Tests
  npm test
  All pre-existing tests must pass. No regressions allowed.

Step 5 — Write and Run New Tests
  Write tests for every new code path before merging.
  See Section 7 for test requirements.

Step 6 — Final Scope Check
  Count the files changed.
  Verify every changed file was in the stated task scope.
  Reject changes to out-of-scope files.
```

### 6.2 Commands to Run

```bash
# 1. See what AI changed
git diff --stat
git diff

# 2. Run all tests
npm test

# 3. Run only the module under change
npm test -- --testPathPattern=refunds

# 4. Check for accidental secrets in diff
git diff | grep -iE "(password|secret|token|api_key|private)" 

# 5. Check for SQL injection patterns
git diff | grep -E "(\`|')\s*\+\s*(req\.|params\.|body\.|query\.)"

# 6. Lint check
npm run lint

# 7. Type check
npm run type-check
```

---

## 7. Testing Requirements for AI-Generated Code

### 7.1 What Must Be Tested

Every AI-generated refund feature must have tests covering:

```
HAPPY PATHS
  ✓ Full refund on a SUCCESS transaction
  ✓ Partial refund on a SUCCESS transaction
  ✓ Multiple partial refunds that sum to the original amount exactly
  ✓ Idempotent request (same key + same payload) returns 200 + original response

ERROR PATHS
  ✓ Refund on a non-SUCCESS transaction → 422 REFUND_INELIGIBLE
  ✓ Refund amount = 0 → 400 INVALID_AMOUNT
  ✓ Refund amount > original transaction amount → 422 AMOUNT_EXCEEDS_LIMIT
  ✓ Total refunded + new amount > original amount → 422 OVER_REFUND
  ✓ Currency other than INR → 422 UNSUPPORTED_CURRENCY
  ✓ Missing Idempotency-Key header → 400 MISSING_IDEMPOTENCY_KEY
  ✓ Idempotency conflict (same key, different payload) → 409 CONFLICT
  ✓ Caller missing refund:create permission → 403 FORBIDDEN
  ✓ Caller accessing out-of-scope merchant → 403 FORBIDDEN
  ✓ Transaction not found → 404 NOT_FOUND
  ✓ DB failure mid-transaction → full rollback, no partial writes

AUDIT & LEDGER
  ✓ Successful refund creates a ledger reversal entry
  ✓ Failed refund attempt still creates an audit record
  ✓ Audit record contains: actorId, transactionId, refundId, amount, outcome
```

### 7.2 Test File Naming Convention

```
src/refunds/refundService.ts         → tests/refunds/refundService.test.ts
src/refunds/refundController.ts      → tests/refunds/refundController.test.ts
src/refunds/refundEligibility.ts     → tests/refunds/refundEligibility.test.ts
```

### 7.3 What AI Can Help With

```
✅ Generating test scaffolding (describe/it blocks)
✅ Generating mock data fixtures
✅ Generating edge case test inputs

❌ AI must not decide which cases to skip
❌ AI must not mark tests as .skip without explicit instruction
❌ AI must not reduce assertion specificity to make tests pass
```

---

## 8. Repeated Commands and Rules Reference

### 8.1 Skills Available

| Skill | Location | Use When |
|---|---|---|
| Safe Fintech Change | `.agents/skills/safe-fintech-change/SKILL.md` | Before any financial code change |
| Anti-Overengineering Review | `.agents/skills/anti-overengineering-review/SKILL.md` | After AI generates code |

### 8.2 Standard AI Prompt Starters

**For a new feature:**
```
Read AGENTS.md, CLAUDE.md, and docs/business-rules.md first.
Then read [target files].
Task: [specific description].
Scope: only modify [file list].
Do not add dependencies. Do not touch protected files.
Ask before implementing any rule not in docs/business-rules.md.
```

**For a bug fix:**
```
Read [affected file] and its test file.
The bug is: [description].
Do not change anything outside this function.
Show me the diff before applying it.
```

**For a code review:**
```
Review this diff for: SQL injection, missing auth middleware,
unmasked PII, missing DB transactions, and business rule violations.
Reference docs/business-rules.md for rule verification.
List every issue found with line numbers.
```

### 8.3 Git Hooks (Recommended)

Add to `.git/hooks/pre-commit` to catch common issues before commit:

```bash
#!/bin/sh
# Block commits that include obvious secrets
if git diff --cached | grep -qiE "(password\s*=\s*['\"][^'\"]+['\"]|secret\s*=\s*['\"][^'\"]+['\"]|api_key\s*=\s*['\"][^'\"]+['\"])"; then
  echo "ERROR: Possible secret detected in diff. Review before committing."
  exit 1
fi

# Block SQL string concatenation patterns
if git diff --cached | grep -qE "query\(\`[^\`]*\$\{"; then
  echo "ERROR: Possible SQL injection pattern detected. Use parameterized queries."
  exit 1
fi

# Run tests
npm test --silent
if [ $? -ne 0 ]; then
  echo "ERROR: Tests failed. Fix before committing."
  exit 1
fi
```

---

## 9. Summary — What AI Can and Cannot Do on This Project

### ✅ AI Can Help With
- Reading and explaining existing code
- Generating boilerplate that follows established patterns
- Writing test cases based on documented rules
- Suggesting parameterized SQL queries
- Generating TypeScript types and interfaces
- Generating error messages that match the existing `AppError` pattern
- Drafting documentation

### ❌ AI Cannot Do Autonomously
- Decide business rules or eligibility logic
- Modify protected files (auth, ledger, audit, migrations, secrets, infra)
- Add new npm packages or infrastructure
- Run database migrations
- Deploy to any environment
- Access or reference real user data, real tokens, or production config
- Skip idempotency, audit logging, or DB transactions "for now"
- Reduce test coverage to meet a deadline
