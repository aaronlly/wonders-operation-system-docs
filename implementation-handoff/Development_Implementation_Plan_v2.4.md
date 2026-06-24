# Wonders Operation System — Development Plan v2.4 (Revised)

> **Baseline**: Authorization Meta Model v2.4 + UI IA Spec v2.4-aligned + 11 Role Specs
> **Last Updated**: 2026-06-24

---

## Done ✅

| Layer | Item |
|-------|------|
| Backend | 35 repos, 74 @PreAuthorize, v2.4 Flyway, SessionContext, CORS |
| Backend | people_admin role + OrgMemberService + email/broadcast controllers |
| Frontend | Login (matches old design), Top Bar (sticky dark), grouped sidebar |
| Frontend | Dashboard (per-role stats), ProspectList (full filters + table) |
| Frontend | 14 CRM pages, email accounts/logs, member create, notifications |
| Docs | Authorization Meta Model, 11 role specs, UI IA Spec, DEBUG_GUIDE |
| Infra | Docker compose, start-all.ps1, health-check, reset-db |

---

## Phase 1: Backend Gaps (Week 1)

### 1.1 Rebuild + Deploy
| # | Task |
|---|------|
| 1.1.1 | Rebuild backend JAR with updated SessionContextController (principal org/email/notification/audit perms) |

### 1.2 Org CRUD APIs
| # | Endpoint | Controller |
|---|----------|-----------|
| 1.2.1 | `GET/POST/PUT /api/admin/org/persons` | new OrgPersonController |
| 1.2.2 | `GET/PUT /api/admin/org/persons/:id` | ↑ |
| 1.2.3 | `GET/POST /api/admin/org/users` | new OrgUserController |
| 1.2.4 | `POST /api/admin/org/users/:id/reset-password` | ↑ |
| 1.2.5 | `PUT /api/admin/org/users/:id/deactivate` | ↑ |
| 1.2.6 | `GET /api/admin/org/roles` | new OrgRoleController |
| 1.2.7 | `POST /api/admin/org/roles/assign` | ↑ |
| 1.2.8 | `DELETE /api/admin/org/roles/:id` | ↑ |

### 1.3 Messages API
| # | Endpoint |
|---|----------|
| 1.3.1 | `GET /api/admin/messages/inbox` |
| 1.3.2 | `GET /api/admin/messages/:id` |
| 1.3.3 | `POST /api/admin/messages` (compose) |

### 1.4 Audit API
| # | Endpoint |
|---|----------|
| 1.4.1 | `GET /api/admin/audit` (school-scoped, read-only) |

---

## Phase 2: Frontend Placeholder → Real Pages (Week 1-2)

| # | Page | Route | Source of truth |
|---|------|-------|-----------------|
| 2.1 | `OrgPersonList.tsx` | `/app/admin/org/persons` | IA Spec §2.5 Org |
| 2.2 | `OrgUserList.tsx` | `/app/admin/org/users` | IA Spec §2.5 Org |
| 2.3 | `OrgRoleList.tsx` | `/app/admin/org/roles` | IA Spec §2.5 Org |
| 2.4 | `MessageInbox.tsx` | `/app/admin/messages/inbox` | IA Spec §2.5 Messages |
| 2.5 | `MessageDetail.tsx` | `/app/admin/messages/:id` | IA Spec §2.5 Messages |
| 2.6 | `EmailTemplateList.tsx` | `/app/admin/email/templates` | IA Spec §2.5 Email |
| 2.7 | `AuditLogList.tsx` | `/app/admin/audit` | IA Spec §2.5 Audit |

---

## Phase 3: Missing Admin Features (Week 2-3)

| # | Feature | Description |
|---|---------|-------------|
| 3.1 | Principal credential flow | SaaSOperator: show generated password in UI (not only email) |
| 3.2 | must_change_password | Login → detect flag → force password change page |
| 3.3 | Password change page | `/app/admin/password-change` form |
| 3.4 | Email send from ProspectList | Hook "发送邮件" button to backend email API |
| 3.5 | Batch export | Prospect export CSV/Excel |
| 3.6 | Notification inbox | Replace placeholder with real notification list |

---

## Phase 4: Academic Workspace (Sprint 8, deferred)

| # | Task |
|---|------|
| 4.1 | Backend: scheduling_run, proposal, baseline, planned/class session controllers |
| 4.2 | Frontend: academic workspace layout + sidebar (per IA Spec §2.1) |
| 4.3 | Academic Dashboard page |
| 4.4 | 16 academic pages (resources, classes, scheduling, execution, review, settings) |

---

## Phase 5: Teacher / Student / Guardian (Sprints 9-11, deferred)

| Sprint | Workspace | # Pages |
|--------|-----------|:---:|
| 9 | Teacher | 12 |
| 10 | Student | 12 |
| 11 | Guardian | 10 |

---

## Milestones

| M# | Phase | Deliverable |
|----|-------|-------------|
| M1 | 1.1 | Backend rebuilt with full principal perms |
| M2 | 1.2-1.4 | Org CRUD + Messages + Audit APIs |
| M3 | 2 | 7 placeholder pages → real pages |
| M4 | 3 | Principal credential, password change, email send, export, notifications |
| M5 | 4 | Academic workspace (Sprint 8) |
| M6 | 5 | Teacher / Student / Guardian |

---

## Current Priority: M1 → M2

Rebuild backend JAR → verify principal sees Org/Email/Notifications/Audit in sidebar.
Then build Org CRUD APIs → then Org frontend pages.
