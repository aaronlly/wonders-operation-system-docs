# ClassingSystem / Wonders Academy UI & IA Specification v2.4

**Document Type**: UI / Information Architecture Specification
**Version**: v2.4-aligned (Database migration baseline: v2.4)
**Scope**: Web-first UI layout, workspace IA, admin role menu boundaries, Top Bar behavior
**Baseline**: v1.5 Final Requirements Frozen + people_admin + v2.4 DB schema

---

## 1. Purpose, Scope & Conventions

### Purpose

This document defines the **Information Architecture (IA)** and **UI layout** for the ClassingSystem React SPA (`/app/*`). It does NOT cover:
- Backend API contracts (see `Wonders_Academy_Role_Permission_Matrix`)
- Route-to-API mapping (see `CRM_Integration_Architecture`)
- Role permission tokens (see `Role_Function_Specs/*`)
- Database schema (see Flyway migrations)

### Scope

```text
React SPA (/app/*) only.
Public Website (static HTML) is out of scope.
SaaSOperator (static HTML) is out of scope.
Backend (/api/*) is out of scope.
```

### Conventions

- **Workspace** = URL prefix (e.g. `/app/admin`). Not a role.
- **Role** = determines menu visibility + permission tokens within a workspace.
- Sidebar menus are **dynamically generated** from `GET /api/session/context` → `permissions_by_workspace`.
- All pages enforce: route guard → permission check → data scope filter.

### Login Entry

```
/app/login
```

- All 5 workspaces share a single login page.
- After login, user is routed to their default workspace based on `available_workspaces` and `role_codes`.
- If user has only 1 workspace, enter directly.
- If user has ≥2 workspaces, enter the last-used workspace (or default per role).
- The login page always shows "当前学校" section (school name + institution code) — loaded from session context or default config.

**Platform user (saas_admin) login**：

SaaSOperator 是一个独立应用，**逻辑上高于 WondersAcademy 整个系统**。它不属于 5 个 workspace，也不属于任何单一学校。

```text
                    ┌──────────────────────┐
                    │    SaaSOperator       │  ← 平台级：卖给多个学校
                    │  (独立应用，非 SPA)     │
                    └──────┬───────────────┘
                           │ 创建学校 A、任命校长A
                           │ 创建学校 B、任命校长B
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ 学校 A    │ │ 学校 B    │ │ 学校 C    │
        │ 5 workspace│ │ 5 workspace│ │ ...       │
        └──────────┘ └──────────┘ └──────────┘
```

平台用户（saas_admin）在 SaaSOperator 中：
- 创建/管理多所学校
- 为每所学校任命校长
- 查看所有学校的校长和基本信息
- **不进入**任何学校的 5 个 workspace 操作 CRM、人员、教学等业务数据

平台用户不是 platform role 中的 `system_maintainer`（技术维护）。saas_admin 是商业运营角色。

---

## 2. 信息架构（IA）概览（v2.4-aligned）

本系统采用五个正式 Workspace：

```text
academic
teacher
student
guardian
admin
```

Workspace 不是 Role。  
Role 决定用户在当前 Workspace 中能看到哪些菜单、能调用哪些 API、能操作哪些数据。

---

## 2.1 教务端 IA（Academic Workspace）

教务端是教学、资源、排课与执行管理的主工作区，负责课程计划、班级结构、排课基线、上课执行和教学内容审核。

```text
/app/academic/*
```

### 教务端一级导航

- Dashboard（教务态势 / 今日执行 / 待处理事项）
- 我的消息
  - 收件箱
  - 未读消息
  - 消息详情
- 资源管理
  - 学生资源
  - 教师资源
  - 教室资源
- 班级管理
  - 班级列表
  - 班级详情
  - 班级成员
  - 班级基线
- 排课管理
  - Scheduling Runs（排课任务）
  - Scheduling Proposals（候选方案）
  - 进度与确认
