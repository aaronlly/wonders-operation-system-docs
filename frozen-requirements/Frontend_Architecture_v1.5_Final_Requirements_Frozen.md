# Frontend Architecture Design v1.5 — Final Requirements Frozen

**Date**: 2026-06-24  
**Status**: v1.5 Final Requirements Frozen — implementation handoff baseline

---

## 1. Project Structure

整个系统包含三个前端项目，各有独立的部署和路由体系：

| Project | Directory | Route System | Scope |
|---------|-----------|-------------|-------|
| **CRM Frontend** | `D:\dev\CRM` | `/app/admin/*` (workspace route) | 管理员/运营/销售使用的 CRM SPA |
| **SaaSOperator** | `D:\dev\LLYProjects\ClassingSystem\SaaSOperator` | 独立登录，无 workspace 路由 | 平台运营管理工具（创建机构、指定校长），**不纳入 workspace 体系** |
| **Wonders Academy Website** | `D:\dev\LLYProjects\WondersAcademyWebsite` | 独立域名，传统多页 | 官网展示、招生公开表单 |

本文档覆盖的是 **CRM Frontend** 的架构设计。SaaSOperator 和 Website 各自独立维护。

---

## 2. Core Principles

### 2.1 Workspace Is Not Role

The system has five workspaces:

```text
academic
teacher
student
guardian
admin
```

A user may have multiple roles and may access multiple workspaces.

### 2.2 Effective Permission Rule

```text
effective_permission = current_workspace ∩ user_roles ∩ object/action
```

### 2.3 Guardian Workspace

Guardian is the fifth workspace, confirmed in Permission Matrix v1.0 §18.

```text
workspace = guardian
identity_source = org.guardian_assignment
```

Guardian identity is not an institution role — it is resolved through `org.guardian_assignment.guardian_person_id`.

### 2.4 Sales Role

Sales is not a workspace — sales operates under the admin workspace.

```text
role_code = sales
workspace = admin
```

---

## 3. Technology Stack

- React
- TypeScript
- Vite
- Browser Router
- Ant Design
- CSS Modules
- Zustand
- TanStack Query

Application entry:

```text
/app
```

---

## 4. Layout Architecture

Desktop layout:

```text
Top Bar
├─ Logo
├─ Workspace Switcher
├─ Global Search
├─ Messages
├─ Notifications
└─ User Menu

Side Menu
Main Content
```

Mobile adaptation:

- Side Menu becomes drawer.
- Complex calendar/grid views degrade to list/day views.

---

## 5. Top Bar Rules

Workspace switcher is hidden when the user has only one available workspace:

```ts
showWorkspaceSwitcher = availableWorkspaces.length >= 2
```

Reasoning:
- `guardian` is a relationship identity (from `org.guardian_assignment`), not a `member_type`
- A user may have multiple workspaces (e.g. teacher + guardian), so the switcher should just follow the count of available workspaces
- If a user only has one workspace (e.g. only `admin`), no switcher needed
- `teacher` and `student` workspaces are hidden from the switcher only if those are the **only** workspaces the user has — but if a teacher is also a guardian, the switcher appears

---

## 6. Global Search

Global search is object-location search, not full-text search.

Searchable objects:

```text
student
teacher
class
course
contract                    ← academic / fulfillment contract（contract.contract）
room
crm.prospect
crm.sales_contract          ← sales-side contract（crm.sales_contract）
crm.marketing_activity
crm.conversion              ← Conversion Pipeline deals（searched through prospect/student detail if not globally searchable）
```

Results must be permission-filtered by workspace and role.

---

## 7. Messages vs Email vs Notifications

```text
Messages
```

User-to-user conversations, reply-capable（personal inbox, `/app/*/messages/*`）.

```text
Email
```

Infrastructure for email accounts, templates, send logs, delivery tracking, SMTP configuration.

```text
Notifications
```

System-triggered event alerts（task_assigned / renewal_reminder / churn_alert）.

```text
Broadcast
```

