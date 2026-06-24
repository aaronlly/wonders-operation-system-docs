# Initial School Setup, Principal Administration, and Prospect Management Flow — v1.5 Final Requirements Frozen

## 1. Purpose

本文档定义 Wonders Academy 新架构下，从平台初始化学校，到校长登录，再到校长建立学校内部角色、管理线索的完整业务闭环。

该流程覆盖以下关键问题：

1. `SaaSOperator` 在新架构中的职责是否变化；
2. 校长账号如何创建、绑定、登录和修改初始密码；
3. 校长是否有权限创建本校内部角色和用户；
4. 旧官网后台页面 `admin-login.html`、`admin-prospect.html` 在新架构下如何替代；
5. 官网表单线索如何进入新的 CRM 表；
6. 校长、sales、教务、运营等角色如何在 Admin Workspace 下协同工作。

---

## 2. Core Architectural Decision

### 2.1 SaaSOperator 仍然负责平台级初始化

`SaaSOperator` 继续作为平台级初始化工具使用。  
其职责不迁移到官网，也不迁移到 CRM。

`SaaSOperator` 负责：

- 创建学校 / institution；
- 创建第一位校长 / principal；
- 创建校长登录账号；
- 将校长绑定到学校；
- 授予校长 `principal` role；
- 触发初始密码邮件发送；
- 完成学校首次启用前的 tenant provisioning。

### 2.2 校长是本校级管理员，不是平台级管理员

校长登录新系统后，是该学校的 **school-level administrator**。

校长可以管理本校内部人员、账号和角色，例如：

- 教务负责人 / `academic_director`
- 销售 / `sales`
- 市场 / `marketing`
- 运营 / `operations`
- 财务 / `finance`
- 教师 / `teacher`
- 其他学校工作人员

但校长不能执行平台级操作，例如：

- 创建其他学校；
- 删除或修改其他学校；
- 管理全局 SaaS 配置；
- 查看其他学校数据；
- 操作平台级 SaaSOperator 后台。

### 2.3 旧官网后台页面被新 SPA 替代

旧入口：

```text
/admin-login.html
/admin-prospect.html
```

新入口：

```text
/app/login
/app/admin/crm/prospects
```

最终跳转关系：

```text
/admin-login.html     → /app/login
/admin-prospect.html  → /app/admin/crm/prospects
```

旧页面不再承载正式业务能力，只作为 legacy redirect 或过渡页面存在。

---

## 3. High-Level Flow Overview

完整流程如下：

```text
Platform Operator / SaaSOperator User
  ↓
SaaSOperator
  ↓
Create Institution + Principal
  ↓
Send Initial Password Email
  ↓
Principal logs in via /app/login
  ↓
Forced Password Change
  ↓
Session Context resolves Admin Workspace
  ↓
Principal enters /app/admin/*
  ↓
Principal creates internal school users and roles
  ↓
Sales / Marketing / Operations manage CRM leads
  ↓
Public website forms write to crm.prospect
  ↓
Leads are managed in /app/admin/crm/prospects
```

---

## 4. Step-by-Step Flow

## Step 1 — SaaSOperator 初始化学校与校长账号

平台运维人员在 `SaaSOperator` 项目中完成学校初始化。

项目路径：

```text
D:\dev\LLYProjects\ClassingSystem\SaaSOperator
```

SaaSOperator 创建以下核心对象：

```text
org.education_institution
org.person
auth.user_account
org.institution_member
org.institution_role_assignment
```

具体动作包括：

| 动作 | 目标对象 | 说明 |
|---|---|---|
| 创建学校 | `org.education_institution` | 建立学校 / institution 主记录 |
| 创建校长 person | `org.person` | 校长作为正式 person 进入统一人员体系 |
| 创建校长账号 | `auth.user_account` | 用于 `/app/login` 登录 |
| 绑定校长到学校 | `org.institution_member` | 表示该 person 属于该 institution |
| 授予校长角色 | `org.institution_role_assignment(role_code = 'principal')` | 校长获得本校级管理权限 |
| 发送初始密码邮件 | `email.email_template` + `email.email_log` | 通知校长账号和初始登录方式 |

---

## Step 2 — 系统发送校长初始密码邮件

学校与校长账号创建完成后，系统应向校长发送初始密码邮件。

该邮件不属于 in-app notification，也不属于旧 messaging 模块，而应使用统一 email infrastructure。

涉及对象：

```text
email.email_template
email.email_log
email.email_account
email.email_delivery
```

邮件内容应包括：

- 登录入口；
- 登录账号；
- 初始密码；
- 首次登录必须修改密码的说明；
- 安全提示；
- 学校名称；
- 联系支持方式。

