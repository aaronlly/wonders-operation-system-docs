# Role Function Spec — student v1.0

> **Role**: student (学生)
> **Workspace**: student
> **Identity Source**: `org.institution_member.member_type = 'student'`
> **Role Source**: member_type (NOT institution_role_assignment)
> **Workspace Binding**: `auth.user_workspace_binding`
> **Resource Scope**:
>   scope_level: SELF
>   scope_source: current_user.person_id
>   scope_filter: person_id = current_user.person_id
> **Sprint**: Sprint 10

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 学生 |
| Workspace | student |
| Identity source | `org.institution_member.member_type = student` |
| Data scope | `self` (person_id = current_user.person_id) |
| Created by | SaaSOperator → principal → academic_director (member) + people_admin (account) |

student is a **self-service learner**, viewing own courses, homework, attendance, makeup sessions, leave requests, and messages.

### NOT: admin, other students, CRM, teacher workspace

---

## 2. Workspace Access

| Workspace | Access |
|-----------|:---:|
| student | ✅ |
| all others | ❌ |

---

## 3. Default Landing Page

```
/app/student/home
```

---

## 4. Sidebar Menu

| Menu | Route |
|------|-------|
| 首页 | `/app/student/home` |
| 消息 | `/app/student/messages/inbox` |
| 消息详情 | `/app/student/messages/:messageId` |
| 我的课程 | `/app/student/courses` |
| 课程详情 | `/app/student/courses/:courseId` |
| 课程考勤 | `/app/student/courses/:courseId/attendance` |
| 课程作业 | `/app/student/courses/:courseId/homework` |
| 请假申请 | `/app/student/courses/:courseId/leave-request` |
| 终止申请 | `/app/student/courses/:courseId/termination-request` |
| 我的补课 | `/app/student/makeups` |
| 补课详情 | `/app/student/makeups/:makeupId` |
| 出勤汇总 | `/app/student/attendance/summary` |
| 出勤明细 | `/app/student/attendance/history` |
| 学习状态 | `/app/student/status/termination` |
| 复课记录 | `/app/student/status/resume` |

---

## 5. Core User Stories

| # | Story |
|---|-------|
| US-1 | View my courses and schedule |
| US-2 | View my attendance |
| US-3 | View and submit homework |
| US-4 | Request leave |
| US-5 | Request course termination |
| US-6 | View makeup sessions |
| US-7 | Read messages from teachers |
| US-8 | View my learning status |

### Message Sending Scope

Student can send messages to: their own teachers (`class_member` role = teacher), school admin (via portal). Cannot message other students or parents.

### Hidden / NOT Accessible

| Object / Area | Reason |
|--------------|--------|
| Other students' data | self scope only |
| Teacher workspace | out of scope |
| Admin workspace | out of scope |
| Academic workspace | out of scope |
| CRM | out of scope |
| Contracts backend | out of scope |
| Account management | out of scope |

### Notifications

Student notifications are **Top Bar only** — bell icon in header, not a sidebar menu item. Per IA Spec §2.3.

---

## 6-7. API & Permissions

### Granted (all scoped to self)
```
course: read (own)
attendance: read (own)
homework: read, submit (own)
message: read (own)
leave_request: add (own)
termination_request: add (own)
makeup: read (own)
```

### NOT granted
```
All crm.*, org.*, auth.*, email.*, contract.*
Other students' data
Teacher workspace
Admin workspace
```

---

## 8. Backend Rules

```
RULE-STUDENT-1: person_id = current_user.person_id (self-only)
RULE-STUDENT-2: Cannot modify own identity profile
RULE-STUDENT-3: Leave/termination requests create records, not direct status changes
RULE-STUDENT-4: Homework submission is append-only (submit, not modify after grading)
```

## 9. Database

**Read**: `academic.class`, `academic.class_member`, `academic.course`, `academic.homework`, `execution.attendance`, `execution.class_session`, `execution.makeup`, `messaging.*`
**Write**: `academic.homework_submission`, `messaging.*`, leave/termination request tables
**NOT**: All admin/CRM/email/contract tables
