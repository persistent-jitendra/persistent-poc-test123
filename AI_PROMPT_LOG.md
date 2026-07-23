# AI Prompt Log
## FinPay Admin Refund Module — Assessment

---

## Tool Used
**SASVA AI** (AI coding assistant embedded in IDE)

All prompts were issued inside the IDE session. The assistant had access
to the workspace file system and could read, create, and patch files directly.
Every generated file was reviewed before being accepted into the codebase.

---

---

## Prompt 1

**Context:** Starting Task 1 — AI workflow setup files.

**Prompt (summarised):**
```
Create AGENTS.md for the FinPay Admin Refund Module.
It should instruct all AI agents on how to behave safely in a fintech
codebase. Cover: protected files, business rule enforcement, no-guessing
policy, no overengineering, secrets and PII handling, SQL safety,
and a must-not-do checklist.
```

### AI Output Accepted
- The overall structure: sections for project context, absolute rules,
  business rules table, code generation standards, post-generation checklist.
- The `Protected Files` list covering auth, config, ledger, audit,
  migrations, infra.
- The SQL parameterisation example showing safe vs unsafe patterns.
- The financial transaction wrapper pattern using a `withTransaction()` call.
- The must-not-do checklist at the end.

### AI Output Rejected
- An initial draft included a section titled **"Suggested AI Tools"** that
  listed Copilot, Cursor, and ChatGPT with capability comparisons.
  This was rejected — it has no place in an agent instruction file and
  distracts from the actual rules.
- A paragraph recommending **"automatic code formatting on save"** was
  removed. Formatting preferences belong in `.editorconfig`, not in
  agent instruction files.

### Why Rejected
The rejected sections were scope creep. `AGENTS.md` has one job: tell AI
agents what they must and must not do. Any content that does not serve
that purpose dilutes the file and makes it less likely to be read fully.

### Manual Verification
- Read every rule and mapped it to a concrete scenario: "what happens if
  an agent violates this?"
- Confirmed the business rules table matches the assessment specification
  exactly — compared field by field.
- Confirmed the protected files list matches the actual directory structure
  of the codebase.
- Verified there are no contradictions between AGENTS.md and CLAUDE.md.

### Final Assumptions
- Assumed agents read `AGENTS.md` before any other file. In practice this
  requires a project-level prompt or system prompt that loads this file
  first. Documented this in `AI_WORKFLOW.md` Section 1.

---

---

## Prompt 2

**Context:** Task 2 — Database schema.

**Prompt (summarised):**
```
Create db/schema.sql for the FinPay refund module.
Tables needed: transactions, refunds, idempotency_keys, refund_audit,
ledger_entries, admin_users.
Follow docs/business-rules.md strictly.
Mark audit and ledger tables as append-only.
Use appropriate constraints and indexes.
Do not add tables or columns that are not needed for this module.
```

### AI Output Accepted
- The six-table structure with correct foreign keys and `CHECK` constraints.
- `BIGINT` for all amount columns (stores paise — avoids floating point
  precision issues in financial calculations).
- PostgreSQL `RULE` statements on `refund_audit` and `ledger_entries`
  to enforce append-only behaviour at the database level:
  ```sql
  CREATE RULE no_update_refund_audit AS ON UPDATE TO refund_audit DO INSTEAD NOTHING;
  CREATE RULE no_delete_refund_audit AS ON DELETE TO refund_audit DO INSTEAD NOTHING;
  ```
- `idempotency_keys.expires_at` defaulting to `NOW() + INTERVAL '24 hours'`
  (Rule ID-005).
- Indexes on `merchant_id`, `transaction_id`, `created_at DESC` for
  expected query patterns.

### AI Output Rejected
- AI initially added a `refund_policies` table with columns for
  `max_refund_window_days`, `auto_approval_threshold`, and
  `requires_manager_approval`. **Rejected entirely.**
- AI added a `notifications` table for "refund status email alerts."
  **Rejected entirely.**
- AI added a `soft_deleted_at` column to the `refunds` table.
  **Rejected.**

