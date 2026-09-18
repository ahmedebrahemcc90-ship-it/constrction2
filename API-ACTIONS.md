# API & SERVER ACTIONS CONTRACT (MVP)

**Status:** Draft for Technical Approval
**Transport:** Next.js Server Actions for all business operations. Route Handlers only for binary/streaming and Better Auth.

---

## 1. Universal Contract

Every action returns `Result<T>` and never throws across the boundary.

```ts
export type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: AppError };

export type AppError = {
  code: 'UNAUTHENTICATED' | 'FORBIDDEN' | 'VALIDATION' | 'NOT_FOUND'
      | 'CONFLICT' | 'RULE_VIOLATION' | 'INTERNAL';
  messageKey: string;                        // i18n key rendered client-side
  fieldErrors?: Record<string, string[]>;
};
```

Shared types:

```ts
type Paginated<T> = { items: T[]; total: number; page: number; pageSize: number };
type DateRange   = { from: Date; to: Date };
type Money       = string;                   // decimal string, never a JS number
type Id          = string;                   // cuid
```

Every action performs, in order: **session → permission → scope → Zod parse → use case → transaction (mutation + audit + notification) → revalidate**.

Money crosses the wire as a **decimal string** to avoid IEEE-754 loss; `MoneyInput` and formatters handle conversion.

---

## 2. Authentication (`modules/iam`)

| Action | Signature | Roles |
|---|---|---|
| `signInAction` | `(input: { email: string; password: string }) => Promise<Result<{ redirectTo: string }>>` | public |
| `signOutAction` | `() => Promise<Result<null>>` | any authenticated |
| `getSessionContext` | `() => Promise<SessionContext \| null>` | internal (server-only) |
| `updateOwnProfileAction` | `(input: UpdateOwnProfileInput) => Promise<Result<UserProfile>>` | any authenticated |
| `changeOwnPasswordAction` | `(input: { currentPassword: string; newPassword: string }) => Promise<Result<null>>` | any authenticated |

```ts
type SessionContext = { userId: Id; role: UserRole; employeeId: Id | null; locale: Locale };
type UpdateOwnProfileInput = { name: string; locale: Locale; image?: string | null };
```

Rules: inactive users are rejected with a generic message; failed logins are rate-limited and audited (`LOGIN_FAILED`) without revealing whether the email exists.

---

## 3. Users & Settings (`modules/iam`, `modules/organization`)

| Action | Signature | Permission |
|---|---|---|
| `listUsersAction` | `(f: UserFilters) => Promise<Result<Paginated<UserListItem>>>` | `iam.user.read` |
| `createUserAction` | `(i: CreateUserInput) => Promise<Result<{ id: Id }>>` | `iam.user.create` |
| `updateUserAction` | `(i: UpdateUserInput) => Promise<Result<UserListItem>>` | `iam.user.update` |
| `changeUserRoleAction` | `(i: { userId: Id; role: UserRole }) => Promise<Result<null>>` | `iam.user.changeRole` (Super Admin) |
| `setUserActiveAction` | `(i: { userId: Id; isActive: boolean }) => Promise<Result<null>>` | `iam.user.setActive` |
| `linkUserToEmployeeAction` | `(i: { userId: Id; employeeId: Id \| null }) => Promise<Result<null>>` | `iam.user.linkEmployee` |
| `getCompanySettingsAction` | `() => Promise<Result<CompanySettings>>` | `organization.settings.read` |
| `updateCompanySettingsAction` | `(i: UpdateCompanySettingsInput) => Promise<Result<CompanySettings>>` | `organization.settings.update` |

```ts
type CreateUserInput = {
  name: string; email: string; role: UserRole;
  locale?: Locale; employeeId?: Id | null; temporaryPassword: string;
};

type UpdateCompanySettingsInput = {
  companyName: string; defaultLocale: Locale; currencyCode: 'EGP' | 'USD' | 'SAR';
  timezone: string; largeExpenseThreshold: Money; budgetAlertPercent: number;
  fiscalYearStartMonth: number;
};
```

Rules: role change and deactivation invalidate the target's sessions; the last active `SUPER_ADMIN` cannot be demoted, deactivated, or deleted; threshold changes require `organization.settings.updateThreshold` and are audited (`SETTINGS_UPDATED`) with before/after values.

---

## 4. Projects (`modules/projects`)

