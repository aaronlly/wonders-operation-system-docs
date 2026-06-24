# Role Function Spec — operations v1.0

> **Role**: operations (学校运营角色)
> **Workspace**: admin
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'operations'`
> **Resource Scope**:
>   scope_level: INSTITUTION
>   scope_source: institution_role_assignment
>   scope_filter: institution_id = current_user.institution_id (excludes HR privacy & technical data)
> **Sprint**: Sprint 5+

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 运营管理员 |
| Workspace | admin (primary), academic (read/support optional) |
| Scope | institution-wide — can see ALL CRM data within school |
| Identity | staff member_type |
| Created by | principal |

operations is the school's **daily operations role**, assisting the principal with full CRM management, marketing activities, task coordination, renewal/churn handling, and notification broadcasting.

### What operations IS NOT

- Not a CRM-only role (has org + email access too)
- Not a financial role (no payment processing)
- Not an academic role (no class/schedule management)
- Not a platform role

---

## 2. Workspace Access

| Workspace | Access | Notes |
|-----------|:---:|-------|
| admin | ✅ full | Primary |
| academic | ⚠️ read/support | Optional — view student resources, contracts |
| teacher | ❌ | |
| student | ❌ | |
| guardian | ❌ | |

---

## 3. Default Landing Page

```
/app/admin/crm/prospects
```

Per IA Spec v1.5.1 §4.1: operations enters CRM prospects or tasks.

**Dashboard**：全校待办任务（按优先级）、续费到期提醒、流失预警列表、今日活动（per IA Spec §2.6）。

**Delivery Status**：institution-wide — 查看本校全部邮件投递记录。v1.5.1 不按 campus 分割（投递属于基础设施层）。

**Data Scope**：institution-wide CRM scope。不包含 HR 人员隐私数据（org.person 敏感字段）、技术配置数据（it_admin 域）、平台数据。GDPR 下默认排除跨学校数据访问。

**Org boundary**: org.person read is minimal (CRM-linked persons only, not full HR profiles). Email accounts: no access. Email templates: read/use only, no infrastructure modification. These boundaries prevent overlap with people_admin and it_admin domains.

**Tag management**：拥有 CRM 对象 modify 权限 → 默认拥有该对象上 Tag 的管理权（per IA Spec §2.6）。

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

### CRM
| Menu | Route |
|------|-------|
| 线索管理 | `/app/admin/crm/prospects` |
| 客户管理 | `/app/admin/crm/students` |
| 活动管理 | `/app/admin/crm/marketing-activities` |
| 转化管理 | `/app/admin/crm/conversions` |
| 合同管理 | `/app/admin/crm/sales-contracts` |
| 任务跟踪 | `/app/admin/crm/tasks` |
| 续费管理 | `/app/admin/crm/renewals` |
| 流失预警 | `/app/admin/crm/churn-warnings` |
| 沟通记录 | `/app/admin/crm/communications` |
| 标签管理 | `/app/admin/crm/tags` |

### Email
| Menu | Route |
|------|-------|
| 邮箱账户 | `/app/admin/email/accounts` |
| 邮件模板 | `/app/admin/email/templates` |
| 投递日志 | `/app/admin/email/logs` |

### 通知
| Menu | Route |
|------|-------|
| 通知收件箱 | `/app/admin/notifications/inbox` |
| 发送广播 | `/app/admin/notifications/broadcast` |

---

## 5. Core User Stories

| # | Story |
|---|-------|
| US-1 | View ALL prospects in my institution (not just assigned) |
| US-2 | View ALL students |
| US-3 | Create and manage marketing activities |
| US-4 | Coordinate tasks across teams |
| US-5 | Handle renewal reminders |
| US-6 | Manage churn warnings |
| US-7 | View all communications |
| US-8 | Manage email accounts (SMTP) |
| US-9 | View and manage email templates |
| US-10 | View email delivery logs |
| US-11 | Send school-wide notification broadcasts |
| US-12 | View notifications |

---

## 6. Page-Level Functions

All CRM pages with **full read/write** on assigned objects. Key differentiators from sales:

| Page | operations can | sales cannot |
|------|---------------|-------------|
| Prospects | See ALL prospects | Only assigned |
| Students | Full read + modify CRM fields | Read own converted |
| Churn Warnings | Add + modify + resolve | Read assigned only |
| Renewals | Create + resolve + dismiss | Read own only |
| Email Accounts | CRUD | No access |
| Email Templates | CRUD | No access |
| Broadcast | Send | No access |

---

## 7. API Requirements

### CRM (full access)
All `/api/admin/crm/*` endpoints — read access across entire institution.

### Email
| Method | Endpoint |Token |
|--------|----------|------|
| CRUD | `/api/admin/email/accounts` | `email.account:*` |
| CRUD | `/api/admin/email/templates` | `email.template:*` |
| GET | `/api/admin/email/logs` | `email.log:read` |

### Notifications
| Method | Endpoint | Token |
|--------|----------|-------|
| POST | `/api/admin/messaging/broadcasts` | `notification.broadcast:send` |

---

## 8. Backend Service Rules

```
RULE-SCOPE-1: Institution-scoped (all queries filtered by institution_id)
RULE-SCOPE-2: operations can view ALL CRM data within institution (no assigned_to filter)
RULE-EMAIL-1: Can manage email accounts (smtp config)
RULE-EMAIL-2: Can manage email templates
RULE-BROADCAST-1: Can send school-wide broadcasts
```

---

## 9. Database Objects

### Read
All `crm.*` tables (no assigned_to filter), `org.person`, `org.institution_member`, `email.*`, `messaging.*`

### Write
`crm.prospect`, `crm.student_profile`, `crm.marketing_activity`, `crm.activity_participant`, `crm.task`, `crm.renewal_reminder`, `crm.churn_warning`, `crm.communication`, `email.email_account`, `email.email_template`, `messaging.broadcast_message`

---

## 10. Permission Token Mapping

```
crm.prospect: read
crm.student_profile: read, modify
crm.student: read, add_communication, add_guardian
crm.marketing_activity: read
crm.task: read, add, modify, change_status
crm.conversion: read
crm.sales_contract: read
crm.renewal: read, modify
crm.churn_warning: read, add, modify
crm.communication: read
crm.tag: read

email.account: read, add, modify, delete
email.template: read, add, modify, delete
email.log: read

notification: read
notification.broadcast: send
```

---

## 11. Notification Interactions

| Receives | Sends |
|----------|-------|
| System notifications | School-wide broadcasts |
| Task assignments | |
| Renewal alerts | |
| Churn alerts | |

---

## 12. Acceptance Tests

| # | Test |
|---|------|
| AT-1 | Login → session returns `role_codes=["operations"]` + all above permissions |
| AT-2 | Prospect list returns ALL institution prospects (not filtered by assigned_to) |
| AT-3 | Can create email account |
| AT-4 | Can send broadcast |
| AT-5 | Can create churn warning |

---

## 13. Out of Scope

Org management (persons, members, users, roles — owned by principal/people_admin), Audit (principal/it_admin), Sales payments (finance), Academic workspace (academic_director), Platform management (SaaS operator), HR privacy data, cross-institution data
