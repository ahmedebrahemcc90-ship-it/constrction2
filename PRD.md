# Product Requirements Document (PRD)

## 1. Document Overview

**Product:** Construction Company Management ERP  
**Release:** MVP  
**Status:** Draft for Product Approval  
**Primary market:** Small and medium construction companies in Egypt and the Arab region  
**Default locale:** Arabic (Egypt), with English support  
**Default currency:** EGP  

This MVP is a modular monolith for one company and one branch. It starts with the operational needs of a small construction company and allows optional capabilities to be added as the company grows.

## 2. Problem Statement

Small and medium construction companies commonly manage projects, expenses, payments, employees, and supporting documents using disconnected Excel files, paper forms, notebooks, and WhatsApp messages.

This creates several problems:

- Management cannot quickly determine whether a project is profitable.
- Project expenses are difficult to separate from general company expenses.
- Planned budgets are not compared consistently with actual costs.
- Project status and financial information are scattered across multiple sources.
- Owners and managers lack timely, reliable reports.
- Approval decisions and sensitive changes are difficult to audit.
- Contracts, receipts, invoices, and site photos are difficult to find.

The product will provide one simple, secure system for the company owner, project managers, accountant, and employees to manage the essential operational and financial information of the business.

## 3. Product Goals

### 3.1 Primary Goals

1. Give owners and managers a reliable overview of active projects, cash flow, and recent expenses.
2. Make project profitability and budget variance visible without manual spreadsheet consolidation.
3. Centralize project, finance, employee, and supporting-document data.
4. Provide controlled access based on predefined roles.
5. Record sensitive actions and approval history for accountability.
6. Support Arabic RTL usage while providing English localization.
7. Keep the initial workflow simple enough for small companies and extensible for medium companies.

### 3.2 Success Metrics

The MVP will be considered successful when, during an initial validation period:

- At least 80% of active projects have a complete project profile, assigned manager, and planned budget.
- At least 90% of project-related expenses are recorded against the correct project.
- Management can produce a project profitability report in under two minutes.
- Management can produce a monthly cash-flow report in under two minutes.
- At least 80% of finance users' routine records require no spreadsheet duplication.
- 100% of budget changes and large-expense approvals are recorded in the audit history.
- At least 90% of invited users can complete their primary workflow without assistance after onboarding.
- No critical or high-severity security issue remains open at MVP release.
- The application works on supported desktop and mobile browser sizes in Arabic and English.

These metrics will be validated with a pilot company using realistic Egyptian construction data.

## 4. User Personas

### 4.1 Owner / General Manager

**Profile:** Owns or manages a small or medium construction company and needs a fast view of performance.  
**Goals:** Understand project profitability, approve significant decisions, monitor cash flow, and identify budget problems.  
**Needs:** Dashboard, reports, project overview, approval queue, audit history.  
**Example:** Reviews whether a residential finishing project is still profitable after labor and material-related expenses.

### 4.2 Project Manager

**Profile:** Responsible for one or more assigned construction projects.  
**Goals:** Keep project information current, monitor the project budget, and record project expenses.  
**Needs:** Assigned-project access, project details, expense entry, documents, photos, budget alerts.  
**Example:** Uploads a site photo and records a 7,500 EGP transportation expense against a project.

### 4.3 Accountant

**Profile:** Records company income and expenses and prepares management reports.  
**Goals:** Maintain accurate financial records and explain project costs and cash movement.  
**Needs:** Finance transactions, categories, project association, reports, exports, audit information.  
**Example:** Records a client payment and classifies an office rent payment as a general expense.

### 4.4 Employee

**Profile:** A regular company employee who needs limited access to their own information.  
**Goals:** View their personal profile and assignment information.  
**Needs:** Read-only access to permitted personal data.  
**Example:** Confirms their job title and current project assignment.

### 4.5 Super Admin

**Profile:** Technical or business administrator responsible for the system setup.  
**Goals:** Manage users, roles, and system-level settings.  
**Needs:** Full access, including administrative configuration and security-sensitive operations.

## 5. MVP Scope and Core Features

### 5.1 Authentication and Role-Based Access Control

- Secure login and session management using Better Auth.
- Predefined roles only:
  - Super Admin
  - Admin/Manager
  - Project Manager
  - Accountant
  - Employee
- Enforce authorization on the server for every protected operation.
- Project Managers can access only projects assigned to them.
- Employees can view only permitted personal information.
- No custom roles in the MVP.

### 5.2 Management Dashboard

Display role-appropriate information, including:

- Number of active, completed, and paused projects.
- Planned project budgets and actual project costs.
- Current-period income and expenses.
- Simplified cash-flow summary.
- Recent expenses and payments.
- Pending large-expense and budget-change approvals.
- Budget-overrun alerts.
- Recent security/audit activity for authorized users.

The dashboard must not expose financial data to roles that are not authorized to view it.

### 5.3 Projects

Users with permission can:

- Create, view, update, and soft-delete projects.
- Store project name, location, description, start date, expected end date, and status.
- Store basic client information.
- Store contract reference and planned budget.
- Assign a Project Manager.
- View project income, expenses, actual cost, variance, and estimated profitability.
- Upload and download project contracts, receipts, invoices, photos, and related documents.
- View project activity and approval history.

Initial statuses: **Planned, Active, Paused, Completed, Cancelled**.

MVP does not include tasks, milestones, BOQ, detailed progress tracking, change orders, or client approvals.

### 5.4 Finance

#### Income and Payments

- Record client payments and other company income.
- Store amount, date, description, category/type, reference, and optional project association.
- Display project-associated income in project profitability calculations.

#### Expenses

- Record general company expenses and project-specific expenses.
- Store amount, date, description, category, payment method/reference, optional notes, and optional project association.
- Allow receipt or invoice attachment.
- Identify expenses requiring approval.
- Prevent unauthorized users from approving their own restricted transactions where separation is required by the role policy.

#### Financial Views

- Total income and expenses by period.
- Project cost totals.
- Planned budget versus actual cost.
- Basic cash flow: income minus expenses for a selected period.
- Project profitability: project income minus project expenses.

MVP is not a full accounting ledger and does not include tax automation, bank reconciliation, accounts payable, or accounts receivable.

### 5.5 Employees

Authorized users can:

- Create, view, update, and soft-delete employee records.
- Store name, phone, job title, department, salary, employment status, and basic contact information.
- Assign employees to projects.
- Upload permitted employee documents.
- Restrict employees to their own permitted profile data.

MVP does not include attendance, payroll processing, leave management, overtime, certifications, or performance management.

### 5.6 Reports and Exports

Required reports:

1. Management dashboard summary.
2. Project profitability.
3. Project budget versus actual cost.
4. Expense report by date range, category, and project.
5. Monthly cash flow.

Report capabilities:

- Filter by date range, project, status, category, and other relevant dimensions.
- Respect role-based data visibility.
- Show totals and clear empty states.
- Export authorized report results to Excel and PDF.
- Use EGP formatting by default.

### 5.7 Documents and File Attachments

MVP supports basic upload, download, and authorized access for:

- Contracts.
- Invoices and receipts.
- Project/site photos.
- Employee documents.

Files will use Vercel Blob Storage in the target deployment. File metadata and authorization must be stored separately from the binary object. File type, size, and access must be validated on the server.

### 5.8 Notifications

In-app notifications will be generated for:

- A new project being created.
- A large expense requiring approval.
- A project budget being exceeded.
- A budget change requiring approval.

Email, SMS, WhatsApp, and push notifications are out of scope for MVP.

### 5.9 Audit Logs

Record audit events for sensitive actions, including:

- Project creation, update, and deletion.
- Expense and income creation, update, approval, rejection, and deletion.
- Budget creation and changes.
- User and role changes.
- Approval decisions.
- File access or deletion where required by the security policy.

Each relevant event should include actor, action, entity, entity identifier, timestamp, and a safe summary of the change. Secrets and sensitive authentication data must never be logged.

## 6. Business Rules and Validations

### 6.1 General Rules

- Every protected operation must be authorized server-side; client-side checks are not sufficient.
- All user-entered values must be validated on both client and server.
- Soft-deleted records must not appear in normal lists or calculations.
- Financial amounts must be greater than zero unless an explicitly supported correction workflow is introduced later.
- Monetary values use two-decimal precision internally and display the configured currency.
- Dates must be valid and follow the configured timezone, defaulting to Africa/Cairo.
- Records must retain `createdAt`, `updatedAt`, creator, and last updater information.
- Duplicate accidental submissions must be prevented where practical.
- Destructive operations require confirmation and must be auditable.

### 6.2 Project Rules

- Project name is required and must be meaningful after trimming whitespace.
- Planned budget is required for a financially tracked project and must be greater than 0 EGP.
- Project start date cannot be later than the expected end date.
- A Project Manager assignment is required for an active project.
- A project cannot be marked Completed if required project financial data is invalid or incomplete.
- Project Managers may modify only assigned projects and permitted fields.
- A project budget increase or decrease requires Owner/General Manager approval.
- Budget changes must preserve the previous value, requested value, requester, approver, timestamp, and decision.