- 课表与执行
  - Schedule Baselines（课表版本）
  - Planned Sessions（计划课次）
  - Class Sessions（执行课次）
  - 执行历史
  - 补课管理
- 教学审核
  - 学习计划审核
  - 课程变更审核
  - 教学材料变更审核
- 合同与履约
  - 合同列表
  - 合同详情
  - 结算信息
- 系统配置
  - 时间规则
  - 排课策略
  - 通知策略
  - 审计日志

### 教务端边界

教务端管理的是正式教学域：

```text
formal class
schedule baseline
class session
attendance
makeup
academic review
```

不管理：

```text
CRM prospect
marketing activity (commercial campaigns, events)
sales contract
email account
platform role
```

营销活动、线索转化、销售合同归 Admin / CRM；正式课程、排课与上课执行归 Academic。

> **注意**：Admin CRM 下的 "Marketing Activities"（营销活动）指商业推广、招生活动、campaign。Academic 下的课程、排课、班级是教学域，两者是不同概念。不要因命名相似而混淆。

---

## 2.2 教师端 IA（Teacher Workspace）

教师端是教学执行端，只服务于教师自己的课程、班级、学生、作业、考勤、课堂记录和消息。

```text
/app/teacher/*
```

### 教师端一级导航

- 首页
  - 今日课程
  - 明日课程
  - 待填写课堂记录
  - 待处理作业
  - 消息提醒
- 我的消息
  - 收件箱
  - 未读消息
  - 消息详情
- 我的课程
  - 课程列表
  - 课程详情
  - 课堂记录
  - 作业管理
  - 教学建议提交
- 出勤记录
  - 出勤汇总
  - 出勤明细
- 替代记录
  - 我作为替代教师
  - 我被替代的记录
- 授课关系终止记录
  - 终止申请
  - 终止详情

### 教师端边界

教师端只能访问：

```text
own_classes
own_courses
own_students
own_messages
own_attendance_records
```

教师不能访问：

```text
全校学生
非自己班级
CRM 线索
销售合同
付款数据
账号管理
角色分配
平台设置
```

教师身份来自：

```text
org.institution_member.member_type = teacher
```

不是：

```text
org.institution_role_assignment.role_code = teacher
```

Teacher 的 Notifications 仅通过 Top Bar 铃铛图标访问，不在侧边栏显示独立菜单项。

---

## 2.3 学生端 IA（Student Workspace）

学生端是学习执行与学习权益端，只显示学生自己的课程、作业、考勤、补课、请假和状态信息。

```text
/app/student/*
```

### 学生端一级导航

- 首页
  - 今日课程
  - 明日课程
  - 作业提醒
  - 出勤提醒
  - 消息提醒
- 我的消息
  - 收件箱
  - 未读消息
  - 消息详情
- 我的课程
  - 课程列表
  - 课程详情
  - 课程考勤
  - 课程作业
  - 请假申请
  - 终止申请
- 我的补课
  - 补课列表
  - 补课详情
- 出勤记录
  - 出勤汇总
  - 出勤明细
- 学习状态
  - 终止记录
  - 复课记录

### 学生端边界

学生只能访问：

```text
self profile
self courses
self attendance
self homework
self messages
self leave requests
self termination requests
self makeups
```

学生不能访问：

```text
其他学生
教师端
教务端
管理端
CRM
合同后台
账号管理
```

学生身份来自：

```text
org.institution_member.member_type = student
```

不是 institution role。

Student 的 Notifications 仅通过 Top Bar 铃铛图标访问，不在侧边栏显示独立菜单项。

---

## 2.4 家长 / 监护人端 IA（Guardian Workspace）

家长端是 guardian self-service portal。  
家长不是学校内部角色，也不是 member_type，而是通过 guardian assignment 与学生建立关系。

```text
/app/guardian/*
```

### 家长端一级导航

- 首页
  - 关联学生概览
  - 今日 / 近期课程
  - 作业提醒
  - 出勤提醒
  - 付款提醒
  - 消息提醒