| Action | Signature | Permission |
|---|---|---|
| `listProjectsAction` | `(f: ProjectFilters) => Promise<Result<Paginated<ProjectListItem>>>` | `projects.project.read` (scoped) |
| `getProjectAction` | `(i: { projectId: Id }) => Promise<Result<ProjectDetail>>` | `projects.project.read` (scoped) |
| `createProjectAction` | `(i: CreateProjectInput) => Promise<Result<{ id: Id; code: string }>>` | `projects.project.create` |
| `updateProjectAction` | `(i: UpdateProjectInput) => Promise<Result<ProjectDetail>>` | `projects.project.update` (scoped, field-limited) |
| `changeProjectStatusAction` | `(i: ChangeProjectStatusInput) => Promise<Result<null>>` | `projects.project.changeStatus` |
| `assignProjectManagerAction` | `(i: { projectId: Id; projectManagerId: Id }) => Promise<Result<null>>` | `projects.project.assignManager` |
| `deleteProjectAction` | `(i: { projectId: Id }) => Promise<Result<null>>` | `projects.project.delete` |
| `getProjectFinancialSummaryAction` | `(i: { projectId: Id; range?: DateRange }) => Promise<Result<ProjectFinancialSummary>>` | `projects.project.readFinancials` |
| `assignEmployeeToProjectAction` | `(i: AssignEmployeeInput) => Promise<Result<{ id: Id }>>` | `projects.assignment.manage` |
| `endProjectAssignmentAction` | `(i: { assignmentId: Id; assignedTo: Date }) => Promise<Result<null>>` | `projects.assignment.manage` |
| `listProjectTeamAction` | `(i: { projectId: Id }) => Promise<Result<ProjectTeamMember[]>>` | `projects.assignment.read` |

```ts
type CreateProjectInput = {
  name: string;                    // 3–160, trimmed, non-empty
  description?: string;
  location?: string;
  clientId?: Id | null;
  projectManagerId?: Id | null;    // required when status = ACTIVE
  status: ProjectStatus;           // default PLANNED
  startDate: Date;
  expectedEndDate: Date;           // >= startDate
  contractReference?: string;
  contractValue?: Money;           // > 0 when present
  plannedBudget: Money;            // > 0  (PRD §6.2)
};

type UpdateProjectInput = { projectId: Id } & Partial<Omit<CreateProjectInput, 'plannedBudget'>>;
// plannedBudget is intentionally absent — it changes only via an approved budget request.

type ProjectFinancialSummary = {
  plannedBudget: Money; actualCost: Money; pendingCost: Money; committedCost: Money;
  totalIncome: Money; profit: Money; profitMargin: number | null;
  budgetVariance: Money; budgetUtilization: number; currencyCode: string;
};
```

Rules: `plannedBudget` is immutable through update actions; `ACTIVE` requires a project manager; `COMPLETED` requires `actualEndDate` and no `PENDING` approvals; `PROJECT_MANAGER` may update only `description`, `location`, `status`, `actualEndDate`, `contractReference` on assigned projects; an unassigned project returns `NOT_FOUND`.

---

## 5. Budget Changes (`modules/projects` + `modules/approvals`)

| Action | Signature | Permission |
|---|---|---|
| `requestBudgetChangeAction` | `(i: RequestBudgetChangeInput) => Promise<Result<{ approvalRequestId: Id }>>` | `projects.budget.request` (scoped) |
| `listProjectBudgetHistoryAction` | `(i: { projectId: Id }) => Promise<Result<BudgetHistoryItem[]>>` | `projects.project.readFinancials` |

```ts
type RequestBudgetChangeInput = {
  projectId: Id;
  requestedAmount: Money;   // > 0, must differ from current plannedBudget
  reason: string;           // 10–500 chars, required (PRD §6.2)
};

type BudgetHistoryItem = {
  id: Id; previousAmount: Money; requestedAmount: Money; reason: string;
  status: ApprovalStatus; requestedBy: string; requestedAt: Date;
  decidedBy: string | null; decidedAt: Date | null; decisionNote: string | null;
};
```

Rules: only one `PENDING` budget request per project (`CONFLICT` otherwise); the project keeps its previous budget until approval; approval updates `Project.plannedBudget` in the same transaction as the decision and the `BUDGET_CHANGED` audit row.

---

## 6. Expenses (`modules/finance`)

