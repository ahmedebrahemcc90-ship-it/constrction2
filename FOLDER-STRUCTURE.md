# FOLDER STRUCTURE — Feature-Sliced Design + Modular Monolith

**Status:** Draft for Technical Approval

---

## 1. The Two-Axis Model

A common mistake is forcing pure FSD onto a full-stack Next.js app. We use each methodology where it is strongest:

| Axis | Methodology | Location | Purpose |
|---|---|---|---|
| **Frontend** | Feature-Sliced Design | `src/app`, `src/widgets`, `src/features`, `src/entities`, `src/shared` | UI composition and screen-level reuse |
| **Backend** | Modular Monolith + Clean Architecture | `src/modules/<module>` | Business rules, persistence, authorization |

FSD layers hold **presentation**; `src/modules` holds **business logic**. The only bridge is a Server Action: a `features/*` slice imports a use case from a module's public API and nothing else.

---

## 2. Top-Level Layout

```
constrction2/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed/
│       ├── index.ts
│       ├── company-settings.seed.ts
│       ├── departments.seed.ts
│       ├── expense-categories.seed.ts
│       ├── income-categories.seed.ts
│       └── super-admin.seed.ts
├── messages/
│   ├── ar/{common,auth,projects,finance,employees,reports,approvals,errors}.json
│   └── en/{...same...}
├── public/
├── src/
│   ├── app/          # FSD layer 1 — routing, layouts, providers
│   ├── widgets/      # FSD layer 2 — composite page blocks
│   ├── features/     # FSD layer 3 — user actions (forms, dialogs, server actions)
│   ├── entities/     # FSD layer 4 — business object UI + client types
│   ├── shared/       # FSD layer 5 — framework-agnostic reusables
│   └── modules/      # backend modules (domain / application / infrastructure)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── PRD.md            # product requirements (approved)
├── ARCHITECTURE.md   # this design set — kept at the repo root for visibility
├── ERD.md
├── RBAC-MATRIX.md
├── FOLDER-STRUCTURE.md
├── API-ACTIONS.md
├── ADR.md
├── TASKS.md          # created in Phase 4
├── .env.example
├── eslint.config.mjs
├── tailwind.config.ts
├── next.config.ts
└── tsconfig.json
```

### The import rule

```
app → widgets → features → entities → shared
                   │
                   └──► modules/<m>/index.ts   (public API only)

modules/<m>: application → domain
             infrastructure → (implements) application ports
```

A layer may import only from layers **below** it, never sideways within the same layer, and never upward. Enforced by ESLint `no-restricted-imports` plus `eslint-plugin-boundaries`; violations fail CI.

---

## 3. Layer 1 — `src/app` (routing)

```
src/app/
├── layout.tsx                        # html shell, fonts (Cairo AR / Inter EN)
├── globals.css
├── api/
│   ├── auth/[...all]/route.ts        # Better Auth handler
│   ├── documents/[id]/download/route.ts
│   ├── documents/upload-token/route.ts
│   └── reports/[report]/export/route.ts   # Excel / PDF streaming
└── [locale]/
    ├── layout.tsx                    # NextIntlClientProvider, dir=rtl|ltr
    ├── (auth)/
    │   ├── layout.tsx
    │   ├── sign-in/page.tsx
    │   └── forgot-password/page.tsx
    └── (dashboard)/
        ├── layout.tsx                # session guard + AppShell
        ├── page.tsx                  # dashboard
        ├── projects/
        │   ├── page.tsx
        │   ├── new/page.tsx
        │   └── [projectId]/
        │       ├── page.tsx          # overview + financial summary
        │       ├── expenses/page.tsx
        │       ├── documents/page.tsx
        │       ├── team/page.tsx
        │       └── history/page.tsx
        ├── finance/
        │   ├── expenses/{page.tsx,new/page.tsx,[expenseId]/page.tsx}
        │   ├── income/{page.tsx,new/page.tsx}
        │   └── categories/page.tsx
        ├── employees/{page.tsx,new/page.tsx,[employeeId]/page.tsx}
        ├── clients/page.tsx
        ├── approvals/page.tsx
        ├── reports/
        │   ├── page.tsx
        │   ├── profitability/page.tsx
        │   ├── budget-variance/page.tsx
        │   ├── expenses/page.tsx
        │   └── cash-flow/page.tsx
        ├── notifications/page.tsx
        ├── audit-log/page.tsx
        └── settings/{page.tsx,users/page.tsx,company/page.tsx}
```