- 我的消息
  - 收件箱
  - 未读消息
  - 消息详情
- 我的孩子
  - 学生列表
  - 学生详情
  - 孩子课程
  - 孩子考勤
  - 孩子作业
  - 请假申请
  - 付款摘要

### 家长端边界

Guardian 数据范围来自：

```text
org.guardian_assignment.guardian_person_id = current_user.person_id
```

家长只能访问：

```text
linked_student
```

家长不能访问：

```text
非关联学生
Admin workspace
CRM
教师端
教务端
学校内部人员管理
```

---

## 2.5 管理端 IA（Admin Workspace）

管理端是学校治理、组织权限、CRM、Email、Notifications、Audit 的统一后台。

```text
/app/admin/*
```

Admin 不是单一角色。不同角色在 Admin Workspace 中看到不同菜单：

```text
principal
people_admin
operations
sales
marketing
finance
it_admin
academic_director limited
```

### 管理端一级导航

```text
管理端 Admin
├─ Dashboard
│  ├─ 统计概览
│  ├─ 待办任务
│  └─ 快捷入口
│
├─ 我的消息
│  ├─ 收件箱
│  ├─ 未读消息
│  └─ 消息详情
│
├─ 组织与权限
│  ├─ 机构信息
│  ├─ 校区管理
│  ├─ 人员档案
│  ├─ 成员管理
│  ├─ 教室管理
│  ├─ 角色管理
│  └─ 用户账户
│
├─ CRM
│  ├─ 线索池 Prospects
│  ├─ 学生 CRM Profile
│  ├─ 学生详情 - Guardian Links
│  ├─ 营销活动
│  ├─ 任务
│  ├─ 转化 Pipeline
│  ├─ 转化阶段配置
│  ├─ 销售合同
│  ├─ 销售合同付款
│  ├─ 续费提醒
│  ├─ 流失预警
│  ├─ 沟通记录
│  └─ 标签管理
│
├─ Email
│  ├─ SMTP Accounts
│  ├─ Email Templates
│  ├─ Email Logs
│  └─ Delivery Status
│
├─ Notifications
│  ├─ 通知收件箱
│  ├─ 通知详情
│  └─ Broadcast
│
└─ Audit
   └─ 审计日志
```

### 管理端角色菜单裁剪

#### principal

可见：

```text
Dashboard
Messages
Org / persons
Org / members
Org / users
Org / roles
Org / institutions
CRM prospects
CRM students
CRM marketing-activities
CRM tasks
CRM conversions
CRM conversion-stages
CRM sales-contracts
CRM sales-payments
CRM renewals
CRM churn-warnings
CRM communications
CRM tags
Email logs / send email
Notifications inbox
Notifications broadcast
Audit
```

不可见：

```text
Platform admin
SaaSOperator
Other institutions
Email accounts (management)
Email templates (modify)
Platform role assignment
```

#### people_admin

可见：

```text
Dashboard
Messages
Org / persons
Org / members
Org / users
Org / roles limited
Notifications inbox
```

默认不可见：

```text
CRM all
Email accounts
Email templates
Email logs
Delivery status
Audit all
Broadcast
Platform roles
```

people_admin 负责学校范围内日常人员、成员、账号与有限角色管理。

#### operations

可见：

```text
Dashboard
Messages
Notifications inbox
Notifications broadcast
CRM all
Marketing activities
Tasks
Renewals
Churn warnings
Communications
Tags
Email accounts
Email templates
Email logs
Delivery status
```

#### sales

可见：

```text
Dashboard
Messages
Notifications inbox
CRM prospects
CRM students limited
CRM conversions
CRM sales contracts
CRM tasks
CRM communications
CRM renewals
Tags assignment limited
Email logs own
```

数据范围默认：

```text
assigned_to = current_user.person_id
```

#### marketing

可见：

```text
Dashboard
Messages
Notifications inbox
CRM prospects marketing leads
Marketing activities
Campaign communications
Tags marketing
Email templates marketing
Email logs campaign
```