| Action | Signature | Permission |
|---|---|---|
| `listExpensesAction` | `(f: ExpenseFilters) => Promise<Result<Paginated<ExpenseListItem>>>` | `finance.expense.read` (scoped) |
| `getExpenseAction` | `(i: { expenseId: Id }) => Promise<Result<ExpenseDetail>>` | `finance.expense.read` (scoped) |
| `createExpenseAction` | `(i: CreateExpenseInput) => Promise<Result<CreateExpenseResult>>` | `finance.expense.createProject` \| `createGeneral` |
| `updateExpenseAction` | `(i: UpdateExpenseInput) => Promise<Result<ExpenseDetail>>` | `finance.expense.update` |
| `deleteExpenseAction` | `(i: { expenseId: Id; reason?: string }) => Promise<Result<null>>` | `finance.expense.delete` |

```ts
type CreateExpenseInput = {
  scope: 'PROJECT' | 'GENERAL';
  projectId?: Id | null;        // required iff scope = PROJECT, forbidden otherwise
  categoryId: Id;
  amount: Money;                // > 0
  expenseDate: Date;            // not > today + 1 day
  description: string;          // 3–500
  paymentMethod: PaymentMethod;
  paymentReference?: string;
  vendorName?: string;
};

type CreateExpenseResult = {
  id: Id; reference: string;
  approvalStatus: ApprovalStatus;      // NOT_REQUIRED | PENDING
  requiresApproval: boolean;
  thresholdAtSubmission: Money;
};

type UpdateExpenseInput = { expenseId: Id } & Partial<Omit<CreateExpenseInput, 'scope'>>;

type ExpenseFilters = {
  scope?: ExpenseScope; projectId?: Id; categoryId?: Id;
  approvalStatus?: ApprovalStatus[]; range?: DateRange;
  minAmount?: Money; maxAmount?: Money; search?: string;
  page?: number; pageSize?: number; sort?: 'date' | 'amount';
};
```

Rules: `amount > threshold` ⇒ `approvalStatus = PENDING` + approval request + notification to approvers; `thresholdAtSubmission` is snapshotted; approved expenses are immutable in `amount`, `projectId`, `scope`, `categoryId` (`RULE_VIOLATION` on attempt); `PROJECT_MANAGER` may edit only their own pending expenses on assigned projects; deletion is soft and audited with a reason.

---

## 7. Income (`modules/finance`)

| Action | Signature | Permission |
|---|---|---|
| `listIncomeAction` | `(f: IncomeFilters) => Promise<Result<Paginated<IncomeListItem>>>` | `finance.income.read` (scoped) |
| `createIncomeAction` | `(i: CreateIncomeInput) => Promise<Result<{ id: Id; reference: string }>>` | `finance.income.create` |
| `updateIncomeAction` | `(i: UpdateIncomeInput) => Promise<Result<IncomeListItem>>` | `finance.income.update` |
| `deleteIncomeAction` | `(i: { incomeId: Id; reason?: string }) => Promise<Result<null>>` | `finance.income.delete` |

```ts
type CreateIncomeInput = {
  projectId?: Id | null;
  clientId?: Id | null;          // defaults to the project's client when project-scoped
  categoryId: Id;
  amount: Money;                 // > 0
  receivedDate: Date;
  description: string;
  paymentMethod: PaymentMethod;
  paymentReference?: string;
  payerName?: string;
};
```

Income requires no approval in the MVP but is fully audited.

---

## 8. Categories (`modules/finance`)

| Action | Signature | Permission |
|---|---|---|
| `listExpenseCategoriesAction` | `(f?: { scope?: ExpenseScope; activeOnly?: boolean }) => Promise<Result<CategoryOption[]>>` | `finance.category.read` |
| `listIncomeCategoriesAction` | `(f?: { activeOnly?: boolean }) => Promise<Result<CategoryOption[]>>` | `finance.category.read` |
| `createExpenseCategoryAction` | `(i: CreateExpenseCategoryInput) => Promise<Result<{ id: Id }>>` | `finance.category.create` |
| `updateCategoryAction` | `(i: UpdateCategoryInput) => Promise<Result<null>>` | `finance.category.update` |
| `deactivateCategoryAction` | `(i: { categoryId: Id; kind: 'EXPENSE' \| 'INCOME' }) => Promise<Result<null>>` | `finance.category.update` |