### Why Rejected
**`refund_policies` table:** No policy engine or approval workflow exists
in the business rules. This is a classic AI overengineering pattern —
inventing a future requirement ("what if you need configurable rules?")
and building infrastructure for it now. The rule is: build what the
business rules say. Nothing more.

**`notifications` table:** Email notifications were never mentioned in
the requirements. Adding a table for them implies a notification service,
an email provider integration, and retry logic — none of which were asked
for. Rejected under the minimal change principle.

**`soft_deleted_at` on refunds:** Refunds are financial records. They
must never be deleted — hard or soft. Adding a soft-delete column implies
a code path where a refund can be "deactivated," which contradicts the
immutability requirement for financial records. The correct approach is
the append-only audit trail, which is already implemented.

### Manual Verification
- Mapped every column in every table to a specific rule ID in
  `docs/business-rules.md`. Any column that could not be mapped was removed.
- Verified that `CHECK` constraints on `status` fields match the exact
  allowed values in the type definitions (`src/types/index.ts`).
- Confirmed `BIGINT` is correct for INR paise: ₹99,99,999 = 9,999,999,00
  paise, well within BIGINT range.
- Verified the `FOR UPDATE` lock would work correctly with the schema —
  the lock is on the `transactions` row, which is the single serialisation
  point for concurrent refunds on the same transaction.

### Final Assumptions
- **[ASSUMPTION]** Migrations are managed separately. The schema file
  represents the target state. A migration tool (e.g., `db-migrate`,
  `flyway`) would apply it incrementally in production. The schema itself
  is not run directly against production.
- **[ASSUMPTION]** `payment_ref` stores the raw UPI VPA / card number /
  account number. This column is never returned in API responses — only
  the masked version computed at query time.

---

---

## Prompt 3

**Context:** Task 2 — Idempotency middleware.

**Prompt (summarised):**
```
Create src/middleware/idempotency.ts.
It must implement Rules ID-001 through ID-005 from docs/business-rules.md.
Use the existing pool from src/db/index.ts.
Use SHA-256 to hash the request body.
Store the key only on 2xx responses.
Do not add Redis. Do not add a separate caching layer.
```

### AI Output Accepted
- The `hashPayload()` helper using Node's built-in `crypto` module — no
  external dependency.
- The lookup query with `expires_at > NOW()` filter so expired keys are
  automatically ignored without a separate cleanup step at read time.
- The `res.json()` wrap pattern to intercept and store the response after
  the controller writes it.
- `ON CONFLICT (key) DO NOTHING` as a safety net against concurrent
  insertions of the same new key.
- Checking `res.statusCode >= 200 && res.statusCode < 300` before storing,
  so failed requests (422, 403, 404) do not consume an idempotency slot.

### AI Output Rejected
- AI suggested adding **Redis** as the idempotency store with a fallback
  to PostgreSQL. Prompt explicitly said no Redis — rejected.
- AI added an **in-memory `Map`** as a "fast path" cache in front of the
  database lookup. Rejected.
- AI suggested wrapping the idempotency key insertion in the **same
  database transaction** as the refund. Rejected — see why below.

### Why Rejected
**Redis:** Adding Redis introduces an external dependency, a new failure
mode (Redis is down), and a deployment concern. For admin-initiated
refunds (low volume, not microsecond-latency-sensitive), the PostgreSQL
table is sufficient and keeps the failure surface small.

**In-memory Map:** A per-process in-memory cache does not work correctly
in a multi-instance deployment. Two requests could arrive at different
instances, both miss the in-memory cache, and both proceed. This is a
correctness bug, not a performance trade-off. Rejected completely.

**Idempotency insert inside the main transaction:** If the main transaction
rolls back (e.g., over-refund), the idempotency key insertion would also
roll back. This means a failed request would not consume an idempotency
slot — which is correct. However, if the key was stored inside the
transaction, a success followed by a network drop before the key could be
committed (if somehow the commit failed) could leave a partially stored
state. The chosen approach — wrapping `res.json()` after the transaction
commits — is simpler and avoids this edge case.

