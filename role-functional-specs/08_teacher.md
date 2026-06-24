# Role Function Spec — teacher v1.0

> **Role**: teacher (教师)
> **Workspace**: teacher
> **Identity Source**: `org.institution_member.member_type = 'teacher'`
> **Role Source**: member_type (NOT institution_role_assignment)
> **Workspace Binding**: `auth.user_workspace_binding`
> **Resource Scope**:
>   scope_level: ASSIGNED
>   scope_source: academic.class_member via core.resource
>   scope_filter: resource.person_id = self AND member_role IN ('teacher','assistant') AND left_at IS NULL
> **Sprint**: Sprint 9

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 教师 |
| Workspace | teacher |
| Identity source | `org.institution_member.member_type = teacher` |
| Data scope | `own_classes` (classes where teacher is assigned via class_member) |
| Created by | academic_director (member) + people_admin (account) |

teacher manages **own courses, own class students, homework, attendance, course records, and class-related messages**.

### NOT: admin workspace, CRM, all-school data, other teachers' classes

---

### 数据边界

```text
own_classes:
  SELECT class_id
  FROM academic.class_member cm
  JOIN core.resource r ON cm.member_resource_id = r.id
  WHERE r.person_id = current_user.person_id
    AND r.resource_type = 'teacher'
    AND cm.member_role IN ('teacher', 'assistant')
    AND cm.left_at IS NULL

own_courses:   course linked to own_classes via academic.class
own_students:  class_member WHERE member_role = 'student' AND class_id IN own_classes
               AND left_at IS NULL
```

teacher 只能看到：
- 自己的授课班级（通过 `resource.person_id = self` 关联，不是 teacher_person_id）
- 自己班级的课程
- 自己班级的学生

teacher 不能看到：
- 其他教师的班级
- 全校学生资源列表
- 非自己班级的学生详细信息

| Workspace | Access |
|-----------|:---:|
| teacher | ✅ full |
| admin | ❌ |
| academic | ❌ |
| student | ❌ |
| guardian | ❌ |

---

## 3. Default Landing Page

```
/app/teacher/home
```

---

## 4. Sidebar Menu

| Menu | Route |
|------|-------|
| 首页 | `/app/teacher/home` |
| 消息收件箱 | `/app/teacher/messages/inbox` |
| 未读消息 | `/app/teacher/messages/unread` |
| 消息详情 | `/app/teacher/messages/:messageId` |
| 我的课程 | `/app/teacher/courses` |
| 课程详情 | `/app/teacher/courses/:courseId` |
| 课程记录 | `/app/teacher/courses/:courseId/records` |
| 作业管理 | `/app/teacher/courses/:courseId/homework` |
| 课程建议 | `/app/teacher/courses/:courseId/suggest` |
| 考勤汇总 | `/app/teacher/attendance/summary` |
| 出勤明细 | `/app/teacher/attendance/history` |
| 代课（替代） | `/app/teacher/substitution/as-substitute` |
| 代课（被替代） | `/app/teacher/substitution/replaced` |
| 终止记录 | `/app/teacher/terminations` |

---

## 5. Core User Stories

| # | Story |
|---|-------|
| US-1 | View my courses |
| US-2 | View students in my classes |
| US-3 | Write course records |
| US-4 | Assign and review homework |
| US-5 | Record attendance for my classes |
| US-6 | View my attendance history |
| US-7 | Handle substitution (as substitute / replaced) |
| US-8 | View termination/leave/makeup requests |
| US-9 | Send messages to my classes / students |

### Message Sending Scope (per IA Spec §2.6)

| Target | Condition |
|--------|----------|
| Own students | class_member WHERE member_role='student' AND class_id IN own_classes (via resource.person_id join, NOT teacher_person_id) |
| Students' guardians | Linked via `guardian_assignment` |
| Co-teachers in same class | Same `class` other `class_member` (role = teacher) |
| Academic director | Institution-wide |

### Teaching Delivery Notes (v2.4)

**Hybrid Delivery**: Teacher may view and manage multiple delivery channels within one class_session (onsite + online). The `session_delivery_channel` table supports per-channel participant tracking and attendance.

**Class Member Lifecycle**: Teacher visibility uses `joined_at ≤ now() AND (left_at IS NULL OR left_at > now())` — supports future-dated assignments without prematurely granting access.

Cannot send to: non-own students, other parents, principal, sales, school-wide broadcast.

### Messaging Server-Side Rule (applies to ALL roles)

```text
All message target resolution MUST be server-side.
Client may request a logical target only (own_class, linked_student_teacher, school_admin).
Client MUST NOT submit arbitrary person_id recipient lists.
Backend resolves logical targets from session context + data scope.
```

### Hidden / NOT Accessible

| Object / Area | Reason |
|--------------|--------|
| Admin workspace | out of scope |
| CRM all | out of scope |
| Sales contracts | out of scope |
| Payment data | out of scope |
| Account management | out of scope |
| Role assignment | out of scope |
| Platform settings | out of scope |
| Other teachers' classes | data boundary |
| All-school student list | data boundary |

### Notifications

Teacher notifications are **Top Bar only** — bell icon in header, not a sidebar menu item. Per IA Spec §2.2.

---

## 6. API Requirements

All `/api/teacher/*` endpoints with `teacher.*` tokens.

Key data scope rule: `WHERE resource.person_id = current_user.person_id AND class_member.member_role IN ('teacher','assistant') AND left_at IS NULL`

---

## 7. Permission Tokens

### Granted
```
class: read (own_classes)
course: read (own_courses)
student: read (own_students, limited fields)
attendance: add, modify (own_classes)
homework: add, modify (own_classes)
course_record: add, modify (own_classes)
message: read, send (own_classes, own_students)
```

### NOT granted
```
org.person: add/modify
auth.user_account: *
crm.* (all)
email.account: *
org.institution_role_assignment: *
```

---

## 8. Backend Rules

```
RULE-TEACHER-1: All queries filtered by resource.person_id = current_user.person_id (via class_member + resource join)
RULE-TEACHER-2: class_member.member_role IN ('teacher','assistant') + left_at IS NULL defines active teaching assignments
RULE-TEACHER-3: Cannot modify contract or payment data
RULE-TEACHER-4: Cannot access other teachers' classes
RULE-TEACHER-5: Messages limited to own students/classes
```

## 9. Database

**Read/Write**: `academic.class`, `academic.class_member`, `academic.course`, `academic.homework`, `academic.homework_submission`, `execution.attendance`, `execution.class_session`, `messaging.*`
**Read-only**: `org.person` (own students), `academic.study_plan`
**NOT**: `crm.*`, `contract.*`, `email.*`