默认不执行：

```text
convert_to_student
sales payment
finance operation
org user management
```

#### finance

可见：

```text
Dashboard
Messages
Notifications inbox
Sales contracts
Sales payments
Finance-related email logs
Finance audit limited
Academic contracts optional read
```

默认不可见：

```text
Prospects
Marketing activities
Org users
Role assignment
Email accounts
```

#### it_admin

可见：

```text
Dashboard
Messages
Notifications inbox
Notifications broadcast
Org technical management
Users technical support
Email accounts
Email templates
Email logs
Delivery status
Audit
```

it_admin 是 school-level institution role，不是 platform role。

#### academic_director（Admin limited）

可见（Admin workspace 内）：

```text
Dashboard
Messages
Notifications inbox
CRM prospects read only
CRM conversions read
CRM contracts read
CRM student-profile read
```

不可见：

```text
Org / persons (read-only via academic workspace)
Org / members create/modify
Email infrastructure
Audit
Broadcast
Sales payments
```

academic_director 的主 workspace 是 academic（§2.1），Admin 为只读辅助视图。

---

### 2.6 Admin Workspace 补充规范

#### Dashboard 按角色内容

Dashboard 顶部为跨角色通用区域（今日统计），下部为角色专属区域。

| 角色 | Dashboard 专属内容 |
|------|------|
| **principal** | 全校 KPI（线索转化率、合同总金额、续费率、流失率）；待审批事项；最近操作审计 |
| **people_admin** | 本周新增人员、待激活账号数、密码过期提醒、角色分配变更日志 |
| **operations** | 全校待办任务（按优先级）、续费到期提醒、流失预警列表、今日活动 |
| **sales** | 我的待跟进线索、我的进行中转化、我的到期合同、本月业绩 |
| **marketing** | 本周新增线索来源分布、进行中活动列表、活动报名统计、campaign 邮件打开率 |
| **finance** | 本月合同总额、待收款项、近期付款记录、退款申请 |
| **it_admin** | 邮件投递失败数、重试队列、系统健康状态、最近登录失败日志 |
| **academic_director (admin)** | CRM 端仅展示通知和消息预览，完整仪表盘在 academic workspace |

#### Tag 管理权

Tag 的创建、编辑、删除权限遵循"谁有该实体类型的 CRUD 权限，谁就有 Tag 管理权"：

| 实体类型 | Tag 可管理角色 | 说明 |
|----------|-------------|------|
| `student` | principal, operations | sales 仅可 assign 已有 Tag |
| `prospect` | principal, operations, marketing | sales 仅可 assign 已有 Tag |
| `marketing_activity` | operations, marketing | |
| `sales_contract` | principal, operations | finance 仅可 read |

**原则**：Tag 管理权不单独分配。拥有该对象的 `modify` 权限，就默认拥有该对象上 Tag 的管理权。

#### Teacher 消息发送范围

教师在 Teacher workspace 内可发送消息的目标：

| 可发送给 | 条件 |
|----------|------|
| 自己的学生 | class_member JOIN core.resource WHERE resource.person_id = self AND member_role = 'student' |
| 自己学生的家长 | 学生的 `guardian_assignment` 中已关联的 guardian |
| 同班级的其他教师 | 同一 `class` 的其他 `class_member`（role = teacher） |
| academic_director | 本校的教务负责人 |

**不可发送给**：非自己班级的学生、其他学生的家长、principal、sales、全校广播。

#### Guardian 消息发送范围

家长在 Guardian workspace 内可发送消息的目标：

| 可发送给 | 条件 |
|----------|------|
| 关联孩子的教师 | 孩子的 `class_member` 中标记为 teacher |
| 学校管理员 | principal 或 operations（通过学校通用联系入口） |

**不可发送给**：其他家长、其他学生的教师、学校其他工作人员。

---

## 3. 全局 UI 布局架构（Web 优先）