推荐登录入口：

```text
/app/login
```

如仍需兼容旧入口，可提示：

```text
/admin-login.html → /app/login
```

但正式文案应以 `/app/login` 为准。

---

## Step 3 — 校长账号进入“首次登录必须改密码”状态

校长账号创建后，应标记为首次登录必须修改密码。

推荐账号状态字段：

```text
must_change_password = true
```

或者使用等价状态：

```text
password_status = INITIAL
```

推荐逻辑：

```text
Create principal user account
  ↓
Set temporary password
  ↓
Set must_change_password = true
  ↓
Send initial password email
```

该状态应由 Auth 模块在登录时检查。

---

## Step 4 — 校长通过统一入口登录

校长访问：

```text
/app/login
```

登录流程：

```text
/app/login
  ↓
POST /api/auth/login
  ↓
Auth module validates login_name + password + institution_code
  ↓
If must_change_password = true
  ↓
Redirect to forced password change flow
```

旧入口处理：

```text
/admin-login.html → /app/login
```

---

## Step 5 — 校长首次登录后强制修改密码

校长使用初始密码登录后，系统检测到：

```text
must_change_password = true
```

此时校长不能直接进入业务系统，而应进入强制修改密码流程。

推荐流程：

```text
Initial password login
  ↓
Force password change page
  ↓
Input new password
  ↓
Update password hash
  ↓
Set must_change_password = false
  ↓
Invalidate old temporary password
  ↓
Login again with new password
  ↓
Enter unified business platform
```

是否要求重新登录可由实现决定。

推荐安全策略：

```text
修改密码后重新登录
```

这样可以保证：

- 初始临时 token 不进入正式业务系统；
- 新密码生效后重新签发正式 session；
- 审计日志更清楚。

---

## Step 6 — Session Context 决定校长可访问的 Workspace

校长使用新密码登录成功后，前端调用：

```text
GET /api/session/context
```

系统根据以下数据计算校长可访问内容：

```text
auth.user_account
org.person
org.institution_member
org.institution_role_assignment
```

> **Note**: Below is a **minimal illustrative payload** showing the structure only. The full principal session context is defined in Role & Permission Matrix §21.2, which is authoritative. The complete set includes `crm.prospect.archive`, `crm.marketing_activity.add`, `email.send`, `notification.broadcast`, and additional CRM permissions.

示例返回：

```json
{
  "current_workspace": "admin",
  "available_workspaces": ["admin", "academic"],
  "account_scope": "institution",
  "institution_id": "...",
  "person_id": "uuid",
  "user_account_id": "uuid",
  "role_codes": ["principal"],
  "platform_role_codes": [],
  "member_type": ["staff"],
  "permissions_by_workspace": {
    "admin": {
      "org.person": ["read", "add", "modify", "deactivate"],
      "org.institution_member": ["read", "add", "modify", "deactivate"],
      "org.institution_role_assignment": ["read", "assign", "revoke"],
      "auth.user_account": ["read", "invite", "reset_password", "deactivate"],

      "crm.prospect": ["read", "add", "modify", "assign_owner", "change_status", "add_communication", "convert_to_student", "archive", "export"],
      "crm.student_profile": ["read", "modify"],
      "crm.marketing_activity": ["read", "add", "modify", "publish", "cancel", "register", "checkin"],
      "crm.task": ["read", "add", "modify", "assign", "complete", "cancel"],
      "crm.sales_contract": ["read", "add", "modify", "activate", "terminate"],
      "crm.sales_payment": ["read", "add", "void", "export"],
      "crm.renewal_reminder": ["read", "add", "modify", "cancel", "resolve", "dismiss"],
      "crm.churn_warning": ["read", "add", "modify", "resolve"],
      "crm.communication": ["read", "add_communication", "send", "export"],
      "crm.tag": ["read", "add", "modify", "delete", "assign"],
      "crm.conversion": ["read", "add", "modify", "move_stage", "assign_owner", "archive"],
      "crm.conversion_stage": ["read", "add", "modify", "delete"],

      "email.send": ["send"],
      "email.log": ["read"],
      "notification": ["read"],
      "notification.broadcast": ["send"]
    }
  }
}
```

---

## Step 7 — 校长进入 Admin Workspace

校长默认进入：

```text
/app/admin/...
```

Admin Workspace 下主要模块包括：

```text
/app/admin/messages/*
/app/admin/org/*
/app/admin/crm/*
/app/admin/email/*
/app/admin/notifications/*
/app/admin/audit
```

校长不是进入独立的 principal workspace，也不是进入 sales workspace。  
校长属于：

