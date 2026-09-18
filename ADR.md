# ARCHITECTURE DECISION RECORDS

Format: Context → Decision → Alternatives → Consequences.
Status values: Accepted · Superseded · Deprecated.

---

## ADR-001 — Modular Monolith over Microservices

**Status:** Accepted

**Context.** The MVP serves one company, 5–10 users, 3–10 active projects, and 100–500 transactions per month, growing to ~50 users and ~5,000 transactions in 2–3 years. The team is small and the domain modules (projects, finance, approvals) share transactional invariants — approving a budget change must atomically update a project.

**Decision.** A single deployable Next.js application organized into business modules with enforced boundaries (`src/modules/*`, public APIs, ESLint import rules).

**Alternatives considered.**
- *Microservices:* would require distributed transactions or sagas for "approve budget change → update project budget", plus service discovery, per-service CI/CD, and distributed tracing. The operational cost dwarfs the benefit at this scale.
- *Unstructured monolith:* cheaper now, but cross-module coupling would make the Phase 2 additions (inventory, suppliers, payroll) expensive.

**Consequences.**
- ✅ ACID guarantees for every cross-module invariant via one `prisma.$transaction`.
- ✅ One deployment, one log stream, one database to operate.
- ✅ Module boundaries act as extraction seams if a service is ever justified.
- ⚠️ Boundaries are conventions and must be mechanically enforced — mitigated by `eslint-plugin-boundaries` in CI.
- ⚠️ All modules scale together; acceptable at this load profile.

---

## ADR-002 — Better Auth over NextAuth/Auth.js

**Status:** Accepted

**Context.** We need email/password authentication, database sessions, a first-class Prisma schema, role data on the session, session invalidation on deactivation or role change, and full TypeScript inference.

**Decision.** Better Auth with the Prisma adapter.

**Alternatives considered.**
- *NextAuth/Auth.js v5:* strong OAuth story, but credentials-based flows are second-class, the database session model is less explicit, and type inference around custom session fields is weaker.
- *Clerk / Auth0:* excellent DX but adds an external dependency, per-user cost, and stores user identity outside our database — awkward for a self-hostable single-company ERP and for joining `User` to `Employee`.
- *Hand-rolled auth:* re-implementing password hashing, session rotation, and rate limiting is exactly the kind of security work that should not be bespoke.

**Consequences.**
- ✅ `Session`, `Account`, `Verification` live in our Postgres, so `User.role`, `User.isActive`, and `User.employeeId` join naturally.
- ✅ Revoking sessions on deactivation or role change is a database operation we control (RBAC §4 special rules).
- ✅ Plugin path available for Phase 2 2FA without re-platforming.
- ⚠️ Smaller ecosystem than NextAuth; mitigated by isolating all auth code behind `modules/iam` so the library is replaceable.

---

## ADR-003 — Server Actions as the primary API, Route Handlers by exception

**Status:** Accepted

**Context.** A browser-only client written in the same TypeScript codebase as the server. The PRD requires Zod validation on both client and server (§6.1) without duplicating contracts.

**Decision.** All business mutations and queries go through Server Actions returning `Result<T>`. Route Handlers are used only for Better Auth, binary download, blob upload tokens, streamed exports, and health.

**Alternatives considered.**
- *REST API + fetch client:* a parallel surface with hand-written types, manual serialization, and duplicated validation — more code for no consumer that needs it.
- *tRPC:* excellent type safety, but largely redundant with Server Actions in the App Router and adds a router layer.
- *GraphQL:* solves over-fetching for many heterogeneous clients; we have exactly one client. YAGNI.

**Consequences.**
- ✅ One Zod schema powers both the React Hook Form resolver and the server parse.
- ✅ No client/server type drift; no API versioning burden for a single client.
- ✅ Progressive enhancement and automatic revalidation come for free.
- ⚠️ Server Actions are POST-only and unsuitable for streaming — hence the documented Route Handler exceptions.
- ⚠️ Every action is a public endpoint: authorization inside the action is mandatory (`API-ACTIONS.md` §18 checklist).

