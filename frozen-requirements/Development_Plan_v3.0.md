# Development Plan v3.0 — Refreshed Architecture-Led Strategy

**Date**: 2026-06-24  
**Status**: Active — v1.5 Final Requirements Frozen — implementation handoff baseline  
**Previous**: `Development_Plan_v2.0.md`  
**Related**: `Wonders_Academy_Role_Permission_Matrix_v1.0.md`, `Initial_School_Setup_and_Principal_Administration_Flow_v1.0.md`

---

## 0. Strategy Change

### Why v3.0?

Sprint 0–3 validated that the vertical-slice strategy works for CRM features. However, **two architectural foundations are missing** that block production deployment:

1. **Workspace & Session Infrastructure** — No `GET /api/session/context`, no workspace routing (`/app/admin/*`), no role-gated APIs
2. **Permission Model** — All CRM controllers are wide open; no `@PreAuthorize`, no data-scope filtering

These are not "nice-to-have" — the `Initial_School_Setup` flow and `Role_Permission_Matrix` both require them as preconditions.

### New Approach: Two Parallel Tracks

```text
Track A — Architecture Foundation (blocks everything else)
  DDL alignment with Permission Matrix
  GET /api/session/context
  Workspace routing /app/admin/*
  @PreAuthorize on all controllers
  Must-change-password flow

Track B — Business Features (continues from v2.0 Sprint 4)
  Remaining CRM frontend migration
  Admin Org CRUD pages (principal-managed persons, members, users, roles)
  Email & Notifications
```

Both tracks run in parallel but **Track A items gate Track B for go-live**.

---

## 1. Current State (June 24)

### Done (Sprint 0–3 + Sprint 4 partial)
- **Backend** (`D:\dev\LLYProjects\ClassingSystem\Backend\BE_Classing`): Spring Boot 3.5 + Kotlin on 8080, PostgreSQL on 5555, Flyway (V0_0→V2_4) boots successfully; DDL not yet aligned with Permission Matrix
- `com.admin.crm.*` — full CRM backend (prospect, tag, communication, conversion, task, marketing, contract, payment, renewal, churn_warning, student, guardian)
- **CRM Frontend** (`D:\dev\CRM`): prospect pages (list + detail) working
- **CRM Frontend**: API client (`client.ts`) updated to point all modules to Spring Boot `/admin/crm/*` paths
- Student facade and student-scoped guardian link management working; standalone Guardian CRUD must be aligned to /api/admin/crm/students/:id/guardians
- Login returns `{ accessToken, user.institutionId }`
- `GET /api/session/context` **does not exist yet**
- `@PreAuthorize` **not applied anywhere**
- Frontend routes are flat `/*`, **no workspace prefix**

### Blocking Gaps
| Gap | Impact |
|-----|--------|
| No `GET /api/session/context` | Frontend cannot determine workspace, role, or permissions |
| No workspace routing | `/app/admin/*` not enforced; all users see all menus |
| No `@PreAuthorize` | Any authenticated user can access any API |
| No `must_change_password` flow | Principal cannot onboard via initial password |
| DB enum mismatch | `workspace_type_enum` has old values; `org_institution_role_enum` missing `sales` |
| `crm.task` missing `institution_id` | Cannot institution-filter tasks without joining to related entity |
| `crm.tag_assignment` missing `institution_id` | Cannot institution-filter tag assignments directly |
| `crm.marketing_activity_participant` missing `institution_id` | Cannot institution-filter participants directly |
| `crm.sales_contract` lacks `UNIQUE(institution_id, contract_no)` | Contract number is global instead of per-institution |
| ~~Conversion routes/permissions — P0 gaps now closed per doc freeze~~ | Resolved — route/permission gaps fixed in all 6 documents |

---

## 2. Revised Sprint Plan

### Sprint 4 — Architecture Foundation + CRM Migration (2 weeks)

**Track A — Architecture Foundation (Week 1–2)**

