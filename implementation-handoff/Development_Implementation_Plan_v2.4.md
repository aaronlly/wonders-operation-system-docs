# Wonders Operation System — Development Implementation Plan v2.4

> **Baseline**: Authorization Meta Model v2.4 + 11 Role Functional Specs + UI IA Spec v2.4-aligned
> **Status**: Draft for implementation

---

## Current State

| Layer | Done | Remaining |
|-------|------|-----------|
| Backend controllers | 21 controllers, 74 @PreAuthorize | Org/role management APIs, Messages API, Audit API |
| Flyway migrations | v2.4 (8 files) | — |
| Frontend pages | 19 React pages under /app/admin/* | Dashboard, org/members detail, org/users, org/roles, messages, audit |
| Sidebar | Flat menu, no role filtering | Grouped menu, per-role visibility |
| Login | Unified page matching old design | must_change_password redirect, workspace routing after login |
| Docs | Authorization Model + 11 specs + UI IA | — |

---

## Phase 1: Admin Workspace UI Restructuring (Week 1)

### 1.1 Dashboard Page

| Task | File | Description |
|------|------|-------------|
| 1.1.1 | `pages/Dashboard.tsx` (new) | Stats cards + todo list + quick actions |
| 1.1.2 | Route | `App.tsx`: add `<Route path="dashboard" element={<Dashboard />} />` |
| 1.1.3 | Per-role content | Dashboard shows role-specific stats (per §2.6) |

### 1.2 Sidebar Grouping + Role Visibility

| Task | File | Description |
|------|------|-------------|
| 1.2.1 | `DashboardLayout.tsx` | Group menu into 4 sections: Messages / Org / CRM / Email / Notifications |
| 1.2.2 | `DashboardLayout.tsx` | Filter menu items by `useAuthStore.hasPermission(object, action)` |
| 1.2.3 | `DashboardLayout.tsx` | Add Top Bar: school name + notification bell + user menu |

### 1.3 Login → Role Routing

| Task | File | Description |
|------|------|-------------|
| 1.3.1 | `Login.tsx` | After login, read session context, route to role default page |
| 1.3.2 | `Login.tsx` | Add `must_change_password` check → force password change flow |

### 1.4 Sidebar Items for Each Role

| Role | Visible Groups | Default Page |
|------|---------------|-------------|
| principal | All | `/app/admin/dashboard` |
| people_admin | Messages + Org + Notifications | `/app/admin/org/persons` |
| operations | Dashboard + Messages + CRM + Email + Notifications | `/app/admin/crm/prospects` |
| sales | Dashboard + Messages + CRM + Notifications | `/app/admin/crm/prospects` |
| marketing | Dashboard + Messages + CRM + Email + Notifications | `/app/admin/crm/marketing-activities` |
| finance | Dashboard + Messages + CRM + Notifications | `/app/admin/crm/sales-contracts` |
| it_admin | Dashboard + Messages + Email + Org + Audit | `/app/admin/email/accounts` |

---

## Phase 2: Org Management Pages (Week 2)

### 2.1 Person Management

| Task | Backend | Frontend |
|------|---------|----------|
| 2.1.1 | `GET /api/admin/org/persons` (existing, verify) | `OrgPersonList.tsx` — table + search + create modal |
| 2.1.2 | `GET /api/admin/org/persons/:id` | `OrgPersonDetail.tsx` — detail drawer |
| 2.1.3 | `PUT /api/admin/org/persons/:id` | Edit form in detail drawer |
| 2.1.4 | `PUT /api/admin/org/persons/:id/deactivate` | Deactivate button with confirm |

### 2.2 User Account Management

| Task | Backend | Frontend |
|------|---------|----------|
| 2.2.1 | `GET /api/admin/org/users` (new controller) | `OrgUserList.tsx` |
| 2.2.2 | `POST /api/admin/org/users/invite` | Invite modal with role selector |
| 2.2.3 | `POST /api/admin/org/users/:id/reset-password` | Reset button |
| 2.2.4 | `PUT /api/admin/org/users/:id/deactivate` | Deactivate button |

### 2.3 Role Management

| Task | Backend | Frontend |
|------|---------|----------|
| 2.3.1 | `GET /api/admin/org/roles` | `OrgRoleList.tsx` |
| 2.3.2 | `POST /api/admin/org/roles/assign` | Assign role dropdown (respecting people_admin limits) |
| 2.3.3 | `DELETE /api/admin/org/roles/:id` | Revoke button |

---

## Phase 3: CRM Feature Parity (Week 3)

### 3.1 Prospect Full Feature (Match Old admin-prospect.html)

| Task | Description |
|------|-------------|
| 3.1.1 | ✅ Completed (stats, filters, table, bulk, edit, create, status change) |
| 3.1.2 | Hook up "send email" functionality to backend email API |
| 3.1.3 | Hook up batch export |

### 3.2 Missing CRM Pages

| Page | Status |
|------|:---:|
| Student Detail | ✅ Exists, verify with real data |
| Activity Detail | ✅ Exists |
| Contract Detail | ✅ Exists |
| Communication History | Shared component, `includeProspectHistory` param |
| Tag Management Page | Standalone page under CRM group |

### 3.3 Messages / Inbox

| Task | Backend | Frontend |
|------|---------|----------|
| 3.3.1 | `GET /api/admin/messages/inbox` | `MessageInbox.tsx` |
| 3.3.2 | `GET /api/admin/messages/:id` | `MessageDetail.tsx` |
| 3.3.3 | `POST /api/admin/messages` | Compose modal |

---

## Phase 4: Email & Notifications (Week 4)

### 4.1 Email Pages Polish

| Page | Status |
|------|:---:|
| Email Account List | ✅ Exists, verify CRUD |
| Email Log List | ✅ Exists, add filters |
| Email Template List | New page |

### 4.2 Notifications Inbox

| Page | Status |
|------|:---:|
| Notification Inbox | Placeholder → full implementation |
| Notification Detail | New page |
| Broadcast Send | ✅ Exists |

---

## Phase 5: Academic Workspace (Sprint 8, deferred)

Dependent on backend academic modules being built first (scheduling_run, scheduling_proposal, schedule_baseline, planned_session, class_session). Full IA defined in UI IA Spec §2.1.

---

## Phase 6: Teacher / Student / Guardian (Sprints 9-11, deferred)

Dependent on Academic workspace completion + messaging backend.

---

## Backend Tasks (Parallel to Frontend)

| # | Task | Priority |
|---|------|:---:|
| B-1 | Org CRUD endpoints (persons, users, roles) | P0 |
| B-2 | Messages API (inbox, send, read) | P0 |
| B-3 | Audit log API | P1 |
| B-4 | Email template CRUD (existing EmailTemplateController — verify) | P1 |
| B-5 | people_admin role enforcement (OrgMemberService rules — already coded) | P0 |
| B-6 | Notification inbox backend | P1 |
| B-7 | SaaSOperator: fix principal credential flow (auto-generate password visible in UI, not only email) | P0 |

---

## Milestones

| M# | Week | Deliverable |
|----|------|-------------|
| M1 | 1 | Admin workspace: Dashboard + grouped sidebar + role visibility + login routing |
| M2 | 2 | Org management: persons, users, roles CRUD (backend + frontend) |
| M3 | 3 | CRM: messages inbox + tag page + email frontend |
| M4 | 4 | Email+Notifications: template management + notification inbox |
| M5 | 5-8 | Academic workspace (Sprint 8) |
| M6 | 9-11 | Teacher / Student / Guardian (Sprints 9-11) |