---

## ADR-004 — Vercel Blob over S3/R2 for MVP file storage

**Status:** Accepted

**Context.** MVP file needs are contracts, receipts, site photos, and employee documents — an estimated few GB over the first year, with all deployment on Vercel.

**Decision.** Vercel Blob with client-side direct upload via server-issued scoped tokens. All downloads are proxied through an authorized Route Handler.

**Alternatives considered.**
- *AWS S3:* the mature, cheapest-at-scale option, but requires IAM policy management, CORS configuration, SDK wiring, and a separate account for a small volume.
- *Cloudflare R2:* no egress fees and S3-compatible; still a second vendor and dashboard for the MVP.
- *Local filesystem / `public/uploads`:* incompatible with serverless (ephemeral disk) and would expose files without authorization. Rejected outright.

**Consequences.**
- ✅ Zero extra vendor setup; one dashboard, one billing relationship.
- ✅ Direct client upload keeps large files out of serverless function memory.
- ⚠️ More expensive per GB at scale than R2/S3.
- ✅ Mitigated by design: all storage access is behind `modules/documents`'s `BlobStorage` port, so a Phase 2 migration to S3/R2 changes one infrastructure file plus a data backfill — no call-site changes.

---

## ADR-005 — Soft delete for business entities; append-only audit and approvals

**Status:** Accepted

**Context.** The PRD requires data retention, complete approval history, and full auditability (§5.9, §6.1). Construction companies must be able to explain historical financial records, and accidental deletion of a project or expense must be recoverable.

**Decision.** Every business table carries `deletedAt`. Application deletes set the timestamp; physical deletion never happens through application flows. `AuditLog` and `ApprovalRequest` are append-only and are never soft-deleted. All read paths filter `deletedAt: null` through a shared query helper.

**Alternatives considered.**
- *Hard delete:* simplest, but destroys the audit trail and breaks foreign keys from financial records. Directly contradicts the PRD.
- *Archive tables:* preserves history but doubles the schema and complicates queries and restores.
- *Temporal/bitemporal tables:* full history for every column, but heavy for an MVP; the audit log's `changes` JSON already answers "what changed and who changed it".

**Consequences.**
- ✅ Recoverable mistakes; intact financial lineage; `ON DELETE RESTRICT` acts as a safety net against maintenance scripts.
- ⚠️ Every query must filter soft-deleted rows — a forgotten filter is a data-correctness bug. Mitigated by a single shared helper plus integration tests asserting deleted rows never appear.
- ⚠️ Unique constraints must ignore deleted rows — implemented as partial unique indexes (`WHERE deletedAt IS NULL`) so a deleted record never blocks a legitimate new one.
- ⚠️ Tables grow indefinitely; negligible at this volume, and a purge policy can be added later.

---

## ADR-006 — `Decimal(14,2)` for money; decimal strings across the wire

**Status:** Accepted

**Context.** This is a financial system. Profitability, budget variance, and cash flow must be exact. JavaScript numbers are IEEE-754 doubles: `0.1 + 0.2 !== 0.3`, and cumulative error across thousands of transactions is unacceptable in an ERP.

**Decision.** PostgreSQL `NUMERIC(14,2)` for all monetary columns, mapped to Prisma `Decimal`. Aggregation happens in SQL. Money crosses the Server Action boundary as a **decimal string**, never a JS `number`. Formatting for display happens only at the edge via `Intl.NumberFormat`.

**Alternatives considered.**
- *Integer minor units (piastres):* also exact and a legitimate choice, but every read and write needs ×100 / ÷100 conversion, and raw SQL reports become harder to read. `NUMERIC` gives exactness without conversion ceremony.
- *Float/double:* unacceptable for financial data.
- *Decimal(19,4):* more precision than EGP transactions need; 2 decimals matches the currency and the PRD (§6.1).