## 3.1 桌面端通用布局

```text
┌─────────────────────────────────────────────┐
│ Top Bar                                     │
├──────────────┬──────────────────────────────┤
│ Side Menu    │ Main Content                 │
└──────────────┴──────────────────────────────┘
```

### Top Bar

用于全局上下文：

```text
Logo
Workspace Switcher
Global Search
Messages
Notifications
User Menu
```

### Side Menu

由当前 workspace + permissions_by_workspace 动态生成。

同一个 `/app/admin/*` 下，不同角色看到不同菜单。  
例如：

```text
principal      → Dashboard + Messages + Org + CRM + Email logs + Notifications + Broadcast + Audit
people_admin   → Dashboard + Messages + Org persons/members/users/roles + Notifications inbox
sales          → Dashboard + Messages + CRM prospects/students/conversions/contracts/tasks/renewals + Notifications inbox
marketing      → Dashboard + Messages + CRM prospects/marketing-activities/tags + Email templates/logs + Notifications inbox
finance        → Dashboard + Messages + CRM contracts/payments + Email logs + Notifications inbox
it_admin       → Dashboard + Messages + Email accounts/templates/logs/delivery + Org technical + Notifications + Broadcast + Audit
```

### Main Content

用于承载具体业务页面。  
所有页面必须经过：

```text
route guard
+
permission object/action check
+
data scope filtering
```

---

## 3.2 移动端适配原则

移动端不单独设计第二套 IA，但需要降级复杂交互：

- Side Menu → Drawer
- Top Bar 保持
- Workspace Switcher 保持相同显示规则
- 周视图 / 大表格 / 排课矩阵 → 降级为按日列表或卡片列表
- CRM Pipeline → 横向滑动或阶段列表
- 表格批量操作 → 二级页面或底部操作栏
- Calendar / Timetable → Day View 优先
- Audit / Logs → 筛选优先，表格列减少

---

# 4. Top Bar 设计规范（v2.4-aligned）

Top Bar 结构：

```text
Logo | Workspace Switcher | Global Search | Messages | Notifications | User Menu
```

---

## 4.1 Workspace Switcher

### 目的

Workspace Switcher 用于切换工作台上下文。  
切换后会影响：

```text
route prefix
side menu
permissions_by_workspace
data scope
default landing page
```

### 显示规则

旧规则中“教师端与学生端一律不显示”不再适用。

v2.4-aligned 规则：

```ts
showWorkspaceSwitcher = availableWorkspaces.length >= 2
```

原因：

- 一个用户可能同时是 teacher + guardian
- 一个用户可能同时有 admin + academic
- teacher / student 不是必须隐藏 switcher
- 是否显示只取决于用户是否真的拥有多个 workspace

### 示例

| 用户身份 | available_workspaces | 是否显示 |
|---|---|---:|
| 纯学生 | `["student"]` | ❌ |
| 纯教师 | `["teacher"]` | ❌ |
| 教师 + 家长 | `["teacher", "guardian"]` | ✅ |
| principal | `["admin", "academic"]` | ✅ |
| people_admin | `["admin"]` | ❌ |
| sales | `["admin"]` | ❌ |
| academic_director | `["academic", "admin"]` | ✅ |
| finance | `["admin", "academic"]` | 取决于配置 |

### 切换行为

切换 workspace 后：

```text
更新 current_workspace
刷新 side menu
刷新 route guard
刷新权限判断
跳转到该 workspace 默认首页
记录 last_used_workspace
```

### 默认进入规则

登录后默认进入：

1. 如果 `last_used_workspace` 存在，且当前仍有权限，则进入上一次使用的 workspace；
2. 否则按角色默认落点进入；
3. 如果用户只有一个 workspace，直接进入该 workspace。

默认落点建议：

| Workspace | 默认页 |
|---|---|
| `admin` | `/app/admin/crm/prospects` 或按角色默认页 |
| `academic` | `/app/academic/dashboard` |
| `teacher` | `/app/teacher/home` |
| `student` | `/app/student/home` |
| `guardian` | `/app/guardian/home` |