Route Handlers exist only for cases Server Actions cannot serve: Better Auth, binary download, upload tokens, and streamed exports.

---

## 4. Layer 2 — `src/widgets` (composite blocks)

A widget assembles features and entities into a self-contained screen block. Widgets do not define business rules.

```
src/widgets/
├── app-shell/            # sidebar, topbar, locale switcher, notification bell
├── dashboard-overview/   # KPI cards, cash-flow chart, recent activity
├── project-financial-summary/
├── project-table/
├── expense-table/
├── income-table/
├── employee-table/
├── approval-inbox/
├── audit-timeline/
└── report-viewer/        # filters + table + export toolbar
```

Example:
```
src/widgets/approval-inbox/
├── ui/ApprovalInbox.tsx
├── ui/ApprovalRow.tsx
├── model/useApprovalFilters.ts
└── index.ts
```

---

## 5. Layer 3 — `src/features` (user actions)

One slice = one thing a user can *do*. This is where Server Actions live, because an action is the boundary between an interaction and a use case.

```
src/features/
├── auth/{sign-in,sign-out}/
├── project/
│   ├── create-project/
│   ├── update-project/
│   ├── change-project-status/
│   ├── delete-project/
│   ├── assign-project-manager/
│   ├── request-budget-change/
│   └── manage-project-team/
├── expense/
│   ├── create-expense/
│   ├── update-expense/
│   ├── delete-expense/
│   └── filter-expenses/
├── income/{create-income,update-income,delete-income}/
├── approval/{decide-approval,cancel-approval-request}/
├── employee/{create-employee,update-employee,delete-employee,assign-employee-to-project}/
├── client/{create-client,update-client}/
├── document/{upload-document,delete-document}/
├── notification/{mark-notification-read}/
├── report/{run-report,export-report}/
└── settings/{update-company-settings,manage-users}/
```

Canonical slice:

```
src/features/expense/create-expense/
├── ui/
│   ├── CreateExpenseDialog.tsx      # client component
│   └── ExpenseForm.tsx              # RHF + zodResolver
├── model/
│   ├── create-expense.action.ts     # "use server" — the only backend touchpoint
│   └── use-create-expense.ts        # client hook: submit, toast, router refresh
└── index.ts                         # exports CreateExpenseDialog only
```

```ts
// src/features/expense/create-expense/model/create-expense.action.ts
'use server';

import { createExpenseUseCase, createExpenseSchema } from '@/modules/finance';
import { getSessionContext } from '@/modules/iam';
import type { Result } from '@/shared/lib/result';

export async function createExpenseAction(
  input: unknown,
): Promise<Result<{ id: string; reference: string }>> {
  const ctx = await getSessionContext();
  if (!ctx) return { ok: false, error: { code: 'UNAUTHENTICATED', messageKey: 'errors.auth.required' } };

  const parsed = createExpenseSchema.safeParse(input);
  if (!parsed.success) {
    return {
      ok: false,
      error: {
        code: 'VALIDATION',
        messageKey: 'errors.validation.failed',
        fieldErrors: parsed.error.flatten().fieldErrors,
      },
    };
  }

  return createExpenseUseCase(ctx, parsed.data);
}
```

The action does three things only: authenticate, validate, delegate. No Prisma, no business rules.

---

## 6. Layer 4 — `src/entities` (business object UI)

Reusable, action-free presentation of a business object.

