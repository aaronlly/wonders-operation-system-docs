# Role Function Spec — marketing v1.0

> **Role**: marketing (市场)
> **Workspace**: admin (limited — CRM marketing only)
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'marketing'`
> **Resource Scope**:
>   scope_level: INSTITUTION
>   scope_source: institution_role_assignment
>   scope_filter: institution_id = current_user.institution_id (campaign-scoped for email)
> **Created by**: principal or people_admin

---

## 1. Role Positioning

marketing manages **leads, marketing activities, campaign communications, marketing tags, and email campaign logs**.

### Lead Ownership (per IA Spec §2.6)

```text
marketing → creates leads (prospect:add)
sales     → follows up leads (prospect:modify, add_communication, change_status, convert)
```
Marketing owns lead creation. Sales owns lead follow-up. Marketing does NOT convert leads to students.

### NOT: sales conversion, payments, user management, email accounts

---

## 2-5. Summary

| Section | Detail |
|---------|--------|
| Landing | `/app/admin/crm/prospects` |
| Dashboard | 本周新增线索来源分布、进行中活动列表、活动报名统计、campaign 邮件打开率（per IA Spec §2.6） |
| Sidebar | Messages / CRM (prospects, marketing-activities, communications, tags) / Email (templates, logs) / Notifications |
| Core Stories | Create marketing leads, run activities, register participants, manage tags |
| Hidden | Sales contracts/payments, Org management, Email accounts, Broadcast, Audit |

---

## 6-7. API & Permissions

### Granted
```
crm.prospect: read, add (read: campaign segmentation; modify: marketing-owned fields only — source, campaign tags, activity participation. Sales follow-up fields read-only.)
crm.communication: read, add
crm.tag: read, add, modify, delete
crm.marketing_activity: read, add, modify, register, checkin
crm.task: read, add, modify, change_status
email.log: read (campaign)
email.template: read, modify (marketing templates)
```

### NOT granted
`crm.prospect:convert_to_student` (sales only), `crm.sales_contract:*`, `crm.sales_payment:*`, `email.account:*`, `org.person:*`

---

## 8. Backend Rules

```
RULE-MKT-1: Can create marketing leads (source = campaign/event)
RULE-MKT-2: Can manage own activities (created_by check)
RULE-MKT-3: Cannot convert prospects (sales responsibility)
RULE-MKT-4: Tags limited to prospect/activity entity types
```

## 9. Database

**Read/Write**: `crm.prospect`, `crm.marketing_activity`, `crm.activity_participant`, `crm.tag`, `crm.tag_assignment`, `crm.communication`, `crm.task`
**Read-only**: `email.email_log`, `email.email_template`
**NOT**: `crm.sales_contract`, `crm.sales_payment`
