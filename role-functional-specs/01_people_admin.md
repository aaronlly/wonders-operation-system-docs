# Role Function Spec — people_admin v1.0

> **Role**: people_admin (学校级人员与账号管理员)
> **Workspace**: admin
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'people_admin'`
> **Resource Scope**:
>   scope_level: INSTITUTION
>   scope_source: institution_role_assignment
>   scope_filter: institution_id = current_user.institution_id
> **Sprint**: v1.5.1

---

## 1. Role Positioning

### What people_admin IS

| Attribute | Value |
|-----------|-------|
| 中文名称 | 人员与账号管理员 |
| Workspace | admin |
| Role source | `org.institution_role_assignment` |
| Scope | institution (single school) |
| Identity | staff member_type |
| Created by | principal or SaaS operator |

people_admin is a **school-level HR / people & account administrator**, responsible for day-to-day personnel management previously handled by the principal.

### What people_admin IS NOT

| NOT | Reason |
|-----|--------|
| Not a platform role | Cannot access `auth.platform_role_assignment` |
| Not a SaaS operator | Cannot create institutions |
| Not a system maintainer | Cannot access technical audit |
| Not a principal | Cannot manage all school aspects |
| Not a CRM role | Cannot access prospects, sales, payments |
| Not a teacher/student/guardian | No academic workspace access |

---

## 2. Workspace Access

| Workspace | Access | Notes |
|-----------|:---:|-------|
| admin | ✅ | Primary workspace |
| academic | ❌ | |
| teacher | ❌ | |
| student | ❌ | |
| guardian | ❌ | |

### Session Context

```json
{
  "current_workspace": "admin",
  "available_workspaces": ["admin"],
  "account_scope": "institution",
  "role_codes": ["people_admin"],
  "platform_role_codes": [],
  "member_type": ["staff"],
  "permissions_by_workspace": {
    "admin": {
      "org.person": ["read", "add", "modify", "deactivate"],
      "org.institution_member": ["read", "add", "modify", "deactivate"],
      "auth.user_account": ["read", "invite", "reset_password", "deactivate"],
      "org.institution_role_assignment": ["read", "assign", "revoke"],
      "notification": ["read"]
    }
  },
  "data_scope": {
    "institution_id": "uuid",
    "scope": "institution"
  }
}
```

---

## 3. Default Landing Page

```
/app/admin/org/persons
```

Per IA Spec v1.5.1 §4.1: people_admin enters Org / persons as primary work area.

---

## 4. Sidebar Menu

### Dashboard
| Menu | Route |
|------|-------|
| 仪表盘 | `/app/admin/dashboard` |

### 我的消息
| Menu | Route |
|------|-------|
| 收件箱 | `/app/admin/messages/inbox` |

### 组织与权限
| Menu | Route |
|------|-------|
| 人员档案 | `/app/admin/org/persons` |
| 成员管理 | `/app/admin/org/members` |
| 用户账号 | `/app/admin/org/users` |
| 角色分配 | `/app/admin/org/roles` |

### 通知
| Menu | Route |
|------|-------|
| 通知收件箱 | `/app/admin/notifications/inbox` |

### Hidden
CRM全部、Email infrastructure（accounts/templates/logs/delivery）、Broadcast send、Audit

### Dashboard
本周新增人员、待激活账号数、密码过期提醒、角色分配变更日志（per IA Spec §2.6）

---

## 5. Core User Stories

| # | Story |
|---|-------|
| US-1 | View all persons in my institution |
| US-2 | Create a new person record |
| US-3 | Modify person details |
| US-4 | Deactivate a departed person |
| US-5 | View all institution members |
| US-6 | Add a new member (staff/teacher) |
| US-7 | Modify member status |
| US-8 | Deactivate a member |
| US-9 | View all user accounts |
| US-10 | Invite a new user |
| US-11 | Reset a user's password |
| US-12 | Deactivate a user account |
| US-13 | View all role assignments |
| US-14 | Assign non-sensitive role (sales, marketing) |
| US-15 | Revoke non-sensitive role |
| US-16 | View my notifications |

---

## 6. Page-Level Functions

### 6.1 `/app/admin/org/members`
| Function | Token |
|----------|-------|
| List / Search | `org.person:read` |
| Create | `org.person:add` |
| Edit | `org.person:modify` |
| Deactivate | `org.person:deactivate` |

### 6.2 `/app/admin/org/members`
| Function | Token |
|----------|-------|
| List / Filter by type | `org.institution_member:read` |
| Add | `org.institution_member:add` |
| Modify status | `org.institution_member:modify` |
| Deactivate | `org.institution_member:deactivate` |

### 6.3 `/app/admin/org/users`
| Function | Token |
|----------|-------|
| List / Filter | `auth.user_account:read` |
| Invite | `auth.user_account:invite` |
| Reset password | `auth.user_account:reset_password` |
| Deactivate | `auth.user_account:deactivate` |

### 6.4 `/app/admin/org/roles`
| Function | Token |
|----------|-------|
| List assignments | `org.institution_role_assignment:read` |
| Assign (sales/marketing only) | `org.institution_role_assignment:assign` |
| Revoke | `org.institution_role_assignment:revoke` |

---

## 7. API Requirements

| Method | Endpoint | Token |
|--------|----------|-------|
| GET | `/api/session/context` | — |
| GET/POST/PUT | `/api/admin/org/persons[/:id]` | `org.person:*` |
| GET/POST/PUT | `/api/admin/org/members[/:id]` | `org.institution_member:*` |
| GET/POST/PUT | `/api/admin/org/users[/:id]` | `auth.user_account:*` |
| GET/POST/DELETE | `/api/admin/org/roles[/:id]` | `org.institution_role_assignment:*` |
| GET/PUT | `/api/admin/notifications/inbox[/:id]` | `notification:*` |

---

## 8. Backend Service Rules

### Data Scope
```
RULE-SCOPE-1: All queries filter by institution_id
RULE-SCOPE-2: Cannot cross institutions
RULE-SCOPE-3: Cannot access platform accounts
RULE-SCOPE-4: Cannot modify own role assignments
```

### Role Assignment
```
RULE-ROLE-1: CAN assign: sales, marketing
RULE-ROLE-2: CANNOT assign: principal, people_admin, academic_director, operations, finance
RULE-ROLE-3: CANNOT assign platform roles (saas_operator, system_maintainer)
RULE-ROLE-4: Only principal may assign it_admin
```

### User Account
```
RULE-USER-1: account_scope = 'institution'
RULE-USER-2: must_change_password = true on invite
RULE-USER-3: Deactivate = account_status = 'disabled', not physical delete
RULE-USER-4: Password creation: user may provide initial password or leave empty for system auto-generation (12-char random)
RULE-USER-5: People_admin may reset passwords for all institution users per ADR-025
```

---

## 9. Database Objects

### Read
`org.person`, `org.institution_member`, `org.institution_role_assignment`, `auth.user_account`, `auth.user_password`

### Write
`org.person` (INSERT/UPDATE), `org.institution_member` (INSERT/UPDATE), `org.institution_role_assignment` (INSERT/DELETE), `auth.user_account` (INSERT/UPDATE), `auth.user_password` (INSERT/UPDATE)

### NOT Accessed
`crm.*` (all), `email.*` (all), `messaging.*`, `contract.*`, `academic.*`, `auth.platform_role_assignment`, `core.*`

---

## 10. Permission Token Mapping

### Granted
```
org.person: read, add, modify, deactivate
org.institution_member: read, add, modify, deactivate
auth.user_account: read, invite, reset_password, deactivate
org.institution_role_assignment: read
org.institution_role_assignment: assign_limited   ← role-whitelisted (sales, marketing only)
org.institution_role_assignment: revoke_limited   ← role-whitelisted (sales, marketing only)
notification: read, modify
```

### Denied
All `crm.*`, `email.*`, `notification.broadcast:send`, `audit.*`, `auth.platform_role_assignment.*`

---

## 11. Notification Interactions

| Receives | Trigger |
|----------|---------|
| Password reset notification | User reset request |
| Account deactivation notice | Principal deactivates user |
| Role assignment notice | Role assigned to self |

**Cannot send**: School-wide broadcasts, email campaigns, internal messages to teachers/students

---

## 12. Acceptance Tests

| # | Test | Expected |
|---|------|----------|
| AT-1 | Login as people_admin | Session returns `role_codes=["people_admin"]`, org permissions, NO crm.* permissions |
| AT-2 | Create person | `org.person` INSERT successful, institution-scoped |
| AT-3 | Create member with role=sales | Full transaction succeeds (Person+Member+Role+Account+Password) |
| AT-4 | Create member with role=principal | 403 "Cannot assign high-sensitivity role: principal" |
| AT-5 | Create member with role=people_admin | 403 "Cannot assign high-sensitivity role: people_admin" |
| AT-6 | Cross-institution query | Empty results or 403 |
| AT-7 | Access CRM prospect list | 403 Forbidden |
| AT-8 | Deactivate user | account_status → "disabled", login blocked |
| AT-9 | Reset password | Hash updated, must_change_password = true, tokens invalidated |

---

## 13. Out of Scope

| Area | Reason |
|------|--------|
| CRM (prospects, sales, contracts, payments) | sales / finance / marketing scope |
| Email infrastructure (accounts, templates, logs) | it_admin scope |
| Broadcast send | principal / operations scope |
| Academic workspace | academic_director / teacher scope |
| Platform role management | SaaS operator scope |
| Institution creation/deletion | SaaS operator scope |
| Cross-institution access | platform-level |
| Audit logs | principal / it_admin scope |