Admin workspace 内部还要按角色细分默认页：

| Role | Admin 默认页 |
|---|---|
| `principal` | `/app/admin/crm/prospects` 或 `/app/admin/org/persons` |
| `people_admin` | `/app/admin/org/persons` |
| `operations` | `/app/admin/crm/tasks` 或 `/app/admin/crm/prospects` |
| `sales` | `/app/admin/crm/prospects` |
| `marketing` | `/app/admin/crm/marketing-activities` |
| `finance` | `/app/admin/crm/sales-contracts` |
| `it_admin` | `/app/admin/email/accounts` 或 `/app/admin/org/users` |

---

## 4.2 Global Search / Global Finder

### 定位

Global Search 是对象定位搜索，不是全文检索。

它用于快速跳转到用户有权限访问的业务对象详情页。用户输入关键词后，系统跨对象类型搜索 ID、姓名、标题等关键字段，返回可直接点击跳转的结果列表。

### 搜索行为

```text
输入 ≥2 字符 → 200ms debounce → 后端搜索 → 返回匹配结果（最多每个对象类型 5 条）
```

### v2.4-aligned 搜索对象集合

最小集合：

```text
student（person_id / legal_full_name）
teacher（person_id / legal_full_name）
class（class name）
course（course name）
contract（contract_number）
room（room name / campus）
crm.prospect（legal_full_name / primary_phone）
crm.sales_contract（contract_number / student name）
crm.marketing_activity（title）
crm.conversion（prospect name → deal）
```

### 搜索结果结构

```text
[对象类型图标] [display name]
  └─ secondary info（email / phone / contract_no）
  └─ [workspace] badge → 点击跳转
```

点击后跳转到对应 workspace 内 route。如果当前 workspace 不匹配但用户有目标 workspace 权限，提示是否切换 workspace。

### 搜索结果权限裁剪

搜索结果必须同时满足：

```text
current_workspace has route access
permission object has read action
target data is within data_scope
```

示例：

- teacher 只能搜索自己的课程、自己的班级、自己的学生；
- student 只能搜索自己的课程和作业相关对象；
- guardian 只能搜索 linked students；
- sales 只能搜索 assigned prospects / own conversions / own contracts，除非配置为 all；
- principal / operations 可搜索 institution scope 内对象；
- people_admin 只能搜索 org/person/member/user 相关对象，不搜索 CRM 销售对象。

### 实现优先级

v1.5.1 Phase 1：先在 Admin workspace 实现 student / prospect / contract 三种对象搜索。
v1.5.1 Phase 2：扩展到 teacher / class / course / marketing_activity / conversion。

---

### 4.2b Delivery Status 数据范围

Email Delivery Status（投递状态）默认范围为 **institution-wide**——查看本校所有邮件投递记录。

| 角色 | Delivery Status 范围 |
|------|------|
| principal | 本校全部 |
| operations | 本校全部 |
| it_admin | 本校全部 |

**校区（Campus）筛选**：v1.5.1 暂不实现 campus 级别的 Delivery Status 筛选。校区是物理空间概念，SMTP 投递状态属于基础设施层，不按校区分割。如需校区分割，放后续版本。

---

## 4.3 Messages / Email / Notifications 三分法

系统必须严格区分：

```text
Messages
Email
Notifications
```

### Messages

Messages 是用户面对用户的站内消息 / 会话 / 收件箱。

典型入口：

```text
/app/admin/messages/*
/app/academic/messages/*
/app/teacher/messages/*
/app/student/messages/*
/app/guardian/messages/*
```

**跨 workspace 数据共享**：

Messages 默认为 **workspace-scoped**——不同 workspace 的收件箱相互隔离。

```text
person_id = user        ← 数据按人存储
workspace  = context    ← 发送/接收时标记 workspace
```