**[RISK flagged]:** The `res.json()` wrap means the key is stored
asynchronously after the response is sent. In the event of a crash
between response send and key storage, the key would not be stored, and
the next retry would be treated as a new request. This is an accepted
trade-off: the alternative (storing before responding) adds latency and
requires the key store to be part of the transaction. For this module,
the idempotency window (24 hours) and the over-refund database lock
together provide sufficient protection even if the async storage
occasionally fails.

### Manual Verification
- Traced through the three paths manually: new key, same key + same
  payload, same key + different payload.
- Verified `hashPayload` uses `JSON.stringify(body)` — confirmed `body`
  is the parsed Express request body object, so the hash is of the
  structured payload, not the raw HTTP body string. This means key
  equality is sensitive to field ordering in the client's JSON
  serialisation. Documented as an assumption.
- Verified the UUID v4 regex against the RFC 4122 spec — the pattern
  correctly requires `4` in the third group and `[89ab]` in the fourth
  group.

### Final Assumptions
- **[ASSUMPTION]** The client (frontend `useRefund` hook) always sends
  the same JSON field order on retry because it uses `JSON.stringify` on
  a consistent TypeScript object. If a different client (e.g., curl)
  sends fields in a different order, the hash will differ and a 409 will
  be incorrectly returned. This is a known trade-off of hashing the
  serialised body rather than a canonical form.
- **[ASSUMPTION]** Idempotency key expiry cleanup is handled by a
  scheduled job outside this module. The middleware only enforces TTL
  at read time via `expires_at > NOW()`.

---

---

## Prompt 4

**Context:** Task 2 — Refund service, specifically the over-refund guard
and concurrency control.

**Prompt (summarised):**
```
Create src/refunds/refundService.ts.
The createRefund function must:
  - Use FOR UPDATE on the transaction row inside a DB transaction.
  - Fetch existing refunds after acquiring the lock.
  - Call assertRefundEligibility() before any write.
  - Write refund + ledger + audit atomically.
  - Write a failed audit record outside the transaction on error.
Do not add optimistic locking. Do not add a distributed lock.
Do not add retry logic.
```

### AI Output Accepted
- The `SELECT * FROM transactions WHERE id = $1 FOR UPDATE` pattern —
  correctly placed as the first query inside `withTransaction()` so the
  lock is held for the full duration of the business logic.
- Fetching `existingRefunds` after the lock (not before) so the total
  is computed on committed, locked data.
- The `recordFailedAttempt()` function using a fresh `pool.connect()`
  (not the rolled-back transaction client) to write the failure audit
  record. This was a critical correctness detail.
- `generateRefundId()` using `crypto.randomUUID()` — no external UUID
  library needed.

### AI Output Rejected
- AI added **exponential backoff retry** inside `createRefund()`:
  if the INSERT failed due to a lock timeout, it would retry up to 3
  times with 100ms / 200ms / 400ms delays. **Rejected entirely.**
- AI added a `version` column check (optimistic locking):
  ```sql
  UPDATE transactions SET version = version + 1
  WHERE id = $1 AND version = $2
  ```
  **Rejected.**

### Why Rejected
**Retry logic with backoff:** In a financial system, automatic retries on
a failed write are dangerous. If the retry logic has a bug, or if the
first request actually succeeded but returned an error (network issue
during response), the retry creates a duplicate. The correct pattern is:
let the request fail, let the client retry with the same idempotency key,
and let the idempotency layer handle deduplication. Never retry inside
the service layer for money movements.

**Optimistic locking:** Optimistic locking requires the caller to detect
the conflict (version mismatch → 409) and retry. In a financial context,
this retry must be controlled by the client with an idempotency key — the
same requirement as a clean failure. Optimistic locking adds a column,
a check, and a retry contract without providing more safety than `FOR UPDATE`
in this scenario. `FOR UPDATE` is simpler, already built into PostgreSQL,
and does not require any retry at the application layer.

### Manual Verification
- Traced the concurrent scenario manually:
  - Request A acquires lock, reads ₹0 refunded, inserts ₹3,000.
  - Request B blocks at `SELECT … FOR UPDATE`.
  - Request A commits. Request B unblocks, reads ₹3,000 refunded.
  - ₹3,000 + ₹3,000 = ₹6,000 > ₹5,000 → `OVER_REFUND` thrown.
  - Request B rolls back. Failure audit written.