```text
workspace = admin
role_code = principal
```

---

## Step 8 — 校长创建和管理本校内部人员与角色

校长进入 Admin Workspace 后，可以在本校范围内管理内部人员、账号和角色。

主要入口：

```text
/app/admin/org/persons
/app/admin/org/members
/app/admin/org/users
/app/admin/org/roles
```

### 8.1 校长可以创建的内部角色

校长可以创建或邀请本校工作人员，并授予以下角色/成员类型：

| Role / Member Type | 中文含义 | 典型职责 | 说明 |
|---|---|---|---|---|
| `people_admin` | 人员与账号管理员 | 日常 HR / 人员管理、账号创建、有限角色分配 | v1.5.1 新增；`org.institution_role_assignment` |
| `academic_director` | 教务负责人 | 课程、班级、教学安排、教师协调 | `org.institution_role_assignment` |
| `operations` | 运营 | 日常管理、流程执行、CRM 协同 | `org.institution_role_assignment` |
| `marketing` | 市场 | 活动、招生推广、营销线索 | `org.institution_role_assignment` |
| `sales` | 销售 | 跟进线索、转化学生、销售合同 | `org.institution_role_assignment` |
| `finance` | 财务 | 付款、账务、合同收款查看 | `org.institution_role_assignment` |
| `teacher` | 教师 | 课程、学生、消息、作业、考勤 | `org.institution_member.member_type` = teacher, **not** an institution role |
| `it_admin` | 技术管理员 | 本校技术配置，是否开放由策略决定 | `org.institution_role_assignment` |

### 8.2 校长可以执行的学校级管理操作

推荐权限范围：

```text
org.person: read, add, modify
org.institution_member: read, add, modify, deactivate
org.institution_role_assignment: read, assign, revoke
auth.user_account: read, invite, reset_password, deactivate
```

> **v1.5.1**: 校长可委派 `people_admin` 负责日常 HR / 账号管理工作。people_admin 可分配的角色仅限于 sales、marketing、teacher、staff，不能分配 principal、academic_director、operations、finance、it_admin。

校长可以：

- 新建本校人员；
- 邀请本校员工登录；
- 给员工分配角色；
- 修改员工角色；
- 停用离职员工账号；
- 重置员工密码；
- 查看本校组织结构；
- 查看本校 CRM 和必要运营数据。

### 8.3 校长不能执行的平台级操作

校长不能：

```text
创建其他 institution
删除 institution
修改其他学校信息
访问其他学校数据
管理 SaaSOperator 平台后台
修改全局枚举或系统级配置
查看平台级审计日志
```

权限边界：

```text
principal = school-level administrator
saas_operator = platform-level operator
```

---

## Step 9 — 校长或被授权的 Sales 管理 CRM 线索

校长可以自己查看线索，也可以创建 sales 角色，让销售人员管理线索。

CRM 线索入口：

```text
/app/admin/crm/prospects
```

该页面替代旧页面：

```text
/admin-prospect.html
```

最终跳转：

```text
/admin-prospect.html → /app/admin/crm/prospects
```

---

## Step 10 — 官网表单写入新的 CRM Prospect 表

官网 public form 提交到：

```text
/api/public/**
```

后端写入：

```text
crm.prospect
```

旧表：

```text
prospect.prospect
```

在 pre-production rebuild 中废弃，不再作为正式业务表使用。

完整线索链路：

```text
Public Website Form
  ↓
/api/public/**
  ↓
crm.prospect
  ↓
/app/admin/crm/prospects
  ↓
Principal / Sales / Marketing follow-up
```

---

## Step 11 — 线索转化为正式学生

当线索成熟后，校长、sales 或 operations 可执行转化流程。

入口：

```text
/app/admin/crm/prospects/:id/convert
```

转化动作：

```text
crm.prospect
  ↓
student org.person
  ↓
student org.institution_member
  ↓
student core.resource
  ↓
crm.student_profile
  ↓
optional: guardian org.person
  ↓
org.guardian_assignment
  ↓
optional: guardian auth.user_account
  ↓
optional: auth.user_workspace_binding(workspace_type = guardian)
```

Guardian 处理支持三种模式：

| 模式 | 行为 | 场景 |
|------|------|------|
| **Minimal** | 只创建学生，不处理 guardian | 大学自招、成人学员 |
| **Standard**（默认） | 创建 guardian person + guardian_assignment | 青少年学员 — 家长信息在 prospect 表单中收集 |
| **Full family onboarding** | Standard + 创建 guardian login account + workspace binding | 学校要求家长登录 portal 时 |

