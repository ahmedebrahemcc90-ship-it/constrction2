# RBAC MATRIX — Roles and Permissions (MVP)

**Status:** Draft for Technical Approval
**Model:** Fixed, predefined roles. No custom roles in the MVP (PRD §5.1).
**Enforcement:** Server-side in every Server Action and Route Handler. UI hiding is cosmetic only.

---

## 1. Roles

| Code | Role | Description |
|---|---|---|
| `SUPER_ADMIN` | Super Admin | System owner. Full access including users, roles, and company settings. |
| `ADMIN_MANAGER` | Admin / General Manager | Company owner or GM. Full business access; cannot change system-level settings or elevate roles. |
| `PROJECT_MANAGER` | Project Manager | Operates only on projects where they are the assigned manager. |
| `ACCOUNTANT` | Accountant | Full finance operations and reporting; read-only on projects and employees. |
| `EMPLOYEE` | Employee | Read-only access to their own profile and assignments. No financial visibility. |

---

## 2. Permission Notation

- **C** Create · **R** Read · **U** Update · **D** Delete (soft) · **A** Approve
- **✓** allowed for all in-scope records
- **◐** allowed, restricted by scope (see §5)
- **—** denied
- **own** limited to the actor's own records

Permission identifier format: `<module>.<resource>.<action>` — e.g. `finance.expense.approve`.

---

## 3. Permission Matrix

### 3.1 Projects

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| List / view projects | `projects.project.read` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Create project | `projects.project.create` | C | C | — | — | — |
| Update project details | `projects.project.update` | U | U | ◐ assigned, limited fields | — | — |
| Change project status | `projects.project.changeStatus` | U | U | ◐ assigned | — | — |
| Soft-delete project | `projects.project.delete` | D | D | — | — | — |
| Assign project manager | `projects.project.assignManager` | U | U | — | — | — |
| Set initial budget (on create) | `projects.project.setInitialBudget` | C | C | — | — | — |
| Request budget change | `projects.budget.request` | ✓ | ✓ | ◐ assigned | — | — |
| Approve budget change | `projects.budget.approve` | A | A | — | — | — |
| View project financial summary | `projects.project.readFinancials` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Manage project staffing | `projects.assignment.manage` | C/U/D | C/U/D | ◐ assigned | — | — |
| View project staffing | `projects.assignment.read` | ✓ | ✓ | ◐ assigned | ✓ | own |

Project Manager editable fields: `description`, `location`, `status`, `actualEndDate`, `contractReference`.
Never editable by a Project Manager: `plannedBudget`, `contractValue`, `clientId`, `projectManagerId`, `code`.

### 3.2 Finance — Expenses

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View expenses (all) | `finance.expense.read` | ✓ | ✓ | ◐ assigned projects | ✓ | — |
| Create general expense | `finance.expense.createGeneral` | C | C | — | C | — |
| Create project expense | `finance.expense.createProject` | C | C | ◐ assigned | C | — |
| Update expense (pre-approval) | `finance.expense.update` | U | U | ◐ own + assigned, pending only | U | — |
| Soft-delete expense | `finance.expense.delete` | D | D | — | ◐ pending only | — |
| Approve / reject large expense | `finance.expense.approve` | A | A | — | — | — |
| Cancel own pending request | `finance.expense.cancelOwnRequest` | ✓ | ✓ | own | own | — |
| View approval history | `finance.expense.readHistory` | ✓ | ✓ | ◐ assigned | ✓ | — |

### 3.3 Finance — Income

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View income | `finance.income.read` | ✓ | ✓ | ◐ assigned projects | ✓ | — |
| Record income | `finance.income.create` | C | C | — | C | — |
| Update income | `finance.income.update` | U | U | — | U | — |
| Soft-delete income | `finance.income.delete` | D | D | — | ◐ same-period only | — |

Project Managers may **view** income on assigned projects (needed for profitability) but never record or modify it — separation of duties.

### 3.4 Finance — Categories

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View categories | `finance.category.read` | ✓ | ✓ | ✓ | ✓ | — |
| Create custom category | `finance.category.create` | C | C | — | C | — |
| Update / deactivate category | `finance.category.update` | U | U | — | U | — |
| Delete custom category | `finance.category.delete` | D | D | — | — | — |

System categories (`isSystem = true`) can be deactivated but never deleted, by any role.