- Confirmed `recordFailedAttempt` uses `pool.connect()` not the rolled-back
  client — otherwise the audit write would also be rolled back.
- Verified `filter(r => r.status === 'SUCCESS')` in `assertNoOverRefund`
  correctly excludes FAILED and PENDING refunds from the total.

### Final Assumptions
- **[ASSUMPTION]** PostgreSQL `lock_timeout` is configured at the server
  level. If a lock is held for longer than the timeout (e.g., a request
  hangs), the waiting request will fail with a lock timeout error rather
  than waiting indefinitely. This prevents lock queue build-up under load.
  The actual timeout value is an infrastructure concern outside this module.
- **[ASSUMPTION]** The `withTransaction()` wrapper uses the default
  PostgreSQL isolation level (`READ COMMITTED`). This is sufficient because
  `FOR UPDATE` provides the serialisation guarantee we need on the specific
  row. A higher isolation level (`SERIALIZABLE`) is not needed and would
  increase lock contention.

---

---

## Prompt 5

**Context:** Task 3 — `useRefund` hook and double-submit prevention.

**Prompt (summarised):**
```
Create src/hooks/useRefund.ts.
It must:
  - Generate one idempotency key per modal open (useRef, not useState).
  - Manage a state machine: idle → submitting → success | error.
  - Block concurrent submissions while status === 'submitting'.
  - Rotate the idempotency key only on explicit reset().
  - Not add debounce, throttle, or any timing-based prevention.
```

### AI Output Accepted
- The `useRef` for the idempotency key — correct because a `useState`
  would cause a re-render every time the key was read, and the key must
  not change between renders of the same modal instance.
- The `if (state.status === 'submitting') return;` guard as the primary
  double-submit prevention — clean and explicit.
- The `reset()` function rotating the key with a new `uuidv4()` — ensures
  that after a successful refund and modal reopen, the next submission
  gets a fresh key.
- The discriminated union type for `RefundState` — makes impossible states
  unrepresentable (e.g., cannot be both `success` and have an `error`).

### AI Output Rejected
- AI added a **`useCallback` with `debounce(500)`** wrapping the submit
  function. Rejected.
- AI added a **`useRef` boolean flag** called `isSubmittingRef` in addition
  to the state machine, "for synchronous double-submit prevention."
  Rejected.

### Why Rejected
**Debounce:** Debounce is a timing-based mechanism. It prevents submission
if the function is called again within 500ms. This means: if a network
request takes 600ms and the user clicks again at 501ms, the second click
goes through — the debounce window has already closed. For financial
operations, the double-submit guard must be state-based (is a request
currently in-flight?), not timing-based. The `submitting` state handles
this correctly regardless of how long the request takes.

**Dual `isSubmittingRef` flag:** The state machine already encodes the
submitting state. Adding a second, separate boolean creates two sources of
truth that can drift apart. If `state.status` is `'submitting'` but
`isSubmittingRef.current` is `false` (due to a bug), the guard fails. One
source of truth is simpler and safer.

### Manual Verification
- Verified `useRef` is initialised with `uuidv4()` on hook mount —
  confirmed this means one key per `useRefund()` call, which happens once
  per modal open.
- Verified the state machine transitions: `idle → submitting` on submit
  call, `submitting → success` on clean response, `submitting → error` on
  API error or network failure.
- Confirmed `reset()` generates a new key — so a user who closes and
  reopens the modal to try a different amount gets a fresh key, preventing
  an accidental idempotency match with the previous successful refund.
- Confirmed the `disabled={isSubmitting}` and `aria-busy={isSubmitting}`
  attributes on the submit button are driven by the same state, ensuring
  the UI and the hook are always in sync.

### Final Assumptions
- **[ASSUMPTION]** The `uuid` npm package is already a project dependency
  (used in the API layer). No new dependency was added.
- **[ASSUMPTION]** The modal is unmounted when closed. This means the hook
  is destroyed and the idempotency key ref is garbage-collected. A new
  modal open creates a new hook instance with a new key. If the modal were
  kept mounted (hidden via CSS), the same key would persist — which is
  correct behaviour but worth noting.

