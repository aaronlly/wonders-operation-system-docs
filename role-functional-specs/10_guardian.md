# Role Function Spec — guardian v1.0

> **Role**: guardian (家长/监护人)
> **Workspace**: guardian
> **Identity Source**: `org.guardian_assignment.guardian_person_id = current_user.person_id`
> **Role Source**: guardian_assignment (NOT institution_role_assignment)
> **Workspace Binding**: `auth.user_workspace_binding`
> **Resource Scope**:
>   scope_level: ASSIGNED
>   scope_source: org.guardian_assignment
>   scope_filter: guardian_person_id = self AND valid_from ≤ now AND (valid_to IS NULL OR valid_to ≥ now)
> **Sprint**: Sprint 11

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 家长/监护人 |
| Workspace | guardian |
| Identity source | `org.guardian_assignment` (guardian_person_id) + workspace binding |
| Data scope | `linked_student_person_ids` (students assigned via guardian_assignment) |
| Created by | operations or academic_director (guardian assignment) + people_admin (account) |

guardian monitors **linked students' courses, attendance, homework, payments, and communicates with teachers/the school**.

### NOT: admin, CRM, other people's children, teacher workspace

---

## 2. Workspace Access

| Workspace | Access |
|-----------|:---:|
| guardian | ✅ |
| all others | ❌ |

---

## 3. Default Landing Page

```
/app/guardian/home
```

---

## 4. Sidebar Menu

| Menu | Route |
|------|-------|
| 首页 | `/app/guardian/home` |
| 消息 | `/app/guardian/messages/inbox` |
| 未读消息 | `/app/guardian/messages/unread` |
| 消息详情 | `/app/guardian/messages/:messageId` |
| 我的孩子 | `/app/guardian/students` |
| 孩子详情 | `/app/guardian/students/:studentId` |
| 课程 | `/app/guardian/students/:studentId/courses` |
| 出勤 | `/app/guardian/students/:studentId/attendance` |
| 作业 | `/app/guardian/students/:studentId/homework` |
| 请假申请 | `/app/guardian/students/:studentId/leave-request` |
| 付款 | `/app/guardian/students/:studentId/payments` |

---

## 5. Core User Stories

| # | Story |
|---|-------|
| US-1 | View list of linked students |
| US-2 | View child's courses |
| US-3 | View child's attendance |
| US-4 | View child's homework |
| US-5 | Submit leave request for child |
| US-6 | View payment summary |
| US-7 | Send/receive messages with teachers |
| US-8 | View makeup records |

### Message Sending Scope (per IA Spec §2.6)

| Target | Condition |
|--------|----------|
| Child's teachers | `class_member` for linked child's class, role = teacher |
| School admin | principal or operations via school contact portal |

Cannot send to: other parents, other students' teachers, school staff.

### Hidden / NOT Accessible

| Object / Area | Reason |
|--------------|--------|
| Non-linked students | guardian_assignment scope |
| Admin workspace | out of scope |
| CRM | out of scope |
| Teacher workspace | out of scope |
| Academic workspace | out of scope |
| School internal personnel management | out of scope |

### Notifications

Guardian notifications are **Top Bar only** — bell icon in header, not a sidebar menu item. Per IA Spec §2.4.

---

## 6-7. API & Permissions

### Granted (all scoped to linked_students)
```
linked_student.profile: read
linked_student.course: read
linked_student.attendance: read
linked_student.homework: read
linked_student.message: read, add
linked_student.leave_request: add
linked_student.payment_summary: read
```

### NOT granted
```
All crm.*, org.*, auth.*, email.*
Other families' data
Teacher workspace
Admin workspace
```

---

## 8. Backend Rules

```
RULE-GUARDIAN-1: linked_student_person_ids derived from org.guardian_assignment
              WHERE guardian_assignment.guardian_person_id = current_user.person_id
              AND valid_from <= CURRENT_DATE
              AND (valid_to IS NULL OR valid_to >= CURRENT_DATE)
              (guardian_assignment uses valid_from/valid_to range, NOT a status column)
RULE-GUARDIAN-2: guardian_person_id = current_user.person_id
RULE-GUARDIAN-3: One guardian can be linked to multiple students
RULE-GUARDIAN-4: Payment summary is read-only (no payment initiation)
RULE-GUARDIAN-5: Leave requests are submitted, not auto-approved
RULE-GUARDIAN-6: Messages scoped to linked_student context
```

## 9. Database

**Read**: `org.guardian_assignment`, `org.person` (linked students), `academic.class`, `academic.course`, `execution.attendance`, `academic.homework`, `contract.contract` (payment summary), `messaging.*`
**Write**: `messaging.*`, leave request tables
**NOT**: All admin/CRM/email tables
