# Role Function Spec — it_admin v1.0

> **Role**: it_admin (学校技术管理员)
> **Workspace**: admin (technical), academic (technical optional)
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'it_admin'`
> **Resource Scope**:
>   scope_level: INSTITUTION
>   scope_source: institution_role_assignment
>   scope_filter: institution_id = current_user.institution_id
> **Created by**: principal

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 技术管理员 |
| Workspace | admin, academic (technical) |
| Scope | institution |
| Created by | principal |

it_admin manages **email infrastructure, account technical support, system configuration, and technical audit**.

### NOT: SaaS operator, platform role management, CRM, finance

---

## 2-5. Summary

| Section | Detail |
|---------|--------|
| Landing | `/app/admin/email/accounts` |
| Dashboard | 邮件投递失败数、重试队列、系统健康状态、最近登录失败日志（per IA Spec §2.6） |
| Sidebar | Messages / Org (users, technical) / Email (accounts, templates, logs, delivery-status) / Notifications (inbox, broadcast) / Audit |
| Core Stories | Manage email accounts/templates, view delivery status, retry failed emails, reset passwords, view audit |
| Hidden | CRM all, Sales payments, Platform role management |

---

## 6-7. API & Permissions

### Granted
```
email.account: read, add, modify, delete
email.template: read, add, modify, delete
email.delivery: read
email.send: retry (technical send)
auth.user_account: read, reset_password, deactivate
audit: read, export (school technical logs)
notification: read
notification.broadcast: send
```

### NOT granted
`auth.platform_role_assignment:*`, `org.education_institution:add`, cross-institution data

---

## 8. Backend Rules

```
RULE-IT-1: Cannot manage platform roles
RULE-IT-2: Cannot create institutions
RULE-IT-3: Cannot access other schools' data
RULE-IT-4: Email account changes logged to audit
RULE-IT-5: it_admin may reset password / deactivate normal user accounts only.
              it_admin may NOT deactivate principal, people_admin, finance, academic_director,
              operations, or platform-linked users.
              High-privilege account disablement requires principal approval.
              Per ADR-025: same rule applies to password reset on other users' accounts.
```

## 9. Database

**Read/Write**: `email.email_account`, `email.email_template`, `email.email_log`, `email.email_delivery`, `auth.user_account`
**Read-only**: `org.person`, `org.institution_member`

## 11. Notification Interactions

| Receives | Sends |
|----------|-------|
| System health alerts | School-wide broadcast |
| Email delivery failures | Technical notifications |
| Audit alerts | |

---