| Day | Task | Details |
|-----|------|---------|
| 1 | DDL alignment with Permission Matrix | Modify `V0_2__enums.sql`: fix `workspace_type_enum`, `org_institution_role_enum`; add `platform_role_enum`, `auth_account_scope_enum`; add `auth.platform_role_assignment`; modify `auth.user_account` for nullable `institution_id` + `account_scope` |
| 1 | Update `auth.user_last_workspace` | Switch from `login_name + institution_id` → `user_account_id` FK |
| 2 | `POST /api/auth/change-first-time-password` | Already exists in backend; verify frontend integration |
| 2 | `GET /api/session/context` | New endpoint returning `{ current_workspace, available_workspaces, role_codes, permissions_by_workspace, data_scope }` |
| 2 | Login flow: check `must_change_password` | Frontend: detect flag in login response → redirect to change-password page |
| 2–3 | Frontend: `/app/admin/*` workspace routing | Restructure router: `App.tsx` → `WorkspaceLayout` → `AdminLayout` wrapping all existing CRM/org routes |
| 2–3 | Frontend: session context store | `authStore` populates from `GET /api/session/context` after login |
| 3–4 | Frontend: sidebar menu from permissions | Menu items shown/hidden based on `permissions_by_workspace` from session context |
| 4–5 | `@PreAuthorize` — CRM controllers | Apply permission annotations to all `com.admin.crm.*` controllers using `@PreAuthorize` with institution scope validation |

**Track B — Business Features (Week 1–2, parallel)**

| Day | Task | Details |
|-----|------|---------|
| 1–2 | Student page verification | Verify StudentList + StudentDetail work end-to-end with real API |
| 2–3 | Activity pages adaptation | Update `ActivityList`, `ActivityDetail` to use `/app/admin/crm/marketing-activities` paths |
| 3 | Conversion Pipeline page | Verify `ConversionPipeline` against `/app/admin/crm/conversions`; ensure `conversion-stages` API is listed in Route Map §13 |
| 3–4 | Contract pages | Adapt `ContractList`/`ContractDetail` for `/app/admin/crm/sales-contracts` |
| 4 | Task Board | Verify `TaskBoard` against `/app/admin/crm/tasks` |
| 4–5 | Renewal + Warning pages | Adapt `RenewalList` + `WarningBoard` |
| 5 | Tag management UI in sidebar | The existing `DashboardLayout` tag modal — move to proper `/app/admin/crm/tags` page |

**Milestone M4**: All CRM pages work under `/app/admin/crm/*`; backend APIs permission-gated.

---

### Sprint 5 — Admin Organization + Email (1.5 weeks)

| Day | Task | Details |
|-----|------|---------|
| 1–2 | `/app/admin/org/persons` — read + create + modify | List institution persons; simple table with search; create new person; edit existing |
| 2 | `/app/admin/org/persons/:id` — detail view | Person detail + member info + user account status |
| 2–3 | `/app/admin/org/members` — read + create + deactivate | Institution members table; filter by member_type; add member; deactivate |
| 3 | `/app/admin/org/users` — read + invite + reset_password + deactivate | User accounts + password status + last login; invite new user; reset password |
| 3 | `/app/admin/org/roles` — read + assign + revoke | Institution role assignments; assign role to person; revoke role |
| 4 | Email Account CRUD (backend) | `com.email.emailaccount` package — entity, service, controller |
| 4–5 | Email Account frontend page | `/app/admin/email/accounts` — list + create + edit + delete |
| 5 | Email Template + Log read-only pages | `/app/admin/email/templates`, `/app/admin/email/logs` |

**Note:** Unlike the previous "read-only" plan, Sprint 5 now includes **principal-managed CRUD** for Org (persons, members, users, roles). This is required by the Initial School Setup flow — after SaaSOperator creates the first principal, the principal must be able to create and manage their school's internal organization. Read-only would block the onboarding pipeline.

**v1.5.1**: `people_admin` role added — school-level HR / people & account administrator, delegated by principal for day-to-day personnel management. people_admin has limited role assignment scope (sales, marketing, teacher, staff only).

**Milestone M5**: Admin can view org structure, manage people and roles; email accounts can be managed.

---

### Sprint 6 — Email & Notifications Deepening + Go-Live Prep (1 week)