```
src/entities/
├── project/     ui/ProjectCard.tsx, ui/ProjectStatusBadge.tsx, lib/projectStatus.ts
├── expense/     ui/ExpenseRow.tsx, ui/ApprovalStatusBadge.tsx
├── income/      ui/IncomeRow.tsx
├── employee/    ui/EmployeeCard.tsx, ui/EmployeeStatusBadge.tsx
├── client/      ui/ClientCard.tsx
├── user/        ui/UserAvatar.tsx, ui/RoleBadge.tsx
├── approval/    ui/ApprovalStatusBadge.tsx, ui/ApprovalTypeLabel.tsx
└── notification/ui/NotificationItem.tsx
```

Entities render props; they never call Server Actions and never fetch.

---

## 7. Layer 5 — `src/shared`

```
src/shared/
├── ui/                    # shadcn/ui primitives + thin wrappers
│   ├── button.tsx, input.tsx, dialog.tsx, table.tsx, ...
│   ├── data-table/        # sorting, pagination, empty state
│   ├── money-input.tsx    # decimal-safe, currency-aware
│   ├── date-picker.tsx    # DD/MM/YYYY, RTL-aware
│   └── page-header.tsx
├── lib/
│   ├── result.ts          # Result<T>, AppError
│   ├── money.ts           # Decimal helpers, formatting, no float math
│   ├── dates.ts           # date-fns + Africa/Cairo period boundaries
│   ├── format.ts          # Intl number/currency per locale
│   ├── cn.ts
│   └── authz/             # Permission type, ROLE_PERMISSIONS, can(), scopes
├── config/
│   ├── env.ts             # Zod-validated environment variables
│   ├── navigation.ts      # sidebar items + required permission
│   └── constants.ts
├── i18n/
│   ├── routing.ts, request.ts, locales.ts
│   └── direction.ts
├── hooks/                 # useDebounce, usePermission, usePagination
└── types/                 # global TS types, Prisma re-exports
```

`shared` may not import from any other layer — it is the dependency sink.

---

## 8. Backend — `src/modules`

```
src/modules/
├── iam/
│   ├── domain/            role.ts, permission.ts
│   ├── application/       get-session-context.ts, assert-permission.ts, use-cases/
│   ├── infrastructure/    better-auth.config.ts, prisma-user.repository.ts
│   └── index.ts
├── organization/          # CompanySetting, Department, Client
├── projects/
│   ├── domain/            project.rules.ts, project-status.machine.ts, errors.ts
│   ├── application/
│   │   ├── schemas/       create-project.schema.ts, update-project.schema.ts
│   │   ├── use-cases/     create-project.ts, request-budget-change.ts, ...
│   │   └── ports/         project.repository.ts
│   ├── infrastructure/    prisma-project.repository.ts, project-code.generator.ts
│   └── index.ts
├── finance/
│   ├── domain/            expense.rules.ts, money.ts, threshold.ts
│   ├── application/       schemas/, use-cases/, ports/
│   ├── infrastructure/    prisma-expense.repository.ts, prisma-income.repository.ts
│   └── index.ts
├── approvals/
│   ├── domain/            approval.machine.ts, self-approval.policy.ts
│   ├── application/       use-cases/{request,decide,cancel}.ts
│   ├── infrastructure/    prisma-approval.repository.ts
│   └── index.ts
├── employees/
├── documents/             blob.storage.ts, mime-allowlist.ts
├── notifications/         notification.factory.ts, recipient-resolver.ts
├── audit/                 audit.logger.ts (insert-only), redact.ts
├── reporting/
│   ├── application/       profitability.query.ts, cash-flow.query.ts, ...
│   └── infrastructure/    excel.exporter.ts, pdf.exporter.tsx
└── shared-kernel/
    ├── prisma.ts          # singleton client + soft-delete helper
    ├── transaction.ts     # runInTransaction(ctx, fn)
    └── types.ts           # SessionContext, Paginated<T>, DateRange
```

### Public API example

```ts
// src/modules/finance/index.ts
export { createExpenseUseCase } from './application/use-cases/create-expense';
export { updateExpenseUseCase } from './application/use-cases/update-expense';
export { listExpensesQuery } from './application/use-cases/list-expenses';
export { getProjectCostSummary } from './application/use-cases/get-project-cost-summary';

export { createExpenseSchema, updateExpenseSchema } from './application/schemas';
export type { ExpenseListItem, ProjectCostSummary } from './application/dto';
// Prisma models, repositories, and domain internals are intentionally NOT exported.
```