**Consequences.**
- ✅ Exact arithmetic end to end; database-level aggregation stays precise.
- ✅ `14,2` supports values up to 999,999,999,999.99 — far beyond any SMB construction contract.
- ⚠️ `Decimal` is not JSON-serializable: serialization to string is required at every boundary. Enforced by typing money as `Money = string` in all DTOs (`API-ACTIONS.md` §1).
- ⚠️ Developers must not perform arithmetic on the string form — all math lives in `shared/lib/money.ts` helpers.

---

## ADR-007 — Configurable 10,000 EGP approval threshold, snapshotted per transaction

**Status:** Accepted

**Context.** The PRD sets a large-expense approval threshold of 10,000 EGP (§6.3), configurable by an administrator. A fixed constant cannot serve both a small company (where 10,000 EGP is significant) and a growing one. Critically, changing the threshold must not rewrite the approval semantics of past transactions.

**Decision.** Store `largeExpenseThreshold` in `CompanySetting` (default `10000.00`, changeable only with `organization.settings.updateThreshold`, i.e. Super Admin). At submission, copy the active value into `Expense.thresholdAtSubmission` and compute `requiresApproval` from that snapshot.

**Alternatives considered.**
- *Hardcoded constant:* requires a deployment to change and ignores company size differences.
- *Setting read live at report time:* a threshold change would retroactively make historical approved expenses look like they had bypassed approval — an audit integrity failure.
- *Per-project or per-role thresholds:* more flexible, but multi-dimensional approval policy is explicitly out of MVP scope.

**Consequences.**
- ✅ Historical records remain defensible: each expense records the rule it was judged against.
- ✅ Configurable without deployment; auditable via `SETTINGS_UPDATED` with before/after values.
- ✅ Separation of duties: the Admin/Manager who approves cannot raise the threshold that triggers approval.
- ⚠️ One extra column per expense; trivially cheap relative to the integrity it buys.

---

## ADR-008 — Single `ApprovalRequest` table for both approval types

**Status:** Accepted

**Context.** Two approval flows exist in the MVP — large expenses and project budget changes — and Phase 2 will add more (purchase orders, leave). Management wants a single approval inbox.

**Decision.** One `ApprovalRequest` table discriminated by `type`, with nullable typed foreign keys (`expenseId`, `projectId`) and database CHECK constraints guaranteeing the correct key is set for each type.

**Alternatives considered.**
- *Separate tables per type:* strict FKs and no nullable columns, but the inbox becomes a UNION query and each new approval type duplicates the state machine.
- *Fully polymorphic (`entityType` + `entityId` string):* maximum flexibility, zero referential integrity — unacceptable for approval records tied to money.

**Consequences.**
- ✅ One inbox query, one state machine, one audit shape.
- ✅ Referential integrity preserved through real foreign keys.
- ✅ Adding a Phase 2 approval type means a new enum value and one nullable FK.
- ⚠️ Nullable FKs require CHECK constraints to stay correct — written explicitly in `ERD.md` §4.12.

---

## ADR-009 — Denormalize `Expense.approvalStatus` and `Project.plannedBudget`

**Status:** Accepted

**Context.** The most frequent query in the system is "sum approved expenses for project X", used by the dashboard, project pages, and three reports. Deriving approval status from a join on `ApprovalRequest` for every aggregate is wasteful, and `plannedBudget` is read on nearly every project view.

**Decision.** Two controlled denormalizations:
1. `Expense.approvalStatus` mirrors the linked request (`NOT_REQUIRED` when none).
2. `Project.plannedBudget` holds the current approved budget; history lives in `ApprovalRequest`.

Both are written **only** by the owning use case inside the same transaction as the source-of-truth change.

**Alternatives considered.**
- *Always derive by join:* perfectly normalized, but adds a join to every financial aggregate and complicates index design.
- *Materialized views:* refresh lag is unacceptable for figures that must update immediately after approval.