| Day | Task | Details |
|-----|------|---------|
| 1 | Email delivery status page | `/app/admin/email/delivery-status` |
| 2 | Notification inbox (backend) | `com.messaging.notification` CRUD |
| 2–3 | Notification frontend | `/app/admin/notifications/inbox` — list + read + mark read |
| 3 | Broadcast send UI | `/app/admin/notifications/broadcast` — principal, operations, it_admin |
| 4–5 | Cross-module integration | Communication → email log link; prospect convert → notification; contract → renewal trigger |

**Milestone M6**: Email + Notifications operational.

---

### Sprint 7 — Decommission NestJS + Legacy Redirects (1 week)

| Day | Task | Details |
|-----|------|---------|
| 1 | Verify all NestJS endpoints have Spring Boot equivalents | Compare routes across both systems |
| 2 | Remove `apps/server` (NestJS) directory | If all verified |
| 2 | Remove old `com/prospect/` Spring Boot package | If exists and fully migrated |
| 3 | Legacy redirects | `/admin-login.html` → `/app/login` (301 or JS redirect); `/admin-prospect.html` → `/app/admin/crm/prospects` |
| 3 | Full regression test | All 14 CRM modules CRUD via frontend |
| 4 | Audit log viewer | `/app/admin/audit` — read-only table of key events |
| 5 | PWA + responsive polish | Service Worker, manifest, mobile drawer nav |

**Milestone M7**: NestJS fully decommissioned; single unified backend.

---

### Sprint 8+ — Academic / Teacher / Student / Guardian Workspaces

These sprints are unchanged from v2.0 in scope but gain the workspace infrastructure from Sprint 4.

| Sprint | Workspace | Time |
|--------|-----------|------|
| Sprint 8 | Academic Workspace frontend | 3 weeks |
| Sprint 9 | Teacher Workspace frontend | 2 weeks |
| Sprint 10 | Student Workspace frontend | 2 weeks |
| Sprint 11 | Guardian Workspace frontend | 1.5 weeks |
| Sprint 12 | Polish, load test, bug fix | 1 week |

Guardian workspace was not in v2.0 — added per Permission Matrix v1.0 §18.

---

## 3. Updated Gantt

```
Week   1   2   3   4   5   6   7   8   9   10  11  12  13  14  15
       │   │   │   │   │   │   │   │   │   │   │   │   │   │   │
S4-A Foundations  [████████████]  │   │   │   │   │   │   │   │   │
S4-B CRM Migrate       [████████████] │   │   │   │   │   │   │   │
S5 Admin Org+Email         │   [████████████]   │   │   │   │   │
S6 Email+Notify                      [████████]   │   │   │   │
S7 Decommission                               [████]   │   │   │
S8 Academic                                           [████████]   │
S9 Teacher                                                     [████]   │
S10 Student                                                          [████]
S11 Guardian                                                              [███]
S12 Polish                                                                   [██]
       │   │   │   │   │   │   │   │   │   │   │   │   │   │   │
       M4  M5  M6  M7                  M8  M9      M10 M11 M12
```

### Milestones

| Milestone | Week | Delivery |
|-----------|------|----------|
| M4 | 2 | All CRM APIs permission-gated; workspace routing live |
| M5 | 3.5 | Admin Org CRUD (persons, members, users, roles) + Email Account management |
| M6 | 4.5 | Notifications + Email delivery status |
| M7 | 5.5 | NestJS decommissioned; legacy redirects in place |
| M8 | 8.5 | Academic Workspace frontend |
| M9 | 10.5 | Teacher Workspace frontend |
| M10 | 12.5 | Student Workspace frontend |
| M11 | 14 | Guardian Workspace frontend |
| M12 | 15 | Polish complete |

---

## 4. Dependency Map

```text
Sprint 4 Track A (Foundations)
  ├── DDL changes ──→ Session Context ──→ Frontend workspace routing
  ├── Session Context ──→ @PreAuthorize (needs role/permission model)
  └── @PreAuthorize ──→ Sprint 5+ all controllers inherit this

Sprint 4 Track B (CRM Migration)
  └── uses Track A workspace routing from week 2

Sprint 5 (Admin Org + Email)
  ├── Backend: controllers already exist; add @PreAuthorize (needs Track A)
  └── Frontend: uses workspace routing (needs Track A)

Sprint 6+ (Academic / Teacher / Student / Guardian)
  └── All depend on Track A workspace routing + permission model
```