- 在 admin 端发出的消息 → 仅接收者在 admin 端可见
- 在 teacher 端发出的消息 → 仅接收者在 teacher 端可见
- 同一人在不同 workspace 看不到对方的收件箱内容

**例外**：系统级通知（密码过期、合同到期、账号停用、system announcement）标记为 `workspace = null`，在所有 workspace 的 Top Bar Notifications 中跨 workspace 可见。

Top Bar Messages 显示：

- 未读数
- 最近消息
- 快速预览
- 查看全部

### Email

Email 是 SMTP、邮件模板、邮件发送日志、投递状态等基础设施。

典型入口：

```text
/app/admin/email/accounts
/app/admin/email/templates
/app/admin/email/logs
/app/admin/email/delivery-status
```

Email 不等于 Messages。  
普通用户不直接进入 Email infrastructure。

### Notifications

Notifications 是系统触发的事件提醒和 Broadcast。

典型入口：

```text
/app/admin/notifications/inbox
/app/admin/notifications/:id
/app/admin/notifications/broadcast
```

通知类型包括：

```text
task_assigned
renewal_reminder
churn_alert
broadcast
system announcement
```

Top Bar Notifications 显示：

- 未读通知数
- 最近通知
- 点击跳转到处理页
- 查看全部

### Broadcast

Broadcast 是 institution / role-level announcement。

v1.5.1 默认可发送 Broadcast 的角色：

```text
principal
operations
it_admin
```

可选：

```text
academic_director academic-only broadcast
```

但如果没有单独设计 academic broadcast route，v1.5.1 Web UI 中建议先统一放在：

```text
/app/admin/notifications/broadcast
```

---

## 4.4 User Menu / Account & Context

用户菜单固定项：

- 我的资料
- 当前学校 / 当前 workspace 信息
- 切换校区
  - 仅当用户有多校区 context 时显示
- 通知偏好
- 安全设置
  - 修改密码
  - 会话管理
  - 登录设备
- 退出登录

### 权限相关展示

User Menu 可显示：

```text
current_workspace
role_codes
member_type
account_scope
institution name
```

但不要把完整 permission token 展示给普通用户。  
权限 token 属于开发/调试信息，可在 admin debug 或 dev mode 中展示。

---

# 5. v2.4 相对旧版 UI 文档的关键变化

1. 管理端不再只有“线索管理 v1.0 包装旧页面”，而是正式包含：

```text
Org
CRM
Email
Notifications
Audit
```

2. CRM 不再是旧 prospect 页面，而是 Admin Workspace 下的完整 CRM domain。

3. Email、Messages、Notifications 必须分开，不再混为“消息系统”。

4. Guardian Workspace 已正式加入，不是 future placeholder。

5. Workspace Switcher 不再按 teacher/student 硬隐藏，而是按：

```ts
availableWorkspaces.length >= 2
```

6. `people_admin` 加入 Admin Workspace 的组织与账号管理角色。

7. `teacher` 和 `student` 不是 role enum，而是 member_type；它们通过各自 workspace + data scope 获得功能权限。

8. `sales` 不是 workspace，而是 Admin Workspace 下的 role。

9. Public Website 与 internal SPA 继续分离：

```text
Public Website = static marketing / enrollment / public forms
React SPA = /app/*
Spring Boot API = /api/*
```

---

## 6. 下一步建议

下一步建议单独细化：

```text
Admin Workspace Sidebar by Role
```

尤其是以下角色的菜单可见性表：

```text
principal
people_admin
operations
sales
finance
it_admin
marketing
academic_director
```

该部分建议作为单独文档输出，避免本 UI / IA 总览文档继续膨胀。

### 后续版本建议

**Resource Scope Model**（已定义于 `Role_Function_Specs/00_Master_Index §3`）：当前权限模型已覆盖 SELF / ASSIGNED / INSTITUTION / PLATFORM 四级。CAMPUS 级别留待多校区部署时启用。