```ts
type CategoryOption = { id: Id; code: string; label: string; allowedScope?: 'PROJECT' | 'GENERAL' | 'BOTH'; isSystem: boolean };
```

`label` is resolved server-side from `nameAr`/`nameEn` using the session locale. System categories may be deactivated, never deleted.

---

## 9. Approvals (`modules/approvals`)

| Action | Signature | Permission |
|---|---|---|
| `listPendingApprovalsAction` | `(f: ApprovalFilters) => Promise<Result<Paginated<ApprovalListItem>>>` | `approvals.request.readInbox` |
| `listOwnApprovalRequestsAction` | `(f: ApprovalFilters) => Promise<Result<Paginated<ApprovalListItem>>>` | `approvals.request.readOwn` |
| `getApprovalRequestAction` | `(i: { approvalRequestId: Id }) => Promise<Result<ApprovalDetail>>` | inbox or own |
| `decideApprovalAction` | `(i: DecideApprovalInput) => Promise<Result<null>>` | `approvals.request.decide` |
| `cancelApprovalRequestAction` | `(i: { approvalRequestId: Id }) => Promise<Result<null>>` | `approvals.request.cancelOwn` |

```ts
type DecideApprovalInput = {
  approvalRequestId: Id;
  decision: 'APPROVE' | 'REJECT';
  decisionNote?: string;         // required when decision = REJECT
};

type ApprovalListItem = {
  id: Id; type: ApprovalType; status: ApprovalStatus;
  amount: Money; previousAmount: Money | null;
  projectName: string | null; expenseReference: string | null;
  requestedBy: string; requestedAt: Date; reason: string | null;
};
```

Rules: only `PENDING` requests are decidable (`CONFLICT` otherwise); self-approval is blocked (`FORBIDDEN`) except for an audited `SUPER_ADMIN` override; approving an expense sets `Expense.approvalStatus = APPROVED` and may trigger budget warning/exceeded notifications; approving a budget change updates `Project.plannedBudget` — all within one transaction with the audit row and notifications.

---

## 10. Employees (`modules/employees`)

| Action | Signature | Permission |
|---|---|---|
| `listEmployeesAction` | `(f: EmployeeFilters) => Promise<Result<Paginated<EmployeeListItem>>>` | `employees.employee.read` (scoped) |
| `getEmployeeAction` | `(i: { employeeId: Id }) => Promise<Result<EmployeeDetail>>` | `employees.employee.read` (scoped) |
| `createEmployeeAction` | `(i: CreateEmployeeInput) => Promise<Result<{ id: Id; employeeCode: string }>>` | `employees.employee.create` |
| `updateEmployeeAction` | `(i: UpdateEmployeeInput) => Promise<Result<EmployeeDetail>>` | `employees.employee.update` |
| `deleteEmployeeAction` | `(i: { employeeId: Id }) => Promise<Result<null>>` | `employees.employee.delete` |
| `listDepartmentsAction` | `() => Promise<Result<DepartmentOption[]>>` | `employees.department.read` |
| `manageDepartmentAction` | `(i: ManageDepartmentInput) => Promise<Result<{ id: Id }>>` | `employees.department.manage` |

```ts
type CreateEmployeeInput = {
  fullName: string; nationalId?: string; phone: string; email?: string;
  jobTitle: string; departmentId?: Id | null;
  baseSalary: Money;             // >= 0
  hireDate: Date;                // not in the future
  status: EmployeeStatus;
  address?: string; emergencyContact?: string; notes?: string;
};

type EmployeeDetail = {
  id: Id; employeeCode: string; fullName: string; phone: string; jobTitle: string;
  department: { id: Id; label: string } | null;
  baseSalary?: Money;            // present only with employees.employee.readSalary
  hireDate: Date; terminationDate: Date | null; status: EmployeeStatus;
  activeProjects: { id: Id; name: string; roleOnProject: string | null }[];
};
```

Rules: `baseSalary` is omitted from the Prisma selection for roles lacking `readSalary`; an `EMPLOYEE` reads only the record linked to their user; terminated employees cannot be assigned to projects; `nationalId` is unique among active employees.

---

## 11. Clients (`modules/organization`)