---

## 5. DDL Migration Checklist (from Permission Matrix §6)

### Already Aligned
- `org.person` ✅
- `org.institution_member` ✅
- `org.institution_role_assignment` ✅
- `org.guardian_assignment` ✅
- `auth.user_workspace_binding` ✅ (comments need updating)

### Needs Change
- [ ] `enum.workspace_type_enum`: replace `academic_admin`/`management` → `academic`/`admin`/`guardian`
- [ ] `enum.org_institution_role_enum`: add `sales`, remove `system_maintainer`
- [ ] `enum.platform_role_enum`: new type with `saas_operator`, `system_maintainer`
- [ ] `enum.auth_account_scope_enum`: new type with `institution`, `platform`
- [ ] `auth.user_account`: `institution_id` nullable, add `account_scope`, replace table-level unique constraints with partial unique indexes
- [ ] `auth.platform_role_assignment`: new table
- [ ] `auth.user_last_workspace`: switch to `user_account_id` FK
- [ ] `org.institution_role_assignment` comment update
- [ ] **All `crm.*` operational tables carry `institution_id` NOT NULL** — prospect, student_profile, marketing_activity, marketing_activity_participant, task, conversion, conversion_stage, sales_contract, sales_payment, renewal_reminder, churn_warning, communication, tag, tag_assignment
- [ ] **`crm.student_profile` uses composite PK `(institution_id, person_id)`** — same person can have different CRM profiles across institutions
- [ ] **`crm.sales_contract` uses `UNIQUE(institution_id, contract_no)`** — contract_no is institution-scoped, not globally unique
- [ ] **`crm.communication` has index `(institution_id, entity_type, entity_id)`** — fast multi-tenant polymorphic queries
- [ ] **`email.email_log` moved from V2_1 to V2_2** — placed after `CREATE TABLE email.email_account` so `email_account_id UUID REFERENCES email.email_account(id)` can be defined directly in CREATE TABLE (no ALTER needed); must include index `idx_email_log_email_account`
    - **Implementation QA**: V2_1 must NOT define `email.email_log`; V2_2 must define `email.email_log` with `email_account_id UUID REFERENCES email.email_account(id)` in the CREATE TABLE (no ALTER TABLE).

---

## 6. Role Implementation Order

| Sprint | Roles Covered | Justification |
|--------|--------------|---------------|
| Sprint 4–5 | principal, people_admin, operations, sales, marketing, finance, it_admin | Admin workspace only; people_admin (v1.5.1) handles school-level HR/personnel; finance: read sales_contract + payment permissions in Sprint 5–6, no dedicated finance page before Sprint 7 |
| Sprint 8 | academic_director (academic crossing) | Academic workspace |
| Sprint 9 | teacher | Teacher workspace |
| Sprint 10 | student | Student workspace |
| Sprint 11 | guardian | Guardian workspace |

---

## 7. Risk Register

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| DDL changes break existing seed data | Medium | Write ALTER scripts carefully; test with seed reset |
| Frontend workspace restructuring breaks existing pages | Medium | Keep flat routes as fallback during transition |
| `@PreAuthorize` blocks previously-open APIs | High | Add integration tests for all CRM endpoints |
| Session context endpoint performance (permission computation) | Low | Cache permission results per user per session |
| Guardian workspace not needed for MVP | Low | Defer to Sprint 11; permission matrix already defines it |

---

## 8. Key Metrics

| Metric | Target |
|--------|--------|
| Sprint 4 completion (Track A) | 10 working days |
| Sprint 4 completion (Track B) | 10 working days (parallel) |
| All CRM APIs with `@PreAuthorize` | M4 |
| First workspace routing live | M4 |
| Admin Org CRUD pages | M5 |
| NestJS decommissioned | M7 |
| All 5 workspaces live | M11 |