**Consequences.**
- ✅ Project cost aggregation is a single indexed scan on `(projectId, approvalStatus, deletedAt)`.
- ✅ Budget reads need no history traversal, while history remains fully reconstructible.
- ⚠️ Two places can disagree if written outside the sanctioned path. Mitigated by: writes confined to `modules/approvals` use cases, same-transaction updates, and an integration test asserting `plannedBudget` equals the initial value plus all approved changes.

---

## ADR-010 — Store user content as entered; translate only system lookups

**Status:** Accepted

**Context.** The app is bilingual (Arabic default, English secondary). A naive approach would add `nameAr`/`nameEn` to every table.

**Decision.** Three-tier strategy:
1. **User-entered content** (project names, descriptions, expense descriptions) — stored once, exactly as typed. Users are not asked to translate their own data.
2. **System lookups** (`Department`, `ExpenseCategory`, `IncomeCategory`) — carry `nameEn` + `nameAr`, resolved server-side by session locale.
3. **System-generated text** (notifications, validation errors) — stored/returned as i18n **keys plus a JSON payload**, rendered in the reader's locale at display time.

**Alternatives considered.**
- *Translate everything:* doubles data entry effort for users who work in one language. Rejected.
- *A generic `Translation` table:* flexible, but adds a join to every read for a two-language MVP.
- *Store rendered notification sentences:* fast, but an Arabic-created notification would read in Arabic for an English user forever.

**Consequences.**
- ✅ Minimal data-entry burden; no empty translation fields.
- ✅ The same notification row renders correctly for an Arabic reader and an English reader.
- ✅ Server-side errors localize in the client's language because only keys cross the boundary.
- ⚠️ Search over user content is single-language by nature — acceptable and expected.

---

## ADR-011 — FSD for the frontend, Clean Architecture modules for the backend

**Status:** Accepted

**Context.** Feature-Sliced Design is a frontend methodology. Next.js App Router colocates server code with UI, so a pure-FSD reading would scatter one table's business logic across many slices.

**Decision.** Two axes (`FOLDER-STRUCTURE.md` §1): FSD layers (`app → widgets → features → entities → shared`) own presentation; `src/modules/<module>` (domain/application/infrastructure) owns business logic. Server Actions live in feature slices and are thin: authenticate, validate, delegate.

**Alternatives considered.**
- *Pure FSD including server logic:* would place expense rules inside `features/create-expense` and duplicate them in `features/update-expense`. Violates DRY.
- *Pure layered (controllers/services/repositories) for everything:* loses FSD's UI-coupling benefits and produces god-folders like `components/`.

**Consequences.**
- ✅ A table's rules exist in exactly one place, reusable by every action touching it.
- ✅ UI slices remain independently reviewable and deletable.
- ✅ Domain logic is unit-testable without React, Next.js, or a database.
- ⚠️ Two structural conventions to learn — documented, and the boundary is a single rule: "UI in FSD layers, rules in modules".

---

## ADR-012 — Authorization as query filters, not post-fetch checks

**Status:** Accepted

**Context.** A Project Manager may access only assigned projects (PRD §5.1). A naive implementation fetches the row and then checks ownership — loading unauthorized data into memory and risking leaks through logs, errors, or partial renders (OWASP A01).

**Decision.** Scope is expressed as a Prisma `where` fragment applied at query time (`RBAC-MATRIX.md` §5). Unauthorized rows are never loaded. A scoped miss returns `NOT_FOUND`, not `FORBIDDEN`, so record existence is not disclosed.

**Alternatives considered.**
- *Post-fetch ownership check:* simpler but loads data the user may not see and leaks existence via distinguishable error codes.
- *PostgreSQL Row-Level Security:* strong defense in depth, but requires per-request session variables through a serverless connection pool — meaningful complexity for a single-tenant MVP. Revisit if multi-tenancy arrives.

