# Role Function Spec — academic_director v1.0

> **Role**: academic_director (教务负责人)
> **Workspace**: academic (primary), admin (limited)
> **Identity Source**: `org.institution_member.member_type = 'staff'`
> **Role Source**: `org.institution_role_assignment.role_code = 'academic_director'`
> **Resource Scope**:
>   scope_level: INSTITUTION
>   scope_source: institution_role_assignment
>   scope_filter: institution_id = current_user.institution_id
> **Sprint**: Sprint 8

---

## 1. Role Positioning

| Attribute | Value |
|-----------|-------|
| 中文名称 | 教务负责人 |
| Workspace | academic (primary), admin (limited read) |
| Scope | institution-wide |
| Identity | staff member_type |
| Created by | principal |

academic_director manages **courses, class structures, teacher resources, scheduling runs, proposals, baselines, planned sessions, class sessions, and teaching quality** within the school.

### Data Scope

```
scope_level: INSTITUTION
scope_source: org.institution_role_assignment
scope_filter: institution_id = self.institution_id
```

### Key DB Objects (v2.4-aligned)

`scheduling_run`, `scheduling_proposal`, `scheduling_slot`, `schedule_baseline`, `planned_session`, `class_session`, `academic.class_member`, `core.resource`, `academic.course`, `academic.course_version`, `academic.class`, `academic.class_study_plan`, `execution.attendance`

---

## 2. Workspace Access

| Workspace | Access | Notes |
|-----------|:---:|-------|
| academic | ✅ full | Primary |
| admin | ⚠️ limited | Read CRM prospects, contracts |
| teacher | ❌ | |
| student | ❌ | |
| guardian | ❌ | |

---

## 3. Default Landing Page

```
/app/academic/dashboard
```

---

## 4. Sidebar Menu

### Academic Workspace

**Dashboard**
| Menu | Route |
|------|-------|
| 教务仪表盘 | `/app/academic/dashboard` |

**我的消息**
| Menu | Route |
|------|-------|
| 收件箱 | `/app/academic/messages/inbox` |
| 未读消息 | `/app/academic/messages/unread` |
| 消息详情 | `/app/academic/messages/:messageId` |

**资源管理**
| Menu | Route |
|------|-------|
| 学生资源 | `/app/academic/resources/students` |
| 教师资源 | `/app/academic/resources/teachers` |
| 教室资源 | `/app/academic/resources/rooms` |

**班级管理**
| Menu | Route |
|------|-------|
| 班级列表 | `/app/academic/classes` |
| 班级详情 | `/app/academic/classes/:classId` |
| 班级成员 | `/app/academic/classes/:classId/members` |
| 班级基线 | `/app/academic/classes/:classId/baseline` |

**排课管理**
| Menu | Route |
|------|-------|
| 排课任务 | `/app/academic/scheduling/runs` |
| 候选方案 | `/app/academic/scheduling/proposals` |
| 进度与确认 | `/app/academic/scheduling/confirm` |

**课表与执行**
| Menu | Route |
|------|-------|
| 课表版本 | `/app/academic/schedule-baselines` |
| 计划课次 | `/app/academic/planned-sessions` |
| 执行课次 | `/app/academic/class-sessions` |
| 今日执行 | `/app/academic/execution/today` |
| 执行历史 | `/app/academic/execution/history` |
| 补课管理 | `/app/academic/execution/makeups` |

**教学审核**
| Menu | Route |
|------|-------|
| 学习计划审核 | `/app/academic/review/study-plan` |
| 课程变更审核 | `/app/academic/review/course-changes` |
| 教学材料变更审核 | `/app/academic/review/material-changes` |

**合同与履约**
| Menu | Route |
|------|-------|
| 合同列表 | `/app/academic/contracts` |
| 合同详情 | `/app/academic/contracts/:contractId` |
| 结算信息 | `/app/academic/contracts/:contractId/settlement` |

**系统配置**
| Menu | Route |
|------|-------|
| 时间规则 | `/app/academic/settings/time-rules` |
| 排课策略 | `/app/academic/settings/scheduling-strategy` |
| 通知策略 | `/app/academic/settings/notification-strategy` |
| 审计日志 | `/app/academic/settings/audit-log` |

### Admin Workspace (Limited)

| Menu | Route |
|------|-------|
| CRM Prospects (read) | `/app/admin/crm/prospects` |
| CRM Conversions (read) | `/app/admin/crm/conversions` |
| CRM Student Profile (read) | `/app/admin/crm/students` |
| Contracts (read) | `/app/admin/crm/sales-contracts` |

| Menu | Route |
|------|-------|
| CRM Prospects (read) | `/app/admin/crm/prospects` |
| Contracts (read) | `/app/academic/contracts` |

---

## 5. Core User Stories

| # | Story |
|---|-------|
| US-1 | View all student resources |
| US-2 | View all teacher resources |
| US-3 | View all room resources |
| US-4 | Create and manage courses + course versions |
| US-5 | Create and manage classes + class members |
| US-6 | Assign teachers and students to classes |
| US-7 | Create and manage study plans (class_study_plan) |
| US-8 | Run scheduling proposals and confirm baselines |
| US-9 | Manage schedule baseline versions (is_current flip) |
| US-10 | View today's planned sessions and execution |
| US-11 | View and manage attendance records |
| US-12 | Approve makeup sessions |
| US-13 | Handle transfer requests (student/teacher between classes) |
| US-14 | Review course/material changes |
| US-15 | Configure time rules and scheduling strategy |

### Scheduling Notes (v2.4)

**Preferred Room**: Scheduling supports Preferred / Avoid / Required room constraints. UI should distinguish these levels — Preferred is a suggestion, Required is a hard constraint, Avoid blocks assignment. Per IA Spec §2.1.

**Hybrid Delivery**: `session_delivery_channel` supports multiple channels per class_session. Academic director can configure default teaching modes per class, which flow through to Planned Sessions and Class Sessions.

---

## 6-7. API & Permissions (Summary)

### Academic APIs
All `/api/academic/*` endpoints with `academic.*` tokens.

### CRM Read-only
`crm.prospect:read`, `crm.conversion:read`, `crm.task:read` (academic tasks), `crm.student_profile:read`

---

## 8. Backend Service Rules

```
RULE-BASELINE-1: Baseline is immutable — create new version, flip is_current
RULE-BASELINE-2: Formal class sessions belong to Academic/Execution, NOT CRM marketing_activity
RULE-SCOPE: All operations institution-scoped
RULE-SCHEDULING: Scheduling runs are idempotent — same input produces same output
```

---

## 9. Database Objects

**Read/Write**: `academic.course`, `academic.course_version`, `academic.class`, `academic.class_member`, `academic.schedule_baseline`, `academic.planned_session`, `academic.study_plan`, `execution.class_session`, `execution.attendance`, `execution.scheduling_run`, `core.resource`, `core.room`, `time.*`

**Read-only**: `org.person` (students/teachers), `contract.contract`

---

## 10-13. (See Full Spec)

Full page-level functions, acceptance tests, and out-of-scope items deferred to detailed implementation phase.