### 3.5 Employees

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| List / view employees | `employees.employee.read` | ✓ | ✓ | ◐ assigned projects | ✓ | own |
| View salary | `employees.employee.readSalary` | ✓ | ✓ | — | ✓ | own |
| Create employee | `employees.employee.create` | C | C | — | — | — |
| Update employee | `employees.employee.update` | U | U | — | — | — |
| Soft-delete employee | `employees.employee.delete` | D | D | — | — | — |
| Manage departments | `employees.department.manage` | C/U/D | C/U/D | — | — | — |
| View departments | `employees.department.read` | ✓ | ✓ | ✓ | ✓ | ✓ |

Salary is a **field-level** restriction: it is excluded from the query selection for roles without `readSalary`, so it never reaches the client payload.

### 3.6 Clients

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View clients | `organization.client.read` | ✓ | ✓ | ◐ assigned projects | ✓ | — |
| Create client | `organization.client.create` | C | C | — | C | — |
| Update client | `organization.client.update` | U | U | — | U | — |
| Soft-delete client | `organization.client.delete` | D | D | — | — | — |

### 3.7 Documents

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View project documents | `documents.project.read` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Upload project documents | `documents.project.upload` | C | C | ◐ assigned | C | — |
| Delete project documents | `documents.project.delete` | D | D | ◐ own uploads | — | — |
| View employee documents | `documents.employee.read` | ✓ | ✓ | — | ✓ | own |
| Upload employee documents | `documents.employee.upload` | C | C | — | — | — |
| Delete employee documents | `documents.employee.delete` | D | D | — | — | — |
| Attach receipt to a transaction | `documents.finance.upload` | C | C | ◐ assigned | C | — |
| View transaction attachments | `documents.finance.read` | ✓ | ✓ | ◐ assigned | ✓ | — |

Document access always inherits the authorization of the **owning entity**; there is no standalone document permission that can bypass it.

### 3.8 Reports and Exports

| Report | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| Management dashboard (full) | `reporting.dashboard.viewAll` | ✓ | ✓ | — | ✓ finance view | — |
| Project dashboard (scoped) | `reporting.dashboard.viewAssigned` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Project profitability | `reporting.profitability.read` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Budget vs actual | `reporting.budgetVariance.read` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Expense report | `reporting.expense.read` | ✓ | ✓ | ◐ assigned | ✓ | — |
| Cash flow | `reporting.cashflow.read` | ✓ | ✓ | — | ✓ | — |
| Export to Excel / PDF | `reporting.export` | ✓ | ✓ | ◐ assigned data only | ✓ | — |

Exports execute the same scoped query as the on-screen report — an export can never widen access.

### 3.9 Approvals

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View approval inbox | `approvals.request.readInbox` | ✓ | ✓ | — | — | — |
| View own submitted requests | `approvals.request.readOwn` | ✓ | ✓ | own | own | — |
| Decide (approve/reject) | `approvals.request.decide` | A | A | — | — | — |
| Cancel own pending request | `approvals.request.cancelOwn` | ✓ | ✓ | own | own | — |

### 3.10 Notifications

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View own notifications | `notifications.read` | own | own | own | own | own |
| Mark own as read | `notifications.markRead` | own | own | own | own | own |

No role can read another user's notifications.

### 3.11 Audit Logs

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| View full audit log | `audit.log.readAll` | ✓ | ✓ | — | — | — |
| View entity history | `audit.log.readEntity` | ✓ | ✓ | ◐ assigned projects | ✓ finance entities | — |
| Export audit log | `audit.log.export` | ✓ | — | — | — | — |

Audit logs are immutable: **no role** may create, update, or delete them through the application.

### 3.12 Users, Roles, and Settings

| Capability | Permission | Super Admin | Admin/Manager | Project Manager | Accountant | Employee |
|---|---|:--:|:--:|:--:|:--:|:--:|
| List users | `iam.user.read` | ✓ | ✓ | — | — | — |
| Invite / create user | `iam.user.create` | C | C | — | — | — |
| Update user profile | `iam.user.update` | U | U | own | own | own |
| Change user role | `iam.user.changeRole` | U | — | — | — | — |
| Activate / deactivate user | `iam.user.setActive` | U | U | — | — | — |
| Soft-delete user | `iam.user.delete` | D | — | — | — | — |
| Link user to employee | `iam.user.linkEmployee` | U | U | — | — | — |
| View company settings | `organization.settings.read` | ✓ | ✓ | — | ✓ | — |
| Update company settings | `organization.settings.update` | U | — | — | — | — |
| Update approval threshold | `organization.settings.updateThreshold` | U | — | — | — | — |