---

---

## Prompt 6

**Context:** Task 3 — Validation boundary document.

**Prompt (summarised):**
```
Create docs/validation-boundary.md.
Define exactly which validations are frontend-only (UX) and which are
backend-enforced (security boundary).
Include a worked example for the refund button, amount validation,
double-submit, sensitive data, and permission rendering.
Explain what goes wrong if the boundary is violated.
```

### AI Output Accepted
- The responsibility matrix table — clear, scannable, references rule IDs.
- The "threat model" section explaining why frontend validation can always
  be bypassed.
- The concurrent partial refund example in the amount validation section —
  this is the key insight that justifies running the backend check inside
  the database transaction even when the frontend already caps the input.
- The "What Can Go Wrong" table at the end — good for code review use.

### AI Output Rejected
- AI added a section titled **"Recommended Frontend Validation Libraries"**
  listing `zod`, `yup`, and `react-hook-form`. **Rejected entirely.**
- AI added a section on **"Server-Side Rendering considerations"** for
  Next.js. Rejected — the project uses React (CRA/Vite), not Next.js.

### Why Rejected
**Library recommendations:** The document's purpose is to define the
validation boundary. Recommending libraries is out of scope and violates
the no-unrequested-dependencies principle from AGENTS.md. If a library
is needed, it goes through a separate package approval process.

**SSR section:** The project uses React, not Next.js. Generating content
for a framework that is not in use is a classic AI pattern of
"future-proofing" — rejected under the minimal change principle.

### Manual Verification
- Read each row in the responsibility matrix and asked: "Can a malicious
  user bypass this frontend check?" For every row where the answer is yes,
  confirmed the backend enforces the same rule independently.
- Verified the concurrent refund example is consistent with the
  `FOR UPDATE` lock implementation in `refundService.ts`.
- Confirmed the JWT storage note matches the `refundApi.ts` implementation
  (token in memory, not `localStorage`).

### Final Assumptions
- **[ASSUMPTION]** The auth token is passed into the frontend app at login
  time and stored in a React context or module-level variable. The
  mechanism for the initial login is outside this module's scope.

---

---

## Cross-Cutting Observations

### What AI Did Well
- Consistently used `FOR UPDATE` without being prompted a second time once
  the pattern was established.
- Correctly identified that failed-attempt audit records must be written
  outside the rolled-back transaction — this is a subtle correctness
  requirement that was not stated explicitly in the prompt.
- Generated tests that covered error paths, not just happy paths.
- Used native Node.js `crypto` for hashing rather than adding an npm package.
- Applied the three-middleware pattern (authenticate → permission →
  merchant scope) consistently on every route without needing correction.

### What AI Got Wrong (Patterns)
| Pattern | Frequency | Action Taken |
|---|---|---|
| Suggested Redis for caching/idempotency | 3 times | Rejected each time, rule added to AGENTS.md |
| Added unrequested tables to schema | 2 times | Rejected — mapped every column to a rule ID |
| Added retry logic in financial service | 1 time | Rejected — documented why in prompt log |
| Added debounce to hook | 1 time | Rejected — replaced with state-based guard |
| Recommended libraries not in scope | 2 times | Rejected — referenced no-new-deps rule |
| Added multi-currency support | 1 time | Rejected — only INR is in scope per RE-002 |

### Verification Approach Used Throughout
1. **Rule mapping:** Every piece of business logic was traced to a rule ID
   in `docs/business-rules.md`. Code without a traceable rule was removed.
2. **Diff review:** Every file was read in full before being accepted.
   No file was accepted based on a summary.
3. **Security checklist:** For every new route and service function, the
   checklist from `.cursor/rules/fintech-development.mdc` was applied:
   parameterised SQL, auth middleware, DB transaction, audit write, masking.
4. **Threat model:** For every validation, asked "what happens if an
   attacker skips the frontend?" to confirm the backend enforces the rule.
5. **State tracing:** For concurrent scenarios, traced the timeline
   manually (as shown in `docs/idempotency-and-concurrency.md`) to confirm
   the lock and over-refund guard interact correctly.