Institution/role-level announcements, sent by principal / operations / it_admin.
v1.5: implemented as messaging.notification(type = 'broadcast') + multiple notification_recipient rows.
May be upgraded to dedicated broadcast_message model when rich-text scheduling/targeting is needed.

Admin routes:

```text
/app/admin/messages/*          Personal inbox（user-facing conversations）
/app/admin/email/*             Email infrastructure（accounts, templates, logs, delivery）
/app/admin/notifications/*     In-app notifications / broadcast（messaging schema）
```

These three domains must not be confused. See `CRM_Integration_Architecture` §V2_2 DDL comments for authoritative product semantics.

---

## 8. Route Prefixes

```text
/app/academic/*
/app/teacher/*
/app/student/*
/app/guardian/*
/app/admin/*
```

---

## 9. Admin Workspace Modules

```text
/app/admin/messages/*
/app/admin/org/*
/app/admin/crm/*
/app/admin/email/*
/app/admin/notifications/*
/app/admin/audit
```

### 8.1 CRM Routes

```text
/app/admin/crm/prospects
/app/admin/crm/students
/app/admin/crm/students/:id/guardians      # Student guardian links tab
/app/admin/crm/marketing-activities
/app/admin/crm/tasks
/app/admin/crm/conversions
/app/admin/crm/conversions/:id
/app/admin/crm/conversion-stages
/app/admin/crm/sales-contracts
/app/admin/crm/renewals
/app/admin/crm/churn-warnings
/app/admin/crm/communications
/app/admin/crm/tags
```

### 8.2 Email Routes（email schema）

```text
/app/admin/email/accounts
/app/admin/email/templates
/app/admin/email/logs
/app/admin/email/delivery-status
```

### 8.3 Notification Routes（messaging schema = in-app only）

```text
/app/admin/notifications/inbox
/app/admin/notifications/:id
/app/admin/notifications/broadcast
```

---

## 10. Teacher Messaging Permission Reservation

Even if teacher-facing messaging UI is implemented later, the permission model must reserve:

```text
teacher_send_message
teacher_view_own_messages
teacher_view_own_delivery_status
```

Teacher message route:

```text
/app/teacher/messages/*
```

---

## 11. Session Context

The session context MUST include the full field set defined in Role & Permission Matrix §21.2 / §21.3 / §21.4.

### Principal Session Context Example

```jsonc
{
  "current_workspace": "admin",
  "available_workspaces": ["admin", "academic"],
  "account_scope": "institution",
  "institution_id": "uuid",
  "person_id": "uuid",
  "user_account_id": "uuid",
  "role_codes": ["principal"],
  "platform_role_codes": [],
  "member_type": ["staff"],
  "permissions_by_workspace": {
    "admin": {
      "org.person": ["read", "add", "modify", "deactivate"],
      "org.institution_member": ["read", "add", "modify", "deactivate"],
      "org.institution_role_assignment": ["read", "assign", "revoke"],
      "auth.user_account": ["read", "invite", "reset_password", "deactivate"],
      "crm.prospect": ["read", "add", "modify", "assign_owner", "change_status", "add_communication", "convert_to_student", "archive", "export"],
      "crm.student_profile": ["read", "modify"],
      "crm.marketing_activity": ["read", "add", "modify", "publish", "cancel", "register", "checkin"],
      "crm.task": ["read", "add", "modify", "assign", "complete", "cancel"],
      "crm.sales_contract": ["read", "add", "modify", "activate", "terminate"],
      "crm.sales_payment": ["read", "add", "void", "export"],
      "crm.renewal_reminder": ["read", "add", "modify", "cancel", "resolve", "dismiss"],
      "crm.churn_warning": ["read", "add", "modify", "resolve"],
      "crm.communication": ["read", "add_communication", "send", "export"],
      "crm.tag": ["read", "add", "modify", "delete", "assign"],
      "crm.conversion": ["read", "add", "modify", "move_stage", "assign_owner", "archive"],
      "crm.conversion_stage": ["read", "add", "modify", "delete"],
      "email.send": ["send"],
      "email.log": ["read"],
      "notification": ["read"],
      "notification.broadcast": ["send"]
    }
  },
  "data_scope": {
    "institution_id": "uuid",
    "campus_ids": [],
    "assigned_only": false,
    "own_classes_only": false,
    "linked_student_person_ids": []
  }
}
```