### 6.3 Expense Rules

- Expense amount, date, description, and category are required.
- Project-specific expenses must reference an active or otherwise valid project.
- General expenses must not be assigned to a project.
- An expense greater than **10,000 EGP** requires approval by the Owner/General Manager.
- The threshold is configurable by an authorized administrator but defaults to 10,000 EGP.
- A large expense cannot be treated as fully approved until an authorized approver approves it.
- Approval and rejection decisions require an audit record and optional reason.
- The requester must not approve their own large expense when the role policy requires independent approval.
- Corrected or cancelled transactions must preserve the original audit trail.

### 6.4 Financial Calculation Rules

- Project actual cost equals the sum of approved project-specific expenses, subject to the selected reporting policy.
- Project profitability equals project-associated income minus project actual cost.
- Budget variance equals planned budget minus actual project cost.
- A negative budget variance indicates that actual cost has exceeded the planned budget.
- Cash flow for a period equals recorded income minus recorded expenses for that period.
- Reports must clearly identify whether pending/unapproved transactions are excluded or separately displayed.

### 6.5 File Rules

- Only approved file types and file sizes may be uploaded.
- File names must be normalized and must not be used directly to create executable paths.
- Users may download files only when their role can access the related entity.
- Deleted entity records must not expose their files through normal application flows.

## 7. Key User Flows

### 7.1 Create a Project

1. Admin/Manager opens Projects and selects **Create Project**.
2. User enters name, location, client information, dates, status, contract reference, and planned budget.
3. User assigns a Project Manager.
4. Client and server validation runs.
5. The system creates the project and records an audit event.
6. Authorized users receive an in-app notification.
7. The assigned Project Manager can open the project and add permitted documents or expenses.

### 7.2 Record a Project Expense

1. Accountant or authorized Project Manager opens the expense form.
2. User selects project-specific expense, project, category, date, amount, and description.
3. User optionally attaches a receipt.
4. The system validates the input and checks the user's project/finance permission.
5. If the amount is greater than 10,000 EGP, the expense enters a pending-approval state and the approver receives an in-app notification.
6. If approved, it is included in approved project cost and reports.
7. The system records creation and approval history in the audit log.

### 7.3 Record a General Expense

1. Accountant opens the expense form.
2. User selects general expense and enters category, amount, date, description, and optional attachment.
3. The system confirms that no project is selected.
4. If the threshold is exceeded, the expense requires approval.
5. Approved expense data appears in cash-flow and expense reports but not in project cost totals.

### 7.4 Approve a Large Expense

1. Owner/General Manager opens the approval notification or approval list.
2. The system displays the expense details, requester, attachment, and audit history.
3. Approver selects Approve or Reject and may enter a reason.
4. The system records the decision and timestamp.
5. The requester receives an in-app notification.
6. Approved expenses become eligible for the configured financial reports.

### 7.5 Request a Budget Change

1. An authorized user opens a project and submits a new planned budget with a reason.
2. The system validates that the new budget is greater than zero.
3. The project remains on its previous approved budget until approval.
4. Owner/General Manager reviews the request.
5. On approval, the new budget becomes active and the old value remains in history.
6. On rejection, the old budget remains active and the reason is recorded.

### 7.6 Review Project Profitability

1. Owner/Manager or authorized Accountant opens Reports.
2. User selects a project and optional date range.
3. The system displays project income, approved project costs, planned budget, variance, and profitability.
4. The report clearly separates pending transactions from approved transactions.
5. User exports the result to PDF or Excel if needed.

### 7.7 Upload a Project Document

1. Authorized user opens the project Documents section.
2. User selects a supported file and document category.
3. The server validates file type, size, authorization, and metadata.
4. The file is stored in Vercel Blob and metadata is associated with the project.
5. The upload is recorded in the audit log.

## 8. Non-Functional Requirements

### 8.1 Security

- Apply OWASP Top 10 practices.
- Use server-side authorization for all data access and mutations.
- Validate and sanitize all inputs.
- Protect sessions and avoid exposing secrets to the client.
- Do not log passwords, tokens, or sensitive credentials.
- Enforce secure file upload and download controls.

### 8.2 Performance

- Dashboard and standard report pages should load within an acceptable interactive time for the expected MVP dataset.
- Lists and reports must use pagination or bounded queries where appropriate.
- Database queries must avoid unnecessary N+1 access patterns.