**Consequences.**
- ✅ Unauthorized data never enters application memory.
- ✅ Existence is not leaked through error codes.
- ✅ Pagination counts are correct for the caller's scope.
- ⚠️ Every query must apply the scope helper — enforced by repository-level helpers and table-driven RBAC tests covering every role × permission cell.

---

## ADR-013 — In-app notifications only, created inside the triggering transaction

**Status:** Accepted

**Context.** The MVP requires in-app notifications for four events (PRD §5.8). Email, SMS, and WhatsApp are Phase 2.

**Decision.** A `Notification` table written inside the same transaction as the triggering mutation. Recipients are resolved by *permission* ("all holders of `approvals.request.decide`"), not by hardcoded user ids. The UI reads unread counts via RSC on navigation plus lightweight polling — no WebSockets, no queue.

**Alternatives considered.**
- *WebSockets/SSE real-time:* unnecessary for 5–10 users, adds a stateful connection to a serverless deployment.
- *Background job queue:* another moving part; at this volume an in-transaction insert is simpler and stronger.
- *Fire-and-forget after commit:* risks a successful action with no notification, or a notification for a rolled-back action.

**Consequences.**
- ✅ A notification exists if and only if the action committed.
- ✅ Recipient resolution automatically follows role changes.
- ✅ Adding an email channel in Phase 2 means reading unsent rows from the same table — no schema change.
- ⚠️ Not real-time; acceptable for approval workflows measured in hours.

---

## ADR-014 — `exceljs` and `@react-pdf/renderer` for exports

**Status:** Accepted

**Context.** The PRD requires Excel and PDF export of reports (§5.6), with Arabic RTL content and EGP formatting.

**Decision.** `exceljs` for XLSX and `@react-pdf/renderer` for PDF, both executed in Node-runtime Route Handlers that re-run the same authorized report query.

**Alternatives considered.**
- *Puppeteer/Chromium HTML-to-PDF:* best fidelity and easiest RTL, but a headless browser exceeds serverless function size and cold-start budgets on Vercel.
- *Client-side jsPDF/SheetJS:* pushes data volume and formatting to the browser and makes authorization unverifiable at export time.
- *A third-party export service:* another vendor and data-egress path for a core feature.

**Consequences.**
- ✅ Exports are server-authorized and cannot widen data access.
- ✅ Real XLSX (formulas, column types, number formats), not CSV renamed.
- ⚠️ Arabic PDF requires an embedded font (Cairo/Amiri) and explicit RTL layout — a known, contained cost in `pdf.exporter.tsx`.
- ⚠️ PDF layout is hand-built rather than HTML-driven; acceptable for five report templates.

---

## ADR-015 — Prisma as ORM with a repository port per module

**Status:** Accepted

**Context.** We need type-safe database access, versioned migrations, and a domain layer free of infrastructure imports.

**Decision.** Prisma for schema, migrations, and queries. Each module declares repository **ports** (interfaces) in its application layer; Prisma implementations live in its infrastructure layer. Domain code imports neither.

**Alternatives considered.**
- *Drizzle:* lighter and closer to SQL, but Prisma's migration workflow, relation ergonomics, and `Decimal` handling better fit a schema-heavy ERP with a small team.
- *Raw SQL + query builder:* maximum control, far more boilerplate and hand-written types.
- *Prisma used directly everywhere:* fewer files, but couples domain rules to the ORM and makes unit tests require a database.

**Consequences.**
- ✅ Generated types keep the ERD and the code in sync — the schema is the single source of truth.
- ✅ Domain rules unit-test in milliseconds with no database.
- ✅ `$transaction` gives the atomic mutation + audit + notification guarantee this design depends on.
- ⚠️ Ports add indirection; justified where business rules exist, and deliberately skipped for simple read-only report queries in `modules/reporting`, which use Prisma directly.
