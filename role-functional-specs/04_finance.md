# Role Function Spec — finance v1.0

> **Role**: finance (财务角色)
> **Workspace**: admin (finance limited)
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'finance'`
> **Resource Scope**:
>   scope_level: INSTITUTION
>   scope_source: institution_role_assignment
>   scope_filter: institution_id = current_user.institution_id

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 财务管理员 |
| Workspace | admin (finance), academic (contracts read optional) |
| Scope | institution |
| Created by | principal |

finance handles **sales contracts, payments, void/refund, export, and billing-related email logs**.

### NOT: CRM (prospects), user management, email accounts, teacher/student/guardian

---

## 2-5. Summary

| Section | Detail |
|---------|--------|
| Landing Page | `/app/admin/crm/sales-contracts` |
| Dashboard | 本月合同总额、待收款项、近期付款记录、退款申请（per IA Spec §2.6） |
| Sidebar | Dashboard / Messages / CRM（contracts, payments）/ Email（logs, billing）/ Audit（finance limited）/ Notifications inbox |
| Core Stories | View contracts, process payments, export, void/refund |
| Hidden | Prospects, Marketing activities, Org users, Role assignment, Email accounts |

---

## 6-7. API & Permissions

```
crm.sales_contract: read, export
crm.sales_payment: read, add, modify, void, export
email.log: read (billing only)
audit: read (finance only)
```

### NOT granted
`crm.prospect:*`, `auth.user_account:*`, `email.account:*`, `org.institution_role_assignment:*`

---

## 8. Backend Rules

```
RULE-FIN-1: Cannot modify contracts after settlement
RULE-FIN-2: Void requires audit trail
RULE-FIN-3: Export includes all institution contracts (not filtered by assigned_to)
RULE-FIN-4: Cannot access non-financial CRM data
```

---

## 9. Database

**Read**: `crm.sales_contract`, `crm.sales_payment`, `crm.student_profile`, `email.email_log`
**Write**: `crm.sales_payment`
**NOT accessed**: `crm.prospect`, `org.person`, `auth.user_account`, `email.email_account`

## 11. Notification Interactions

| Receives | Trigger |
|----------|---------|
| Payment confirmed | System notification |
| Refund processed | System notification |
| Contract status change | System notification |

**Cannot send**: School-wide broadcasts, email campaigns.

---
