# Role Function Spec — sales v1.0

> **Role**: sales (销售)
> **Workspace**: admin (CRM)
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'sales'`
> **Resource Scope**:
>   scope_level: ASSIGNED
>   scope_source: crm.prospect.assigned_to
>   scope_filter: assigned_to = current_user.person_id
> **Created by**: principal or people_admin

---

## 1. Role Positioning

sales manages the **prospect → communication → conversion → contract pipeline**.

### Key difference from operations: sales sees only ASSIGNED data, operations sees ALL.

### Lead Ownership (per IA Spec §2.6)

```text
marketing → creates leads (prospect:add)
sales     → follows up leads (prospect:modify, add_communication, change_status, convert)
```
Marketing owns lead creation. Sales owns lead follow-up. Ownership is single: a lead belongs to one sales person (`assigned_to`).

---

## 2-5. Summary

| Section | Detail |
|---------|--------|
| Landing | `/app/admin/crm/prospects` |
| Dashboard | 我的待跟进线索、我的进行中转化、我的到期合同、本月业绩（per IA Spec §2.6） |
| Sidebar | Dashboard / Messages / CRM（prospects, students, conversions, contracts, tasks, communications, renewals, tags）/ Email logs（own）/ Notifications inbox |
| Tag management | 仅可 assign 已有 Tag，不可创建/删除/修改 Tag（per IA Spec §2.6） |
| Core Stories | View assigned prospects, add leads, communicate, convert, manage contracts |
| Hidden | Org management, Email infrastructure, Broadcast, Audit |

---

## 6-7. API & Permissions

### Granted
```
crm.prospect: read, add, modify, assign_owner, change_status, add_communication, convert_to_student, archive, export
crm.communication: read, add
crm.tag: read
crm.conversion: read, modify
crm.marketing_activity: read
crm.task: read, add, modify, change_status
crm.sales_contract: read, add, modify, archive
crm.student_profile: read
crm.student: read
crm.renewal: read
crm.churn_warning: read
```

### Data Scope
```
assigned_to = current_user.person_id
```
Exception: principal/operations may configure "sales can view all prospects" (configurable).

### NOT granted
`crm.marketing_activity:add` (marketing only), `crm.renewal:modify` (operations), `email.account:*`, `org.person:*`, `notification.broadcast:send`

---

## 8. Backend Rules

```
RULE-SALES-1: Default view shows assigned prospects only
RULE-SALES-2: Can convert assigned prospects to students
RULE-SALES-3: Contract operations limited to own contracts
RULE-SALES-4: Cannot view other sales' prospects without config flag
```

## 9. Database

**Read/Write**: `crm.prospect`, `crm.communication`, `crm.conversion`, `crm.sales_contract`, `crm.task`
**Read-only**: `crm.student_profile`, `crm.renewal_reminder`, `crm.churn_warning`, `org.person`