Rationale: an Admin/Manager runs the business but cannot silently raise the approval threshold or promote themselves — that separation is what makes the approval control meaningful.

---

## 4. Special Rules

1. **Project Manager scoping.** Every project-related read and write is filtered by `projectManagerId = ctx.userId` as a *query condition*. An unassigned project returns `NOT_FOUND`, not `FORBIDDEN`, so project existence is not leaked.
2. **Employee self-scope.** An `EMPLOYEE` resolves to `Employee` via `User.employeeId`. If the link is missing, employee-scoped reads return empty rather than erroring.
3. **No self-approval.** `decidedById <> requestedById` is enforced for `ADMIN_MANAGER`. A `SUPER_ADMIN` may override in a single-manager company; the override is written to `AuditLog` with an explicit marker.
4. **Post-approval immutability.** Once an expense is `APPROVED`, no role may edit its amount, project, scope, or category. Corrections require a reversing entry.
5. **Threshold snapshot.** Approval need is decided against `Expense.thresholdAtSubmission`, so a later settings change never rewrites history.
6. **Salary confidentiality.** Roles without `employees.employee.readSalary` never receive the field — enforced in the Prisma `select`, not by UI hiding.
7. **Deactivated users.** `isActive = false` fails authentication and invalidates sessions regardless of role.
8. **Last Super Admin protection.** The system refuses to deactivate, delete, or demote the final active `SUPER_ADMIN`.
9. **Role change side-effects.** Changing a user's role invalidates their sessions so stale permissions cannot survive in a live tab.
10. **Denial auditing.** Every failed authorization writes `AuditLog(PERMISSION_DENIED)` with actor, permission, and target — repeated denials are a security signal.
11. **Deleted records.** Soft-deleted rows are invisible to all roles through normal flows; only `audit.log.readAll` holders can see that a deletion occurred.
12. **Route guards are not security.** Middleware improves UX; the Server Action check is authoritative.

---

## 5. Scope Resolvers

```ts
type Scope = 'ALL' | 'ASSIGNED_PROJECTS' | 'OWN' | 'NONE';

const projectScope: Record<UserRole, Scope> = {
  SUPER_ADMIN:     'ALL',
  ADMIN_MANAGER:   'ALL',
  PROJECT_MANAGER: 'ASSIGNED_PROJECTS',
  ACCOUNTANT:      'ALL',           // read-only, enforced by permission set
  EMPLOYEE:        'NONE',
};

const employeeScope: Record<UserRole, Scope> = {
  SUPER_ADMIN:     'ALL',
  ADMIN_MANAGER:   'ALL',
  PROJECT_MANAGER: 'ASSIGNED_PROJECTS',   // staff on their projects, no salary
  ACCOUNTANT:      'ALL',
  EMPLOYEE:        'OWN',
};
```

Applied as query filters:

| Scope | Prisma `where` fragment |
|---|---|
| `ALL` | `{ deletedAt: null }` |
| `ASSIGNED_PROJECTS` (project) | `{ deletedAt: null, projectManagerId: ctx.userId }` |
| `ASSIGNED_PROJECTS` (expense) | `{ deletedAt: null, project: { projectManagerId: ctx.userId } }` |
| `OWN` (employee) | `{ deletedAt: null, id: ctx.employeeId ?? '__none__' }` |
| `NONE` | short-circuit → `FORBIDDEN` |

---

## 6. Implementation Contract

```ts
// shared/lib/authz/permissions.ts
export const ROLE_PERMISSIONS: Readonly<Record<UserRole, ReadonlySet<Permission>>>;

export function can(ctx: SessionContext, permission: Permission): boolean;

export function assertPermission(
  ctx: SessionContext,
  permission: Permission,
): asserts ctx is SessionContext;   // throws ForbiddenError, writes audit

export function scopeFor(
  ctx: SessionContext,
  resource: ScopedResource,
): Scope;
```

Testing requirement: a table-driven test asserts **every** role × permission cell in this document, so a change to the matrix that is not reflected here fails CI.