### 8.3 Usability and Accessibility

- Responsive desktop and mobile web interface.
- Arabic RTL and English LTR layouts.
- Clear validation messages in the active language.
- Consistent empty, loading, success, and error states.
- Keyboard-accessible primary workflows and usable contrast.

### 8.4 Reliability and Maintainability

- Use a modular monolith with clear module boundaries.
- Keep business rules in reusable server-side domain/application logic rather than only in UI components.
- Use TypeScript throughout the application.
- Use structured error handling and safe user-facing messages.
- Support database migrations and environment-based configuration.
- Preserve auditability and data integrity when records are changed or soft-deleted.

## 9. Assumptions

- MVP serves one company and one branch; multi-tenant support is not required.
- The initial deployment uses Vercel, PostgreSQL through Vercel Postgres or Neon, and Vercel Blob.
- Users have modern browsers and internet access.
- The company will initially enter existing data manually; migration from Excel is not part of MVP unless separately approved.
- EGP is the default currency and Egypt/Africa-Cairo are the default locale and timezone.
- The company defines its own expense categories and may use an initial system-provided category set.
- Financial reports depend on users entering complete and accurate transactions.
- Full legal, tax, and statutory accounting compliance is not a requirement for this MVP.
- The 10,000 EGP large-expense threshold is an initial business default and may be configured by an authorized administrator.
- Vercel Blob is available and properly configured in production; local file storage is not the production target.

## 10. Out of Scope

The following are explicitly excluded from MVP:

- Multi-tenant architecture.
- Multiple branches.
- Inventory, materials, warehouses, and equipment management.
- Suppliers and subcontractors.
- Purchase orders.
- Invoicing, accounts receivable, and accounts payable.
- Attendance, payroll, leave, overtime, and performance management.
- Tax/VAT filing automation.
- Full accounting/general ledger.
- Advanced approval workflows.
- BOQ and detailed cost breakdown structures.
- Tasks, milestones, and Gantt-style planning.
- Client portal and client approvals.
- Email, SMS, WhatsApp, and push notifications.
- 2FA, device management, and advanced backup/restore.
- Real-time collaboration.
- AI/ML features.
- Native mobile applications.
- Public marketing website.
- CRM, sales pipeline, and lead management.
- Advanced equipment maintenance.
- Blockchain or cryptocurrency payments.

## 11. MVP Definition of Done

The MVP is ready for pilot use when all of the following are true:

### Product Functionality

- Authentication and all five MVP roles work end-to-end.
- Server-side RBAC prevents unauthorized data access and mutations.
- Projects can be created, updated, viewed, assigned, soft-deleted, and financially tracked.
- Income, general expenses, and project expenses can be recorded with validation.
- Large expenses and budget changes support the agreed approval flow.
- Employee profiles and project assignments work end-to-end.
- Required dashboard and reports calculate the documented financial metrics correctly.
- Reports export successfully to Excel and PDF.
- Authorized users can upload and download supported project and employee documents.
- In-app notifications work for the defined MVP events.
- Audit logs capture all required sensitive actions.

### Quality and Security

- Arabic RTL is the default interface and English localization is functional.
- Responsive layouts work on supported desktop and mobile viewport sizes.
- Loading, empty, validation, success, and error states are implemented for primary workflows.
- Automated tests cover critical business rules, authorization, financial calculations, and approval behavior.
- Database migrations and seed data can be applied reliably in a clean environment.
- Production configuration uses environment variables and does not expose secrets.
- File uploads are validated and access-controlled.
- No known critical or high-severity security defects remain open.

### Acceptance Validation

- A pilot user can create a project, assign a manager, record income and expenses, approve a large expense, review profitability, and export a report without manual database intervention.
- A Project Manager cannot view or modify an unassigned project.
- An Employee cannot access financial data or another employee's private information.
- A budget change and a large-expense approval show complete history in the audit log.
- The product owner approves the final MVP against the agreed scope and this document.

## 12. Future Roadmap Summary

After MVP validation, the product may add:

- Inventory and materials.
- Suppliers and purchase orders.
- Attendance and basic payroll.
- Multi-branch support.
- Invoicing and receivables/payables.
- Email and WhatsApp notifications.
- BOQ, milestones, progress tracking, and advanced analytics.
- Client portal, native mobile application, and accounting integrations.

---

**Approval Required:** Product owner review and written approval of this PRD before proceeding to Phase 3 — Technical Design.