| Action | Signature | Permission |
|---|---|---|
| `listClientsAction` | `(f: { search?: string; page?: number }) => Promise<Result<Paginated<ClientListItem>>>` | `organization.client.read` |
| `createClientAction` | `(i: CreateClientInput) => Promise<Result<{ id: Id }>>` | `organization.client.create` |
| `updateClientAction` | `(i: UpdateClientInput) => Promise<Result<ClientListItem>>` | `organization.client.update` |
| `deleteClientAction` | `(i: { clientId: Id }) => Promise<Result<null>>` | `organization.client.delete` |

A client referenced by a non-deleted project cannot be deleted (`CONFLICT`).

---

## 12. Documents (`modules/documents`)

| Endpoint | Signature | Permission |
|---|---|---|
| `requestUploadTokenAction` | `(i: RequestUploadTokenInput) => Promise<Result<UploadToken>>` | owning-entity upload permission |
| `confirmUploadAction` | `(i: ConfirmUploadInput) => Promise<Result<{ id: Id }>>` | same |
| `listDocumentsAction` | `(i: { ownerType: DocumentOwnerType; ownerId: Id }) => Promise<Result<DocumentListItem[]>>` | owning-entity read permission |
| `deleteDocumentAction` | `(i: { documentId: Id }) => Promise<Result<null>>` | owning-entity delete permission |
| `GET /api/documents/:id/download` | Route Handler → stream | owning-entity read permission |

```ts
type RequestUploadTokenInput = {
  ownerType: DocumentOwnerType; ownerId: Id;
  documentType: DocumentType;
  fileName: string; mimeType: string; sizeBytes: number;
};

type UploadToken = { uploadUrl: string; storageKey: string; expiresAt: Date };
```

Rules: MIME allow-list and size caps (10 MB general, 25 MB photos/drawings) validated server-side; `storageKey` is server-generated — client-supplied names are metadata only; downloads authorize against the owning entity and return `404` when denied; uploads and confidential downloads are audited.

---

## 13. Reporting & Exports (`modules/reporting`)

| Action | Signature | Permission |
|---|---|---|
| `getDashboardSummaryAction` | `(i: { range?: DateRange }) => Promise<Result<DashboardSummary>>` | `reporting.dashboard.*` (scoped) |
| `getProfitabilityReportAction` | `(f: ReportFilters) => Promise<Result<ProfitabilityReport>>` | `reporting.profitability.read` |
| `getBudgetVarianceReportAction` | `(f: ReportFilters) => Promise<Result<BudgetVarianceReport>>` | `reporting.budgetVariance.read` |
| `getExpenseReportAction` | `(f: ExpenseReportFilters) => Promise<Result<ExpenseReport>>` | `reporting.expense.read` |
| `getCashFlowReportAction` | `(f: CashFlowFilters) => Promise<Result<CashFlowReport>>` | `reporting.cashflow.read` |
| `GET /api/reports/:report/export?format=xlsx\|pdf` | Route Handler → stream | `reporting.export` |

```ts
type DashboardSummary = {
  projects: { active: number; planned: number; paused: number; completed: number };
  finance: {
    periodIncome: Money; periodExpenses: Money; netCashFlow: Money;
    pendingExpenseTotal: Money; pendingApprovalCount: number;
  };
  budgetAlerts: { projectId: Id; projectName: string; utilization: number; severity: 'WARNING' | 'EXCEEDED' }[];
  recentActivity: { id: Id; type: string; summary: string; at: Date }[];
  currencyCode: string;
};

type ProfitabilityReport = {
  rows: {
    projectId: Id; code: string; name: string; status: ProjectStatus;
    totalIncome: Money; actualCost: Money; pendingCost: Money;
    profit: Money; profitMargin: number | null;
  }[];
  totals: { totalIncome: Money; actualCost: Money; pendingCost: Money; profit: Money };
  generatedAt: Date; currencyCode: string;
};

type CashFlowReport = {
  periods: { label: string; from: Date; to: Date; inflow: Money; outflow: Money; net: Money; runningNet: Money }[];
  totals: { inflow: Money; outflow: Money; net: Money };
  granularity: 'MONTH' | 'QUARTER';
};
```

Rules: pending amounts are always returned as **separate fields** and never folded into actuals (PRD §6.4); all reports apply the caller's scope; exports re-run the same scoped query; period boundaries are computed in `CompanySetting.timezone`; a report spanning mixed currencies fails safely with `RULE_VIOLATION`.