### Use case example (the transaction + audit pattern)

```ts
// src/modules/finance/application/use-cases/create-expense.ts
export async function createExpenseUseCase(
  ctx: SessionContext,
  input: CreateExpenseInput,
): Promise<Result<{ id: string; reference: string }>> {
  assertPermission(
    ctx,
    input.scope === 'PROJECT'
      ? 'finance.expense.createProject'
      : 'finance.expense.createGeneral',
  );

  const settings = await organization.getCompanySettings();
  const decision = decideApprovalNeed(input.amount, settings.largeExpenseThreshold);

  return runInTransaction(async (tx) => {
    const expense = await expenseRepo(tx).create({ ...input, ...decision, submittedById: ctx.userId });

    if (decision.requiresApproval) {
      await approvals.requestExpenseApproval(tx, ctx, expense);
    }
    await audit.record(tx, ctx, { eventType: 'CREATED', entityType: 'EXPENSE', entityId: expense.id, summary: ... });
    await notifications.onExpenseCreated(tx, expense);

    return { ok: true, data: { id: expense.id, reference: expense.reference } };
  });
}
```

---

## 9. Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Folders | kebab-case | `create-expense/` |
| React components | PascalCase | `ExpenseForm.tsx` |
| Hooks | camelCase `use*` | `useCreateExpense.ts` |
| Server Actions | `*.action.ts`, verb + `Action` | `createExpenseAction` |
| Use cases | `*.ts`, verb + `UseCase` | `createExpenseUseCase` |
| Zod schemas | `*.schema.ts`, `<verb><Entity>Schema` | `createExpenseSchema` |
| Repositories | `prisma-<entity>.repository.ts` | `prisma-expense.repository.ts` |
| Domain rules | `<entity>.rules.ts` | `expense.rules.ts` |
| Types | PascalCase, no `I` prefix | `ExpenseListItem` |
| i18n keys | dot.case by module | `finance.expense.form.amount` |

---

## 10. Path Aliases and Boundary Enforcement

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "paths": {
      "@/app/*":      ["./src/app/*"],
      "@/widgets/*":  ["./src/widgets/*"],
      "@/features/*": ["./src/features/*"],
      "@/entities/*": ["./src/entities/*"],
      "@/shared/*":   ["./src/shared/*"],
      "@/modules/*":  ["./src/modules/*"]
    }
  }
}
```

ESLint boundary rules:

1. `shared/**` may not import `entities|features|widgets|app|modules`.
2. `entities/**` may import only `shared`.
3. `features/**` may import `entities`, `shared`, and `@/modules/<m>` (root only).
4. `widgets/**` may import `features`, `entities`, `shared`.
5. `modules/**` may not import `app|widgets|features|entities`.
6. `modules/*/domain/**` may not import Prisma, Next.js, or React.
7. Deep module imports (`@/modules/finance/infrastructure/...`) are forbidden outside that module.

---

## 11. Why This Structure

| Decision | Reasoning |
|---|---|
| FSD for UI only | FSD solves UI coupling. Applying it to persistence would scatter one table's logic across many slices. |
| Modules for business logic | A table and its rules live in exactly one place — this is what makes the monolith *modular*. |
| Server Actions inside feature slices | Colocating the action with the form that calls it keeps a vertical slice reviewable in one folder. |
| Module public APIs | Refactoring internals never breaks consumers; boundaries are mechanically enforceable. |
| Domain free of IO | Business rules stay unit-testable in milliseconds without a database. |
| Repository ports | Swapping Prisma or adding a cache later is an infrastructure change, not a rewrite. |
| Route Handlers only where required | Avoids maintaining a parallel REST surface with duplicated validation. |
| Phase-2 readiness | Adding `inventory` or `suppliers` means one new `modules/` folder plus its slices — no existing module changes. |