Standard 模式下，guardian_name / email / phone / wechat 从 `crm.prospect` 对应字段映射到 `org.person` + `org.guardian_assignment`。家长登录账号可后续由 principal 主动邀请。

转化后：

```text
crm.prospect.person_id = org.person.id
crm.prospect.status = CONVERTED
```

沟通记录保持原始阶段：

```text
crm.communication.entity_type = 'prospect'
crm.communication.entity_id = crm.prospect.id
```

正式学生页查询历史时，通过：

```text
crm.prospect.person_id
```

把线索阶段沟通记录合并显示。

---

## 5. Legacy Page Replacement

## 5.1 `admin-login.html`

旧功能：

```text
/admin-login.html
```

新功能：

```text
/app/login
```

最终处理：

```text
/admin-login.html → /app/login
```

### 5.2 `admin-prospect.html`

旧功能：

```text
/admin-prospect.html
```

新功能：

```text
/app/admin/crm/prospects
```

最终处理：

```text
/admin-prospect.html → /app/admin/crm/prospects
```

### 5.3 Legacy 页面不再承载正式业务

旧页面只允许作为：

- redirect；
- 临时兼容入口；
- 过渡期提示页。

不应继续在旧页面中维护新的 CRM 功能。

---

## 6. Role and Permission Model

## 6.1 Platform-Level Operator

平台级操作者在 SaaSOperator 中工作。

职责：

```text
Create institution
Create first principal
Bind principal to institution
Assign principal role
Trigger initial password email
```

权限范围：

```text
platform-wide
multi-institution
tenant provisioning
```

## 6.2 Principal

校长是本校级管理员。

职责：

```text
Manage school internal users
Manage school members
Manage school role assignments
View and manage CRM leads
Assign sales / marketing / operations / academic roles
Monitor school-level operations
```

权限范围：

```text
institution-scoped only
```

## 6.3 Sales

由校长创建或邀请。

职责：

```text
View prospects
Follow up leads
Add communications
Move conversion pipeline
Create sales contracts
Track payments and renewals
```

典型入口：

```text
/app/admin/crm/prospects
/app/admin/crm/sales-contracts
/app/admin/crm/communications
```

## 6.4 Academic Director

由校长创建或邀请。

职责：

```text
View students
Coordinate classes
Review academic-related student status
Access academic workspace if allowed
```

## 6.5 Operations

由校长创建或邀请。

职责：

```text
Full CRM operations
Activity management
Task assignment
Renewal reminders
Churn warnings
Operational coordination
```

## 6.6 Teacher

由校长或教务创建 / 邀请。

职责：

```text
Access own classes
View own students
Send or receive class-related messages
View own delivery status when email/message feature is implemented
```

---

## 7. Recommended Principal Permission Set

> **以下权限仅用于首次 onboarding 所需最低权限，不是完整的 principal 权限定义。** 完整 principal 权限以 `Wonders_Academy_Role_Permission_Matrix_v1.0.md`（Role & Permission Matrix）为准。

建议 principal 的权限集至少包括：

```text
admin.workspace: access

org.person: read, add, modify
org.institution_member: read, add, modify, deactivate
org.institution_role_assignment: read, assign, revoke
auth.user_account: read, invite, reset_password, deactivate

crm.prospect: read, add, modify, assign_owner, change_status, add_communication, convert_to_student, archive
crm.student_profile: read
crm.marketing_activity: read
crm.sales_contract: read
crm.communication: read, add_communication

email.log: read
email.send: send
notification: read
notification.broadcast: send
audit.school: read
```

不建议 principal 拥有：

```text
saas.institution.create
saas.institution.delete
saas.global_config.modify
cross_institution_data.read
platform_audit.read
```

---

## 8. API and Route Summary

## 8.1 SaaSOperator Initialization

SaaSOperator 内部 API 可保持独立，不暴露给普通学校 admin。

典型操作：

```text
Create institution
Create principal person
Create principal user account
Bind member
Assign principal role
Send initial password email
```

## 8.2 Unified Login

```text
POST /api/auth/login
GET  /api/session/context
POST /api/auth/change-first-time-password   -- initial forced password change (requires must_change_password = true)
POST /api/auth/change-password              -- normal password change (requires old password)
```

## 8.3 Admin Organization

```text
GET/POST/PUT /api/admin/org/persons/**
GET/POST/PUT /api/admin/org/members/**
GET/POST/PUT /api/admin/org/users/**
GET/POST/PUT /api/admin/org/roles/**
```

Frontend routes:

```text
/app/admin/org/persons
/app/admin/org/members
/app/admin/org/users
/app/admin/org/roles
```

## 8.4 CRM Prospect Management

```text
/api/admin/crm/prospects/**
```