### people_admin Session Context Example (v1.5.1)

```jsonc
{
  "current_workspace": "admin",
  "available_workspaces": ["admin"],
  "account_scope": "institution",
  "institution_id": "uuid",
  "person_id": "uuid",
  "user_account_id": "uuid",
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
    "campus_ids": [],
    "assigned_only": false,
    "own_classes_only": false,
    "linked_student_person_ids": []
  }
}
```

### Guardian Session Context Example

```jsonc
{
  "current_workspace": "guardian",
  "available_workspaces": ["guardian"],
  "account_scope": "institution",
  "institution_id": "uuid",
  "person_id": "guardian-person-uuid",
  "user_account_id": "uuid",
  "role_codes": [],
  "platform_role_codes": [],
  "member_type": [],
  "relationship_codes": ["mother"],
  "permissions_by_workspace": {
    "guardian": {
      "linked_student.profile": ["read"],
      "linked_student.course": ["read"],
      "linked_student.attendance": ["read"],
      "linked_student.homework": ["read"],
      "linked_student.message": ["read", "add"],
      "linked_student.leave_request": ["add"],
      "linked_student.payment_summary": ["read"]
    }
  },
  "data_scope": {
    "institution_id": "uuid",
    "linked_student_person_ids": [
      "student-person-uuid-1",
      "student-person-uuid-2"
    ]
  }
}
```

### Platform User Session Context Example

```jsonc
{
  "current_workspace": null,
  "available_workspaces": [],
  "account_scope": "platform",
  "institution_id": null,
  "person_id": "platform-person-uuid",
  "user_account_id": "uuid",
  "role_codes": [],
  "platform_role_codes": ["saas_operator"],
  "permissions_by_workspace": {},
  "platform_permissions": {
    "org.education_institution": ["read", "add", "modify", "deactivate"],
    "auth.user_account.principal": ["add"],
    "org.institution_role_assignment.principal": ["assign"],
    "email.initial_password": ["send"]
  },
  "data_scope": {
    "scope": "platform"
  }
}
```

Full field reference: See Role & Permission Matrix §21.2 / §21.3 / §21.4 for authoritative examples.

---

## 12. Engineering Structure

```text
src/
├─ app/
├─ layouts/
├─ routes/
├─ pages/
│  ├─ admin/
│  │  ├─ messages/              Personal inbox
│  │  ├─ crm/
│  │  ├─ email/                 Email infrastructure
│  │  └─ notifications/         In-app notifications（messaging schema）
│  └─ guardian/                 Future (Sprint 11)
│     ├─ home/
│     ├─ messages/
│     └─ students/
├─ features/
│  ├─ crm/
│  │  ├─ prospect/
│  │  ├─ student-profile/
│  │  │  └─ guardians/                  # Student guardian links tab
│  │  ├─ marketing-activity/
│  │  ├─ task/
│  │  ├─ conversion/
│  │  ├─ conversion-stage/
│  │  ├─ sales-contract/
│  │  ├─ renewal/
│  │  ├─ churn-warning/
│  │  ├─ communication/
│  │  └─ tag/
│  ├─ email/                    Email service, SMTP, template rendering
│  ├─ notifications/            In-app notification UI（messaging）
│  └─ guardian/                 Future (Sprint 11)
│     ├─ linked-student/
│     ├─ guardian-message/
│     ├─ guardian-attendance/
│     ├─ guardian-homework/
│     └─ guardian-payment/
├─ shared/
├─ services/
└─ store/
```

---

## 13. Legacy Handling

Legacy pages remain temporary:

```text
/admin-login.html
/admin-prospect.html
```

Target:

```text
/admin-prospect.html → /app/admin/crm/prospects
```