---

## 14. Notifications (`modules/notifications`)

| Action | Signature | Permission |
|---|---|---|
| `listNotificationsAction` | `(f: { unreadOnly?: boolean; page?: number }) => Promise<Result<Paginated<NotificationItem>>>` | own only |
| `getUnreadCountAction` | `() => Promise<Result<{ count: number }>>` | own only |
| `markNotificationReadAction` | `(i: { notificationId: Id }) => Promise<Result<null>>` | own only |
| `markAllNotificationsReadAction` | `() => Promise<Result<{ updated: number }>>` | own only |

```ts
type NotificationItem = {
  id: Id; type: NotificationType;
  title: string; body: string;          // rendered from keys + payload in the reader's locale
  linkPath: string | null; readAt: Date | null; createdAt: Date;
};
```

Notifications are created only by use cases inside their transaction — there is no public "create notification" action.

---

## 15. Audit Log (`modules/audit`)

| Action | Signature | Permission |
|---|---|---|
| `listAuditLogsAction` | `(f: AuditFilters) => Promise<Result<Paginated<AuditLogItem>>>` | `audit.log.readAll` |
| `getEntityHistoryAction` | `(i: { entityType: AuditEntityType; entityId: Id }) => Promise<Result<AuditLogItem[]>>` | `audit.log.readEntity` (scoped) |
| `GET /api/reports/audit/export` | Route Handler | `audit.log.export` (Super Admin) |

```ts
type AuditFilters = {
  entityType?: AuditEntityType; entityId?: Id; actorUserId?: Id;
  eventType?: AuditEventType[]; range?: DateRange; page?: number; pageSize?: number;
};

type AuditLogItem = {
  id: string; actorName: string | null; actorRole: UserRole | null;
  eventType: AuditEventType; entityType: AuditEntityType; entityId: Id;
  summary: string; changes: Record<string, { before: unknown; after: unknown }> | null;
  createdAt: Date;
};
```

No create, update, or delete action exists for audit logs — the write path is an internal, transaction-scoped `audit.record(tx, ctx, event)` helper.

---

## 16. Route Handlers (non-Server-Action surface)

| Route | Method | Purpose | Why not a Server Action |
|---|---|---|---|
| `/api/auth/[...all]` | GET/POST | Better Auth | library-owned HTTP contract |
| `/api/documents/upload-token` | POST | Vercel Blob client-upload token | called by the blob client SDK |
| `/api/documents/:id/download` | GET | authorized file stream | needs `Content-Disposition` and streaming |
| `/api/reports/:report/export` | GET | Excel/PDF stream | binary response with headers |
| `/api/health` | GET | liveness probe | infrastructure |

All Route Handlers run on the Node.js runtime and repeat the same session → permission → scope checks.

---

## 17. Revalidation Map

| Action group | Revalidated paths/tags |
|---|---|
| project create/update/delete | `/[locale]/projects`, `/[locale]/projects/[id]`, `/[locale]` (dashboard) |
| expense create/update/delete | `/[locale]/finance/expenses`, project detail, `/[locale]`, `/[locale]/reports/*` |
| income mutations | `/[locale]/finance/income`, project detail, `/[locale]`, cash-flow report |
| approval decision | `/[locale]/approvals`, related expense/project, `/[locale]` |
| employee mutations | `/[locale]/employees`, project team pages |
| settings update | global layout data |

---

## 18. Action Implementation Checklist

Every Server Action must satisfy all of the following before review:

1. `'use server'` at the top of a `*.action.ts` file inside its feature slice.
2. Session resolved via `getSessionContext()`; `UNAUTHENTICATED` when absent.
3. `assertPermission` with the exact permission from `RBAC-MATRIX.md`.
4. Scope applied as a **query filter**, not a post-fetch check.
5. Input parsed with a Zod schema exported from the owning module; the same schema powers the client form.
6. Business rules live in the module's domain/use case — never in the action.
7. Mutation, `AuditLog`, and notifications share one `prisma.$transaction`.
8. Returns `Result<T>`; no thrown exception crosses the boundary; internals are logged, not returned.
9. `revalidatePath`/`revalidateTag` per §17.
10. Money is passed and returned as a decimal string.
11. Error messages are i18n keys, never hardcoded sentences.
12. Covered by a unit test for the rule and a table-driven RBAC test for the permission.