Frontend routes:

```text
/app/admin/crm/prospects
/app/admin/crm/prospects/:id
/app/admin/crm/prospects/:id/convert
```

## 8.5 Public Website Lead Submission

```text
/api/public/**
  ↓
crm.prospect
```

---

## 9. Data Object Summary

| Business Concept | Target Object |
|---|---|
| 学校 | `org.education_institution` |
| 校长 person | `org.person` |
| 校长登录账号 | `auth.user_account` |
| 校长属于学校 | `org.institution_member` |
| 校长角色 | `org.institution_role_assignment(role_code='principal')` |
| 初始密码邮件模板 | `email.email_template` |
| 初始密码邮件记录 | `email.email_log` |
| 官网线索 | `crm.prospect` |
| CRM 沟通记录 | `crm.communication` |
| 正式学生身份 | `org.person` |
| 学生 CRM 扩展 | `crm.student_profile` |
| 销售合同 | `crm.sales_contract` |
| 正式履约合同 | `contract.contract` |

---

## 10. Implementation Checklist

## 10.1 SaaSOperator

- [ ] 创建 institution；
- [ ] 创建 principal person；
- [ ] 创建 principal user account；
- [ ] 创建 institution membership；
- [ ] 创建 principal role assignment；
- [ ] 生成初始密码；
- [ ] 设置 `must_change_password = true`；
- [ ] 调用 email infrastructure 发送初始密码邮件；
- [ ] 写入 email log。

## 10.2 Auth

- [ ] `/app/login` 支持 principal 登录；
- [ ] `POST /api/auth/login` 检查 `must_change_password`；
- [ ] 支持强制修改密码页面；
- [ ] 修改密码后清除 `must_change_password`；
- [ ] 修改密码后重新登录或重新签发 session；
- [ ] `GET /api/session/context` 返回 admin workspace 和 principal 权限。

## 10.3 Admin Organization

- [ ] `/app/admin/org/persons` 可用；
- [ ] `/app/admin/org/members` 可用；
- [ ] `/app/admin/org/users` 可用；
- [ ] `/app/admin/org/roles` 可用；
- [ ] principal 可以创建本校内部 staff；
- [ ] principal 可以分配本校内部 role；
- [ ] principal 权限限制在本 institution 内。

## 10.4 CRM Prospect

- [ ] `/app/admin/crm/prospects` 可用；
- [ ] `/api/admin/crm/prospects/**` 可用；
- [ ] 官网表单写入 `crm.prospect`；
- [ ] 旧 `prospect.prospect` 不再作为正式业务表使用；
- [ ] `/admin-prospect.html` 跳转到 `/app/admin/crm/prospects`。

## 10.5 Legacy Redirect

- [ ] `/admin-login.html` 跳转到 `/app/login`；
- [ ] `/admin-prospect.html` 跳转到 `/app/admin/crm/prospects`；
- [ ] 旧页面不再维护正式业务逻辑。

---

## 11. Final Target Flow

最终目标流程如下：

```text
Platform Operator / SaaSOperator User
  ↓
SaaSOperator
  ↓
Create Institution + Principal
  ↓
Send Initial Password Email
  ↓
Principal logs in via /app/login
  ↓
Force password change
  ↓
GET /api/session/context
  ↓
Principal enters /app/admin/*
  ↓
Principal creates internal users and roles
  ↓
Sales / Marketing / Operations manage prospects
  ↓
Public website form writes to crm.prospect
  ↓
Prospects managed under /app/admin/crm/prospects
  ↓
Qualified prospect converted to student
```

---

## 12. Architecture Boundary Summary

```text
SaaSOperator
  = platform-level tenant provisioning

/app/login
  = unified login entry

/app/admin/org/*
  = school-level people, users, members, roles management

/app/admin/crm/prospects
  = CRM lead pool and prospect management

/api/public/**
  = public website form submission

crm.prospect
  = canonical lead / prospect table

/admin-login.html
/admin-prospect.html
  = legacy redirects only
```

---

## 13. Key Principle

新架构不是简单地把 `admin-prospect.html` 换个地址，而是把旧的官网临时后台能力正式纳入统一业务平台：

```text
Old:
SaaSOperator → /admin-login.html → /admin-prospect.html

New:
SaaSOperator
  → /app/login
  → /app/admin/org/* for school internal setup
  → /app/admin/crm/prospects for lead management
```

因此，新的初始运营闭环应定义为：

```text
平台初始化学校和校长；
校长登录后管理本校组织和角色；
本校 sales / marketing / operations 在 CRM 中管理线索；
官网表单统一写入 crm.prospect。
```
