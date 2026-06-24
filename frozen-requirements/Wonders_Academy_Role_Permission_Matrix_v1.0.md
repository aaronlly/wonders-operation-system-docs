# Wonders Academy Role & Permission Matrix v1.0

**Date**: 2026-06-23  
**Status**: v1.5 Final Requirements Frozen — implementation handoff baseline  
**Scope**: ClassingSystem / Wonders Academy unified business platform  
**Related Documents**:

- `Frontend_Architecture_v1.5_Final_Requirements_Frozen.md`
- `Workspace_Route_Map_v1.5_Final_Requirements_Frozen.md`
- `CRM_Integration_Architecture_v1.5_Final_Requirements_Frozen.md`
- `Initial_School_Setup_and_Principal_Administration_Flow_v1.0.md`
- `V0_2__enums.sql`
- `V2_0__classing_schema_baseline.sql`

---

# 1. Purpose

本文档定义 Wonders Academy 新架构下的角色、身份、工作区、权限对象、权限动作、数据作用域与数据库落地方案。

本文档的目标是统一以下系统层面的权限判断：

1. React SPA 前端菜单显示；
2. React Router route guard；
3. Spring Boot API 权限控制；
4. Service 层数据范围校验；
5. Repository / Query 层 institution scope 过滤；
6. SaaSOperator 平台级初始化权限；
7. 学校级 Principal Admin 权限；
8. CRM 线索、销售、市场、合同、沟通权限；
9. Academic / Teacher / Student / Guardian 各端权限；
10. Email / Notification / Messages 三类通信能力；
11. Audit 日志查看与导出权限；
12. 未来 Web / App / PWA / React Native 多端权限复用。

本文档不是单纯的“菜单可见性表”，而是正式的权限模型文档。

权限最终应由以下四层共同执行：

```text
Frontend menu / route guard
  +
Backend Spring Security / @PreAuthorize
  +
Service-level scope validation
  +
Database query-level institution / ownership filtering
```

---

# 2. Key Architecture Decisions

本轮权限设计基于以下已确认决策。

## 2.1 Workspace Enum 修改

当前数据库中的 `enum.workspace_type_enum` 为：

```sql
CREATE TYPE enum.workspace_type_enum AS ENUM (
  'academic_admin',
  'teacher',
  'student',
  'management'
);
```

该设计与前端 Route Map 中的 `/app/academic/*`、`/app/admin/*` 命名不一致，也缺少家长 / 监护人端。

最终修改为：

```sql
CREATE TYPE enum.workspace_type_enum AS ENUM (
  'academic',
  'teacher',
  'student',
  'guardian',
  'admin'
);
```

对应关系：

| Old DB Value | New DB Value | Frontend Route |
|---|---|---|
| `academic_admin` | `academic` | `/app/academic/*` |
| `teacher` | `teacher` | `/app/teacher/*` |
| `student` | `student` | `/app/student/*` |
| — | `guardian` | `/app/guardian/*` |
| `management` | `admin` | `/app/admin/*` |

## 2.2 Institution Role Enum 修改

当前数据库中的 `enum.org_institution_role_enum` 为：

```sql
CREATE TYPE enum.org_institution_role_enum AS ENUM (
  'principal',
  'academic_director',
  'people_admin',
  'finance',
  'it_admin',
  'marketing',
  'operations',
  'sales'
);
```

存在两个问题：

1. 缺少学校级 `sales` 角色；
2. `system_maintainer` 不应属于学校级 institution role。

最终修改为：

```sql
CREATE TYPE enum.org_institution_role_enum AS ENUM (
  'principal',
  'academic_director',
  'finance',
  'it_admin',
  'marketing',
  'operations',
  'sales'
);
```

变更点：

```text
+ sales
- system_maintainer
```

## 2.3 Platform Role Model 建立

`saas_operator` 和 `system_maintainer` 属于平台级角色，不属于某一个学校内部角色。

因此新增 platform role model：

```sql
CREATE TYPE enum.platform_role_enum AS ENUM (
  'saas_operator',
  'system_maintainer'
);
```

并新增平台角色分配表：

```sql
CREATE TABLE auth.platform_role_assignment (
    user_account_id uuid NOT NULL REFERENCES auth.user_account(id) ON DELETE CASCADE,
    role_code enum.platform_role_enum NOT NULL,
    assigned_from date NOT NULL,
    assigned_to date,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (user_account_id, role_code),
    CHECK (assigned_to IS NULL OR assigned_to >= assigned_from)
);
```

但由于当前 `auth.user_account.institution_id` 是 `NOT NULL`，若要真正支持 platform account，还必须同步调整 `auth.user_account`，详见第 6 章。

## 2.4 Guardian Workspace 现在纳入正式设计

`guardian` 不再是 future placeholder，而是正式 workspace。

最终 workspace 列表：

```text
academic
teacher
student
guardian
admin
```

Guardian 身份来源不是 `org_institution_role_enum`，而是：

```text
org.guardian_assignment
```

Guardian 登录需要：

```text
org.person
org.guardian_assignment
auth.user_account
auth.user_workspace_binding(workspace_type = 'guardian')
```

Guardian 可访问的数据范围由 `org.guardian_assignment.guardian_person_id = current_user.person_id` 决定。

---

# 3. Current Database Support Review

本章审查当前数据库是否能支持权限矩阵。

## 3.1 当前已支持的部分

当前 DDL 已支持：

```text
org.person
org.institution_member
org.institution_role_assignment
org.guardian_assignment
auth.user_account
auth.user_workspace_binding
auth.user_last_workspace
auth.user_last_view
```

这些表可以支持大部分学校级权限模型。

## 3.2 当前 `member_type` 支持

当前 `enum.org_institution_member_type_enum` 为：

```sql
CREATE TYPE enum.org_institution_member_type_enum AS ENUM (
  'student','teacher','staff','contractor'
);
```

含义：

| Member Type | Meaning | Should be Role Code? |
|---|---|---:|
| `student` | 学生身份 | ❌ |
| `teacher` | 教师身份 | ❌ |
| `staff` | 员工身份 | ❌ |
| `contractor` | 外部合作人员 | ❌ |

结论：

```text
teacher / student 不是 institution role，不能加入 org_institution_role_enum。
它们应继续由 org.institution_member.member_type 表达。
```

## 3.3 当前 `institution_role` 支持

当前支持：

```text
principal
academic_director
finance
it_admin
marketing
operations
system_maintainer
```

需要变更为：

```text
principal
academic_director
finance
it_admin
marketing
operations
sales
```

结论：

```text
sales 必须加入学校级 role enum。
system_maintainer 必须移出学校级 role enum。
```

## 3.4 当前 Workspace Binding 支持

当前 `auth.user_workspace_binding` 结构为：

```sql
CREATE TABLE auth.user_workspace_binding
(
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_account_id uuid NOT NULL REFERENCES auth.user_account(id) ON DELETE CASCADE,
    workspace_type enum.workspace_type_enum NOT NULL,
    related_resource_id uuid,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (user_account_id, workspace_type)
);
```

该结构基本可以支持新的五 workspace。

但注释必须修改：

```text
teacher  → teacher resource_id
student  → student resource_id
guardian → NULL; linked students resolved through org.guardian_assignment
academic → NULL
admin    → NULL
```

## 3.5 当前 Platform Role 不完整

当前 `auth.user_account` 是 institution-scoped：

```text
institution_id uuid NOT NULL
UNIQUE(login_name, institution_id)
UNIQUE(person_id, institution_id)
```

因此不能直接表达跨学校 platform account。

如果选择建立 platform role model，必须同步修改 `auth.user_account`，使其支持：

```text
institution account
platform account
```

详见第 6 章。

---

# 4. Core Permission Model

系统权限不应只按“角色名称”判断，而应由以下维度共同决定：

```text
effective_permission
  =
current_workspace
  ∩
identity_source
  ∩
role_or_member_or_relationship
  ∩
object/action
  ∩
data_scope
```

简化表达为：

```text
当前工作区
+
身份来源
+
角色 / 成员身份 / 关系身份
+
业务对象
+
操作动作
+
数据范围
```

## 4.1 为什么不能只看 role_code

因为系统中存在多种身份来源：

```text
Platform role:
  saas_operator / system_maintainer

Institution role:
  principal / academic_director / operations / sales / marketing / finance / it_admin

Institution member type:
  teacher / student / staff / contractor

Relationship identity:
  guardian
```

这些身份不能都塞进 `org_institution_role_enum`。

## 4.2 推荐权限判断公式

```text
Can(user, action, object, target)
  =
HasWorkspaceAccess(user, current_workspace)
  AND
HasActionPermission(user, object, action)
  AND
IsWithinDataScope(user, target)
  AND
PassesBusinessRule(user, action, target)
```

示例：sales 修改 prospect：

```text
Can(sales, modify, crm.prospect, prospect)
  =
workspace = admin
  AND permission contains crm.prospect.modify
  AND prospect.institution_id = user.institution_id
  AND (prospect.assigned_to = user.person_id OR user has all-prospect permission)
```

---

# 5. Workspace Model

系统正式保留五个 workspace。

```text
academic
teacher
student
guardian
admin
```

| Workspace | Route Prefix | Primary Users | Description |
|---|---|---|---|
| `academic` | `/app/academic/*` | academic_director / operations / principal | 教务、资源、课程、排课、执行管理 |
| `teacher` | `/app/teacher/*` | teacher | 教师端 |
| `student` | `/app/student/*` | student | 学生端 |
| `guardian` | `/app/guardian/*` | guardian | 监护人 / 家长端 |
| `admin` | `/app/admin/*` | principal / operations / sales / marketing / finance / it_admin | 学校管理、CRM、组织、Email、通知、审计 |

## 5.1 Workspace 不是 Role

例如：

```text
sales 不是 workspace
sales 是 admin workspace 下的 institution role
```

```text
guardian 是 workspace
guardian 不是 institution role
```

```text
teacher 是 workspace
teacher 身份来源是 member_type，不是 institution role
```

## 5.2 Workspace Binding

每个可登录用户通过 `auth.user_workspace_binding` 获得 workspace access。

推荐规则：

| Identity | Workspace Binding | related_resource_id |
|---|---|---|
| principal | `admin`, optionally `academic` | NULL |
| academic_director | `academic`, optionally `admin` | NULL |
| operations | `admin`, optionally `academic` | NULL |
| marketing | `admin` | NULL |
| sales | `admin` | NULL |
| finance | `admin`, optionally `academic` | NULL |
| it_admin | `admin`, optionally `academic` | NULL |
| teacher | `teacher` | teacher resource id |
| student | `student` | student resource id |
| guardian | `guardian` | NULL |

Guardian 不通过 `related_resource_id` 绑定具体学生，因为一个 guardian 可对应多个 linked students。

---

# 6. Current Database Modification Plan

本章定义当前数据库 DDL 需要修改的内容。

由于系统尚未商用，本轮建议直接修改 baseline DDL，而不是写复杂 ALTER migration。

## 6.1 `V0_2__enums.sql` 修改

### 6.1.1 修改 `workspace_type_enum`

替换原定义：

```sql
CREATE TYPE enum.workspace_type_enum AS ENUM (
  'academic_admin',
  'teacher',
  'student',
  'management'
);
```

为：

```sql
CREATE TYPE enum.workspace_type_enum AS ENUM (
  'academic',
  'teacher',
  'student',
  'guardian',
  'admin'
);
```

### 6.1.2 修改 `org_institution_role_enum`

替换原定义：

```sql
CREATE TYPE enum.org_institution_role_enum AS ENUM (
  'principal','academic_director','finance','it_admin','marketing','operations','system_maintainer'
);
```

为：

```sql
CREATE TYPE enum.org_institution_role_enum AS ENUM (
  'principal',
  'academic_director',
  'finance',
  'it_admin',
  'marketing',
  'operations',
  'sales'
);
```

### 6.1.3 新增 `platform_role_enum`

新增：

```sql
CREATE TYPE enum.platform_role_enum AS ENUM (
  'saas_operator',
  'system_maintainer'
);
```

### 6.1.4 新增 `auth_account_scope_enum`

为支持 platform account，新增：

```sql
CREATE TYPE enum.auth_account_scope_enum AS ENUM (
  'institution',
  'platform'
);
```

说明：

```text
institution account = 学校级账号，必须绑定 institution_id
platform account    = 平台级账号，不绑定 institution_id
```

---

## 6.2 `V2_0__classing_schema_baseline.sql` 修改

## 6.2.1 修改 `auth.user_account`

当前问题：

```text
auth.user_account.institution_id NOT NULL
```

这会导致 platform account 无法干净建模。

推荐修改为：

```sql
CREATE TABLE auth.user_account
(
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

    -- account scope
    account_scope enum.auth_account_scope_enum NOT NULL DEFAULT 'institution',

    -- association
    institution_id uuid REFERENCES org.education_institution (id),

    -- login identity
    login_name text NOT NULL,
    person_id uuid NOT NULL REFERENCES org.person (id),

    -- account state
    account_status text NOT NULL,

    -- password state
    must_change_password boolean NOT NULL DEFAULT false,
    password_status enum.auth_password_status_enum NOT NULL DEFAULT 'active',

    -- password reset
    password_reset_token text,
    password_reset_expire timestamptz,

    -- audit
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),

    -- scope constraints
    CHECK (
        (account_scope = 'institution' AND institution_id IS NOT NULL)
        OR
        (account_scope = 'platform' AND institution_id IS NULL)
    )
);
```

然后用 partial unique indexes 替代原有 table-level unique constraints：

```sql
CREATE UNIQUE INDEX ux_user_account_institution_login
ON auth.user_account (login_name, institution_id)
WHERE account_scope = 'institution';

CREATE UNIQUE INDEX ux_user_account_platform_login
ON auth.user_account (login_name)
WHERE account_scope = 'platform';

CREATE UNIQUE INDEX ux_user_account_institution_person
ON auth.user_account (person_id, institution_id)
WHERE account_scope = 'institution';

CREATE UNIQUE INDEX ux_user_account_platform_person
ON auth.user_account (person_id)
WHERE account_scope = 'platform';
```

### 6.2.2 修改登录语义

Institution account 登录：

```text
login_name + institution_code + password
```

Platform account 登录：

```text
login_name + password
```

或者：

```text
login_name + platform tenant marker + password
```

推荐在 SaaSOperator 登录页面中明确走 platform login flow，避免与学校端登录混淆。

---

## 6.2.3 新增 `auth.platform_role_assignment`

新增表：

```sql
CREATE TABLE auth.platform_role_assignment
(
    user_account_id uuid NOT NULL
        REFERENCES auth.user_account (id)
        ON DELETE CASCADE,

    role_code enum.platform_role_enum NOT NULL,

    assigned_from date NOT NULL,
    assigned_to date,

    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),

    PRIMARY KEY (user_account_id, role_code),
    CHECK (assigned_to IS NULL OR assigned_to >= assigned_from)
);
```

建议增加约束或 trigger，确保 platform role 只能分配给 platform account：

```sql
-- PostgreSQL CHECK 不能跨表检查 account_scope。
-- 因此建议在 service layer 或 trigger 中保证：
-- auth.user_account.account_scope = 'platform'
```

Service rule：

```text
Only account_scope = platform can receive auth.platform_role_assignment.
```

---

## 6.2.4 修改 `org.institution_role_assignment` 注释

当前注释中含有：

```text
legal
system_maintainer
```

应改为：

```sql
-- role_code:
-- 'principal','academic_director','finance','it_admin','marketing','operations','sales'
--
-- This table stores school-level institution roles only.
-- It does NOT store:
-- - teacher/student identities, which are represented by org.institution_member.member_type
-- - guardian identity, which is represented by org.guardian_assignment
-- - platform roles such as saas_operator/system_maintainer
```

---

## 6.2.5 修改 `auth.user_workspace_binding` 注释

修改为：

```sql
-- workspace_type:
-- 'academic','teacher','student','guardian','admin'
--
-- related_resource_id:
-- teacher  → teacher resource_id
-- student  → student resource_id
-- guardian → NULL; linked students are resolved through org.guardian_assignment
-- academic → NULL
-- admin    → NULL
```

当前 `UNIQUE(user_account_id, workspace_type)` 可以保留。

如果未来需要一个用户在同一 workspace 下拥有多个 context，再考虑改成：

```sql
UNIQUE (user_account_id, workspace_type, related_resource_id)
```

当前不建议修改，以避免 session workspace selection 复杂化。

---

## 6.2.6 修改 `auth.user_last_workspace`

当前 `auth.user_last_workspace` 通过：

```text
login_name + institution_id
```

引用 `auth.user_account`。

这对 platform account 不友好。

建议改成 `user_account_id` 外键：

```sql
CREATE TABLE auth.user_last_workspace
(
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),

    user_account_id uuid NOT NULL
        REFERENCES auth.user_account (id)
        ON DELETE CASCADE,

    workspace_type enum.workspace_type_enum NOT NULL,

    recorded_at timestamptz NOT NULL DEFAULT now(),

    UNIQUE (user_account_id)
);
```

这样 institution account 和 platform account 都可以复用同一结构。

但注意：platform role 默认不进入普通五 workspace。若 platform user 只使用 SaaSOperator，可以不写 `user_last_workspace`。

---

## 6.2.7 是否需要 Guardian Profile 表

当前阶段不强制新增。

已有表足够支持 guardian workspace：

```text
org.person
org.guardian_assignment
auth.user_account
auth.user_workspace_binding
```

未来可选新增：

```sql
CREATE TABLE org.guardian_profile (
    person_id uuid PRIMARY KEY REFERENCES org.person(id) ON DELETE CASCADE,
    preferred_contact_channel text,
    notification_opt_in boolean NOT NULL DEFAULT true,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
```

本轮不建议新增，避免过度设计。

---

# 7. Role Source Model

权限矩阵中的“角色”必须拆分为四类来源。

## 7.1 Platform Roles

来源：

```text
auth.platform_role_assignment
```

角色：

```text
saas_operator
system_maintainer
```

特点：

```text
平台级
跨学校
不属于某个 institution
不通过 org.institution_role_assignment 分配
```

## 7.2 Institution Roles

来源：

```text
org.institution_role_assignment
```

角色：

```text
principal
academic_director
finance
it_admin
marketing
operations
sales
```

特点：

```text
学校级
institution-scoped
由 principal / delegated it_admin 在学校范围内分配;
SaaSOperator 仅在 tenant provisioning 阶段分配第一位 principal
```

## 7.3 Institution Member Types

来源：

```text
org.institution_member.member_type
```

类型：

```text
student
teacher
staff
contractor
```

特点：

```text
表达成员身份
不表达管理权限
teacher / student 不应进入 institution role enum
```

## 7.4 Relationship Identities

来源：

```text
org.guardian_assignment
```

身份：

```text
guardian
```

特点：

```text
关系身份
不是学校 role
不是 member_type
访问范围由 linked students 决定
```

---

# 8. Scope Model

每个权限必须带有 scope 概念。

| Scope | 含义 | 典型使用 |
|---|---|---|
| `platform` | 平台全局，跨学校 | saas_operator / system_maintainer |
| `institution` | 当前学校范围 | principal / operations / it_admin |
| `campus` | 当前校区范围 | campus manager / operations |
| `assigned` | 被分配给自己的数据 | sales assigned prospects |
| `own` | 自己创建或负责的数据 | own tasks / own communications |
| `own_classes` | 教师自己的班级 | teacher workspace |
| `own_students` | 教师自己的学生 | teacher workspace |
| `self` | 当前登录用户本人 | student workspace |
| `linked_student` | guardian 关联学生 | guardian workspace |
| `read_only` | 只读 | principal overview / finance view |

默认原则：

```text
saas_operator / system_maintainer = platform-scoped
principal / operations / it_admin = institution-scoped
sales = institution-scoped + assigned-scoped
marketing = institution-scoped + campaign-scoped
academic_director = academic institution-scoped
teacher = own_classes / own_students scoped
student = self scoped
guardian = linked_student scoped
```

---

# 9. Action Model

统一动作定义如下。

| Action | 中文 | 说明 |
|---|---|---|
| `read` | 查看 | 查看列表、详情、摘要 |
| `add` | 新增 | 创建业务对象 |
| `modify` | 修改 | 修改已有对象 |
| `delete` | 删除 | 物理删除，原则上少用 |
| `deactivate` | 停用 | 停用账号、成员、资源 |
| `archive` | 归档 | 归档线索、历史对象 |
| `revoke` | 撤销 | 撤销角色、权限、绑定 |
| `assign` | 分配 | 分配负责人、标签、角色、任务 |
| `convert` | 转化 | 线索转正式学生 |
| `send` | 发送 | 发送邮件 / 消息 / 通知 |
| `retry` | 重试 | 重试失败邮件 |
| `export` | 导出 | CSV / Excel / PDF |
| `approve` | 审批 | 审批合同、退款、排课确认 |
| `settle` | 结算 | 财务结算 |
| `manage` | 管理配置 | 配置型对象管理 |
| `view_audit` | 查看审计 | 查看审计日志 |
| `cancel` | 取消 | 取消活动、续费提醒 |
| `resolve` | 解决 | 解决续费提醒、流失预警 |
| `dismiss` | 忽略 | 忽略续费提醒、预警 |
| `checkin` | 签到 | 活动参与者签到 |
| `register` | 报名 | 报名活动（创建参与者记录） |
| `move_stage` | 移动阶段 | 转化 deal 推进到下一阶段 |
| `void` | 作废 | 作废付款 |
| `void_draft` | 作废草稿 | 仅作废草稿合同 |
| `terminate` | 终止 | 终止合同 |
| `activate` | 激活 | 激活合同 |
| `publish` | 发布 | 发布营销活动 |
| `complete` | 完成 | 完成任务、流程 |
| `refund` | 退款 | 退款处理（含审批流程） |
| - | - | - |
| `assign_owner` | 分配负责人 | 分配线索/转化负责人 |
| `change_status` | 变更状态 | 变更线索状态机 |
| `add_communication` | 添加沟通 | 在对象上添加沟通记录 |
| `convert_to_student` | 转化为学生 | 线索→学生的转化动作 |
| `register_participant` | 报名参与者 | 等同 register（别名） |
| `assign_tag` | 分配标签 | 等同 assign（针对 tag） |
| `invite` | 邀请 | 邀请用户 |
| `reset_password` | 重置密码 | 重置用户密码 |
| `send_invitation` | 发送邀请 | 发送 guardian/person 邀请 |

推荐避免使用过于笼统的 action：

```text
full
all
admin
super
```

应拆成具体 action。

### 9.1 Canonical Token Rule

Only snake_case tokens are valid for:

- Backend `@PreAuthorize` method security
- Frontend session context `permissions_by_workspace` entries
- Permission matrix "Action" column values

Human-readable labels in tables (e.g. "assign owner to prospect") are UI labels only and MUST map to a canonical snake_case token.

```text
UI label:  "assign owner"
Canonical: assign_owner

UI label:  "register participant"
Canonical: register

UI label:  "check in participant"
Canonical: checkin

UI label:  "add communication"
Canonical: add_communication

UI label:  "convert to student"
Canonical: convert_to_student
```

The Permission Matrix tables below use canonical tokens from §9 in the "Action" column. For non-CRM matrices (§16–19) that still use descriptive labels (e.g. "view own inbox", "send internal message"), implementation teams must map each descriptive row to a canonical object/action pair before coding, following the same principle as §9.1. These workspace-specific mappings will be finalized in the respective Sprint (Academic Sprint 8, Teacher Sprint 9, Student Sprint 10).

### 9.2 Scope and Subtype Mapping Rule

Do NOT create separate action tokens for scope variants or communication subtypes. Map them as follows:

```text
Scope variants:
  read_all, read_assigned          → read (scope enforced by data_scope / @PostFilter)
  modify_assigned                  → modify (same)

Communication subtypes:
  add_note, add_call_record,
  add_meeting_record               → add_communication (type parameter in service layer)

Cross-object creation:
  create_task                      → crm.task: add
  assign_to_student,
  assign_to_activity               → assign (entity_type parameter)

Business operations:
  link_contract                    → modify (linking is a modify operation)
  refund                           → refund (kept as distinct canonical token — requires approve/process workflow)
```

This keeps the Action Model finite and prevents unbounded token growth.

---

# 10. Workspace Access Matrix

## 10.1 Platform Access Matrix

| Role | Role Source | SaaSOperator | Classing SPA | Scope | DB Support |
|---|---|---:|---:|---|---|
| `saas_operator` | `auth.platform_role_assignment` | ✅ | ❌ by default | platform | ✅ after platform role model |
| `system_maintainer` | `auth.platform_role_assignment` | ✅ | ⚠️ future technical | platform | ✅ after platform role model |

说明：

```text
saas_operator 和 system_maintainer 不属于学校内部角色。
它们不通过 org.institution_role_assignment 分配。
它们不出现在普通学校 Admin 的 role list 中。
```

## 10.2 Institution Workspace Access Matrix

| Role / Identity | Source | academic | teacher | student | guardian | admin | DB Support |
|---|---|---:|---:|---:|---:|---:|---|
| `principal` | `org.institution_role_assignment` | ✅ read/overview | ❌ | ❌ | ❌ | ✅ | ✅ |
| `academic_director` | `org.institution_role_assignment` | ✅ | ❌ | ❌ | ❌ | ✅ limited | ✅ |
| `operations` | `org.institution_role_assignment` | ✅ read/support | ❌ | ❌ | ❌ | ✅ | ✅ |
| `marketing` | `org.institution_role_assignment` | ❌ | ❌ | ❌ | ❌ | ✅ limited | ✅ |
| `sales` | `org.institution_role_assignment` | ❌ | ❌ | ❌ | ❌ | ✅ CRM | ✅ after enum update |
| `finance` | `org.institution_role_assignment` | ⚠️ contracts read | ❌ | ❌ | ❌ | ✅ finance | ✅ |
| `it_admin` | `org.institution_role_assignment` | ✅ technical | ❌ | ❌ | ❌ | ✅ technical | ✅ |
| `people_admin` | `org.institution_role_assignment` | ❌ | ❌ | ❌ | ❌ | ✅ org/users/roles limited | ✅ after enum update |
| `teacher` | `org.institution_member.member_type = teacher` | ❌ | ✅ | ❌ | ❌ | ❌ by default | ✅ |
| `student` | `org.institution_member.member_type = student` | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| `guardian` | `org.guardian_assignment` + `auth.user_workspace_binding` | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ after workspace enum update |

---

# 11. SaaSOperator and Platform Permission Matrix

## 11.1 SaaSOperator Responsibilities

SaaSOperator 负责平台级初始化，不负责学校日常运营。

| Object | Action | `saas_operator` | Notes |
|---|---|---:|---|
| `org.education_institution` | add | ✅ | 创建学校 |
| `org.education_institution` | modify | ✅ | 修改学校基础信息 |
| `org.education_institution` | deactivate | ✅ | 停用学校，需谨慎 |
| `org.person` | add principal | ✅ | 创建第一位校长 |
| `auth.user_account` | add principal account | ✅ | 创建校长登录账号 |
| `org.institution_member` | add | ✅ | 绑定校长到学校 |
| `org.institution_role_assignment` | assign principal | ✅ | 授予 principal |
| `email.initial_password` | send | ✅ | 触发初始密码邮件 |
| `auth.platform_role_assignment` | assign | ⚠️ restricted | 仅平台超级管理员或系统维护流程 |
| `crm.prospect` | read | ❌ | 不参与学校日常线索 |
| `crm.sales_contract` | read | ❌ | 不参与学校日常销售 |
| `academic.class` | read | ❌ | 不参与教务运营 |

## 11.2 System Maintainer Responsibilities

`system_maintainer` 是平台级技术维护角色。

| Object | Action | `system_maintainer` | Notes |
|---|---|---:|---|
| system health | read | ✅ | 技术诊断 |
| migration status | read | ✅ | Flyway / DB 状态 |
| audit technical logs | read | ✅ | 技术审计 |
| user business data | read | ⚠️ break-glass only | 需审计 |
| tenant data modify | modify | ❌ by default | 不应参与业务数据修改 |
| role assignment | assign | ❌ by default | 不应分配学校角色 |
| impersonation | use | ⚠️ emergency only | 若支持，必须强审计 |

## 11.3 Platform Boundary

Platform roles 可以：

```text
创建学校
创建第一位校长
初始化学校租户
触发初始密码邮件
维护平台级配置
执行技术诊断
```

Platform roles 不应该：

```text
代替校长创建本校 sales / teacher / finance 等内部角色
直接管理学校日常线索
直接操作学校销售合同
直接参与学校教务排课
绕过审计查看敏感业务数据
```

---

# 12. Principal Permission Matrix

## 12.1 Principal Positioning

`principal` 是学校级最高管理员。

```text
principal = school-level administrator
```

不是平台级管理员。

```text
principal ≠ saas_operator
principal ≠ system_maintainer
```

## 12.2 Principal Organization Permissions

| Object | Action | Scope | principal |
|---|---|---|---:|
| `org.person` | read | institution | ✅ |
| `org.person` | add | institution | ✅ |
| `org.person` | modify | institution | ✅ |
| `org.person` | delete | institution | ❌ |
| `org.person` | deactivate | institution | ✅ |
| `org.institution_member` | read | institution | ✅ |
| `org.institution_member` | add | institution | ✅ |
| `org.institution_member` | modify | institution | ✅ |
| `org.institution_member` | deactivate | institution | ✅ |
| `auth.user_account` | read | institution | ✅ |
| `auth.user_account` | invite | institution | ✅ |
| `auth.user_account` | reset_password | institution | ✅ |
| `auth.user_account` | deactivate | institution | ✅ |
| `org.institution_role_assignment` | read | institution | ✅ |
| `org.institution_role_assignment` | assign | institution | ✅ |
| `org.institution_role_assignment` | revoke | institution | ✅ |
| `auth.platform_role_assignment` | any | platform | ❌ |

## 12.3 Principal Role Assignment Permissions

Principal 可以给本校人员分配以下角色：

| Assignable Role | principal can assign? | Notes |
|---|---|---:|
| `people_admin` | ✅ | 学校级人员与账号管理员（v1.5.1 新增），负责日常 HR / 账号工作 |
| `academic_director` | ✅ | 教务负责人 |
| `operations` | ✅ | 运营 |
| `marketing` | ✅ | 市场 |
| `sales` | ✅ | 销售 |
| `finance` | ✅ | 财务 |
| `teacher` | ✅ as member type | 通过 `org.institution_member.member_type`，不是 role enum |
| `it_admin` | ⚠️ optional | 是否允许由产品策略决定 |
| `principal` | ❌ | 不允许创建第二 principal（参见 §22.5） |
| `saas_operator` | ❌ | 平台级角色 |
| `system_maintainer` | ❌ | 平台级技术角色 |

建议默认：

```text
principal 可以创建本校 staff 并分配常规学校角色。
principal 不能创建 saas_operator / system_maintainer。
principal 不能创建另一个 principal。
Only SaaSOperator or specially authorized platform/system flow can create another principal.
```

---

# 13. Admin Organization Module Permission Matrix

Admin Organization routes:

```text
/app/admin/org/institutions
/app/admin/org/institutions/:institutionId
/app/admin/org/campuses
/app/admin/org/campuses/:campusId
/app/admin/org/persons
/app/admin/org/persons/:personId
/app/admin/org/members
/app/admin/org/members/:memberId
/app/admin/org/rooms
/app/admin/org/roles
/app/admin/org/users
/app/admin/org/users/:userId
```

## 13.1 Organization Permission Matrix

| Module / Object | principal | people_admin | academic_director | operations | marketing | sales | finance | it_admin | teacher |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| View institution profile | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Modify institution profile | ✅ | ❌ | ⚠️ limited | ❌ | ❌ | ❌ | ✅ | ❌ |
| View campuses | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Manage campuses | ✅ | ❌ | ⚠️ limited | ❌ | ❌ | ❌ | ✅ | ❌ |
| View persons | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Create persons | ✅ | ⚠️ teacher/student only | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Modify persons | ✅ | ⚠️ academic fields | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Deactivate persons | ✅ | ❌ | ⚠️ staff only | ❌ | ❌ | ❌ | ✅ | ❌ |
| View members | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Create members | ✅ | ⚠️ academic staff/student | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Modify members | ✅ | ⚠️ academic only | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Deactivate members | ✅ | ❌ | ⚠️ staff only | ❌ | ❌ | ❌ | ✅ | ❌ |
| View users | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Invite users | ✅ | ❌ | ⚠️ limited | ❌ | ❌ | ❌ | ✅ | ❌ |
| Reset passwords | ✅ | ❌ | ⚠️ limited | ❌ | ❌ | ❌ | ✅ | ❌ |
| Deactivate users | ✅ | ❌ | ⚠️ limited | ❌ | ❌ | ❌ | ✅ | ❌ |
| View roles | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Assign roles | ✅ | ❌ | ⚠️ delegated only | ❌ | ❌ | ❌ | ✅ | ❌ |
| Revoke roles | ✅ | ❌ | ⚠️ delegated only | ❌ | ❌ | ❌ | ✅ | ❌ |
| View rooms | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Manage rooms | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |

### 13.1.1 `people_admin` Role Permissions

`people_admin` is a school-level people & account administrator (v1.5.1), responsible for day-to-day HR / personnel management previously handled by principal.

| Object | Action | people_admin | Notes |
|---|---|---|---|
| `org.person` | read | ✅ | institution-scoped |
| `org.person` | add | ✅ | create persons within own institution |
| `org.person` | modify | ✅ | update person fields |
| `org.person` | deactivate | ✅ | soft-deactivate persons |
| `org.person` | delete | ❌ | no physical delete |
| `org.institution_member` | read | ✅ | |
| `org.institution_member` | add | ✅ | create staff/teacher members |
| `org.institution_member` | modify | ✅ | |
| `org.institution_member` | deactivate | ✅ | |
| `auth.user_account` | read | ✅ | institution-scoped only |
| `auth.user_account` | invite | ✅ | create login accounts |
| `auth.user_account` | reset_password | ✅ | |
| `auth.user_account` | deactivate | ✅ | |
| `org.institution_role_assignment` | read | ✅ | |
| `org.institution_role_assignment` | assign | ⚠️ limited | only: sales, marketing, teacher, staff |
| `org.institution_role_assignment` | revoke | ⚠️ limited | only: sales, marketing, teacher, staff |
| `notification` | read | ✅ | own inbox only |
| `notification.broadcast` | send | ❌ | no school-wide broadcast |
| `crm.*` | any | ❌ | no CRM access by default |
| `email.account` | any | ❌ | |
| `email.delivery` | read | ❌ | |
| `audit.school` | read | ⚠️ | HR-related only |
| `auth.platform_role_assignment` | any | ❌ | |
| `org.institution_role_assignment` (people_admin) | assign | ❌ | cannot self-elevate |
| `org.institution_role_assignment` (principal) | assign/revoke | ❌ | |
| `org.institution_role_assignment` (academic_director) | assign/revoke | ❌ | |
| `org.institution_role_assignment` (operations) | assign/revoke | ❌ | |
| `org.institution_role_assignment` (finance) | assign/revoke | ❌ | |

### 13.1.2 `people_admin` Assignable Roles

people_admin can ONLY assign/revoke: `sales`, `marketing`, `teacher` (member_type), `staff` (member_type).

people_admin CANNOT assign/revoke: `principal`, `people_admin`, `academic_director`, `operations`, `finance`, `it_admin`, `saas_operator`, `system_maintainer`.

## 13.2 Organization Scope Rules

All organization queries must enforce:

```text
target.institution_id = current_user.institution_id
```

Principal / operations / it_admin 不能越权访问其他学校。

---

# 14. CRM Permission Matrix

CRM routes:

```text
/app/admin/crm/prospects
/app/admin/crm/prospects/:id
/app/admin/crm/prospects/:id/convert
/app/admin/crm/students
/app/admin/crm/students/:id
/app/admin/crm/students/:id/guardians      # Student guardian links（admin only, org.guardian_assignment）
/app/admin/crm/marketing-activities
/app/admin/crm/marketing-activities/:id
/app/admin/crm/tasks
/app/admin/crm/sales-contracts
/app/admin/crm/sales-contracts/:id
/app/admin/crm/sales-contracts/:id/payments
/app/admin/crm/renewals
/app/admin/crm/churn-warnings
/app/admin/crm/communications
/app/admin/crm/tags
/app/admin/crm/conversions
/app/admin/crm/conversions/:id
/app/admin/crm/conversion-stages
```

## 14.1 CRM High-Level Access

| CRM Module | principal | operations | sales | marketing | academic_director | finance | teacher |
|---|---:|---:|---:|---:|---:|---:|---:|
| Prospects | ✅ | ✅ | ✅ | ✅ | ⚠️ read only | ❌ | ❌ |
| Students CRM Profile | ✅ | ✅ | ✅ | ⚠️ limited | ✅ read | ⚠️ read | ❌ |
| Marketing Activities | ✅ | ✅ | ⚠️ read/register | ✅ | ⚠️ read | ❌ | ❌ |
| Tasks | ✅ | ✅ | ✅ assigned | ✅ campaign | ⚠️ academic tasks | ❌ | ❌ |
| Sales Contracts | ✅ read | ✅ | ✅ | ❌ | ⚠️ read | ✅ | ❌ |
| Sales Payments | ✅ read | ✅ | ⚠️ read own | ❌ | ❌ | ✅ | ❌ |
| Renewals | ✅ | ✅ | ✅ own | ❌ | ⚠️ read | ✅ read | ❌ |
| Churn Warnings | ✅ | ✅ | ⚠️ assigned | ❌ | ✅ read | ❌ | ❌ |
| Communications | ✅ | ✅ | ✅ assigned | ✅ campaign | ✅ read student-related | ❌ | ❌ |
| Tags | ✅ | ✅ | ⚠️ assign existing | ✅ assign existing | ❌ | ❌ | ❌ |
| Conversions | ✅ | ✅ | ✅ assigned | ⚠️ campaign-related | ⚠️ read | ❌ | ❌ |

## 14.2 `crm.prospect` Permission Matrix

| Action | principal | operations | sales | marketing | academic_director | finance | teacher |
|---|---:|---:|---:|---:|---:|---:|---:|
| read | ✅ all | ✅ all | ✅ assigned / configurable all | ✅ marketing leads | ⚠️ qualified read only | ❌ | ❌ |
| add | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| modify | ✅ | ✅ | ✅ assigned | ✅ marketing fields | ❌ | ❌ | ❌ |
| assign_owner | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| change_status | ✅ | ✅ | ✅ assigned | ⚠️ marketing stage only | ❌ | ❌ | ❌ |
| add_communication | ✅ | ✅ | ✅ assigned | ✅ campaign | ❌ | ❌ | ❌ |
| convert_to_student | ✅ | ✅ | ⚠️ configurable | ❌ | ⚠️ approval/read | ❌ | ❌ |
| delete | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| archive | ✅ | ✅ | ⚠️ own | ❌ | ❌ | ❌ | ❌ |
| export | ✅ | ✅ | ⚠️ own | ⚠️ campaign leads | ❌ | ❌ | ❌ |

Recommended default:

```text
sales 管理 assigned_to = self 的线索；
principal / operations 看全校线索；
marketing 创建和维护营销来源线索，但不执行正式转化。
```

## 14.3 `crm.student_profile` Permission Matrix

| Action | principal | operations | sales | academic_director | finance | teacher |
|---|---:|---:|---:|---:|---:|---:|
| read | ✅ | ✅ | ✅ assigned / own converted | ✅ | ⚠️ billing fields | ⚠️ own students via teacher workspace |
| add | ✅ via conversion | ✅ via conversion | ❌ direct | ❌ | ❌ | ❌ |
| modify CRM fields | ✅ | ✅ | ⚠️ sales notes only | ✅ academic fields | ❌ | ❌ |
| archive / lost | ✅ | ✅ | ⚠️ request only | ⚠️ request only | ❌ | ❌ |
| delete | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

## 14.4 `crm.marketing_activity` Permission Matrix

| Action | principal | operations | marketing | sales | academic_director | teacher |
|---|---:|---:|---:|---:|---:|---:|
| read | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ assigned activity only |
| add | ✅ | ✅ | ✅ | ❌ | ⚠️ academic event only | ❌ |
| modify | ✅ | ✅ | ✅ own/campaign | ❌ | ⚠️ academic event only | ❌ |
| publish | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| cancel | ✅ | ✅ | ✅ own/campaign | ❌ | ❌ | ❌ |
| register | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| checkin | ✅ | ✅ | ✅ | ✅ | ⚠️ event support | ⚠️ assigned only |

## 14.5 `crm.task` Permission Matrix

| Action | principal | operations | sales | marketing | academic_director | teacher |
|---|---:|---:|---:|---:|---:|---:|
| read | ✅ all | ✅ all | ✅ assigned | ✅ campaign | ⚠️ academic tasks | ✅ own |
| add | ✅ | ✅ | ✅ own CRM | ✅ campaign | ✅ academic | ❌ |
| modify | ✅ | ✅ | ✅ assigned | ✅ campaign | ✅ assigned | ✅ own |
| assign | ✅ | ✅ | ❌ | ❌ | ⚠️ academic team | ❌ |
| complete | ✅ | ✅ | ✅ assigned | ✅ assigned | ✅ assigned | ✅ own |
| cancel | ✅ | ✅ | ❌ | ⚠️ own campaign | ⚠️ own academic | ❌ |

## 14.6 `crm.sales_contract` Permission Matrix

| Action | principal | operations | sales | finance | academic_director | marketing |
|---|---:|---:|---:|---:|---:|---:|
| read | ✅ | ✅ | ✅ own/assigned | ✅ | ⚠️ read only | ❌ |
| add | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| modify | ✅ | ✅ | ✅ own | ❌ | ❌ | ❌ |
| activate | ✅ | ✅ | ⚠️ configurable | ❌ | ❌ | ❌ |
| terminate | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| refund | ✅ approve | ⚠️ process | ❌ | ✅ process | ❌ | ❌ |
| export | ✅ | ✅ | ⚠️ own | ✅ | ❌ | ❌ |

## 14.7 `crm.sales_payment` Permission Matrix

| Action | principal | operations | finance | sales |
|---|---|---|---|---:|
| read | ✅ | ✅ | ✅ | ⚠️ own contracts |
| add | ✅ | ✅ | ✅ | ⚠️ request only / configurable |
| modify | ✅ | ⚠️ before settlement | ✅ | ❌ |
| void | ✅ approve | ⚠️ process | ✅ process | ❌ |
| export | ✅ | ✅ | ✅ | ❌ |

## 14.8 `crm.communication` Permission Matrix

| Action | principal | operations | sales | marketing | academic_director | teacher |
|---|---:|---:|---:|---:|---:|---:|
| read | ✅ all | ✅ all | ✅ assigned | ✅ campaign | ⚠️ student-related | ❌ |
| add_communication | ✅ | ✅ | ✅ assigned | ✅ campaign | ⚠️ academic note | ❌ |
| send | ✅ | ✅ | ✅ assigned | ✅ campaign | ⚠️ academic only | ⚠️ own class via teacher workspace |
| delete | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| export | ✅ | ✅ | ⚠️ own | ❌ | ❌ | ❌ |

## 14.9 `crm.tag` and `crm.tag_assignment` Permission Matrix

| Action | principal | operations | marketing | sales | academic_director |
|---|---:|---:|---:|---:|---:|
| read | ✅ | ✅ | ✅ | ✅ | ✅ |
| add | ✅ | ✅ | ✅ marketing tags | ❌ | ⚠️ academic tags |
| modify | ✅ | ✅ | ✅ own type | ❌ | ⚠️ own type |
| delete | ✅ | ✅ | ❌ | ❌ | ❌ |
| assign | ✅ all entity types | ✅ all entity types | ✅ prospect / activity | ✅ prospect / student | ⚠️ academic tag on student |

## 14.10 `crm.conversion` Permission Matrix

| Action | principal | operations | sales | marketing | academic_director | finance |
|---|---:|---:|---:|---:|---:|---:|
| read | ✅ all | ✅ all | ✅ assigned | ⚠️ campaign-related | ⚠️ read | ❌ |
| add | ✅ | ✅ | ✅ assigned | ❌ | ❌ | ❌ |
| modify | ✅ | ✅ | ✅ assigned | ❌ | ❌ | ❌ |
| move_stage | ✅ | ✅ | ✅ assigned | ❌ | ❌ | ❌ |
| assign_owner | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| delete | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| archive | ✅ | ✅ | ⚠️ own | ❌ | ❌ | ❌ |

Recommended default:

```text
sales 管理 assigned_to = self 的转化；
principal / operations 看全校转化；
marketing 查看 campaign 相关转化，但不操作；
academic_director 只读查看已转化的学生来源；
finance 无转化权限。
```

## 14.11 `crm.conversion_stage` Permission Matrix

| Action | principal | operations | sales | marketing | it_admin |
|---|---:|---:|---:|---:|---:|
| read | ✅ | ✅ | ✅ | ✅ | ✅ |
| add | ✅ | ✅ | ❌ | ❌ | ✅ |
| modify | ✅ | ✅ | ❌ | ❌ | ✅ |
| delete | ⚠️ restricted | ⚠️ restricted | ❌ | ❌ | ✅ |

Recommended default:

```text
conversion_stage 是配置类数据，主要由 principal / operations / it_admin 管理；
sales / marketing 只能读取阶段配置以执行转化操作；
删除操作限制使用（需确保无转化引用待删除阶段）。
```

## 14.12 `org.guardian_assignment` Permission Matrix

Admin-only — student-scoped guardian link management (via `/api/admin/crm/students/:id/guardians`).

| Action | principal | operations | it_admin | sales |
|---|---|---|---|---:|
| read | ✅ all students | ✅ all students | ✅ all students | ✅ assigned students |
| add | ✅ | ✅ | ✅ | ❌ |
| modify | ✅ | ✅ | ✅ | ❌ |
| revoke | ✅ | ✅ | ✅ | ❌ |
| send_invitation | ✅ | ✅ | ❌ | ❌ |

---

# 15. Academic Workspace Permission Matrix

Academic routes:

```text
/app/academic/resources/students
/app/academic/resources/teachers
/app/academic/resources/rooms
/app/academic/classes
/app/academic/scheduling/attempts
/app/academic/scheduling/confirm
/app/academic/baselines
/app/academic/execution/today
/app/academic/execution/history
/app/academic/execution/makeups
/app/academic/review/study-plan
/app/academic/review/course-changes
/app/academic/review/material-changes
/app/academic/contracts
/app/academic/settings/time-rules
/app/academic/settings/scheduling-strategy
/app/academic/settings/notification-strategy
/app/academic/settings/audit-log
```

## 15.1 Academic High-Level Matrix

| Module | principal | academic_director | operations | finance | teacher |
|---|---:|---:|---:|---:|---:|
| Resources - students | ✅ read | ✅ | ✅ | ⚠️ read | ❌ |
| Resources - teachers | ✅ read | ✅ | ✅ | ❌ | ❌ |
| Resources - rooms | ✅ read | ✅ | ✅ | ❌ | ❌ |
| Classes | ✅ read | ✅ | ✅ support | ❌ | ⚠️ own |
| Scheduling | ✅ read | ✅ | ✅ support | ❌ | ❌ |
| Baselines | ✅ read | ✅ | ⚠️ support | ❌ | ❌ |
| Execution today | ✅ read | ✅ | ✅ | ❌ | ⚠️ own |
| Attendance | ✅ read | ✅ | ✅ | ❌ | ✅ own |
| Makeups | ✅ read | ✅ | ✅ | ❌ | ⚠️ own |
| Review | ✅ read | ✅ | ⚠️ support | ❌ | ❌ |
| Academic contracts | ✅ read | ⚠️ read | ⚠️ support | ✅ | ❌ |
| Academic settings | ✅ approve/read | ✅ manage | ❌ | ❌ | ❌ |

## 15.2 Scheduling / Baseline Permissions

| Action | principal | academic_director | operations | teacher |
|---|---:|---:|---:|---:|
| view schedule attempts | ✅ | ✅ | ✅ | ❌ |
| create scheduling attempt | ❌ | ✅ | ⚠️ support | ❌ |
| confirm schedule | ✅ approve optional | ✅ | ❌ | ❌ |
| create baseline | ❌ | ✅ | ❌ | ❌ |
| update baseline by creating new version | ❌ | ✅ | ❌ | ❌ |
| delete baseline | ❌ | ❌ | ❌ | ❌ |
| view class sessions | ✅ | ✅ | ✅ | ✅ own |
| modify class sessions | ❌ | ✅ | ⚠️ operational only | ❌ |

Reminder:

```text
schedule_baseline 不直接修改 / 删除；
只能创建新版本并翻转 is_current。
```

---

# 16. Teacher Workspace Permission Matrix

Teacher routes:

```text
/app/teacher/home
/app/teacher/messages/inbox
/app/teacher/messages/unread
/app/teacher/messages/:messageId
/app/teacher/courses
/app/teacher/courses/:courseId
/app/teacher/courses/:courseId/records
/app/teacher/courses/:courseId/homework
/app/teacher/courses/:courseId/suggest
/app/teacher/attendance/summary
/app/teacher/attendance/history
/app/teacher/substitution/as-substitute
/app/teacher/substitution/replaced
/app/teacher/terminations
/app/teacher/terminations/:terminationId
```

## 16.1 Teacher Permissions

| Object | Action | Scope | teacher |
|---|---|---|---:|
| `teacher.home` | read | self | ✅ |
| `class` | read | own_classes | ✅ |
| `course` | read | own_courses | ✅ |
| `student` | read | own_students limited | ✅ |
| `attendance` | add / modify | own_classes | ✅ |
| `homework` | add / modify | own_classes | ✅ |
| `course_record` | add / modify | own_classes | ✅ |
| `message` | read / send | own_classes / own_students | ✅ |
| `email` | send | own_classes / own_students | ⚠️ future |
| `delivery_status` | read | own sent messages | ⚠️ future |
| `substitution` | read | own | ✅ |
| `termination_request` | read / comment | own students | ⚠️ configurable |

Teacher cannot:

```text
查看全校学生
查看非自己班级
修改合同
管理线索
创建账号
分配角色
查看销售付款
```

---

# 17. Student Workspace Permission Matrix

Student routes:

```text
/app/student/home
/app/student/messages/inbox
/app/student/messages/unread
/app/student/messages/:messageId
/app/student/courses
/app/student/courses/:courseId
/app/student/courses/:courseId/attendance
/app/student/courses/:courseId/homework
/app/student/courses/:courseId/leave-request
/app/student/courses/:courseId/termination-request
/app/student/makeups
/app/student/makeups/:makeupId
/app/student/status/termination
/app/student/status/resume
/app/student/attendance/summary
/app/student/attendance/history
```

## 17.1 Student Permissions

| Object | Action | Scope | student |
|---|---|---|---:|
| own profile | read | self | ✅ |
| own courses | read | self | ✅ |
| own attendance | read | self | ✅ |
| own homework | read / submit | self | ✅ |
| own messages | read | self | ✅ |
| leave request | add | self | ✅ |
| termination request | add | self | ✅ |
| makeup records | read | self | ✅ |
| payment / contract | read | self / guardian configurable | ⚠️ |
| other students | read | any | ❌ |
| admin data | any | any | ❌ |

---

# 18. Guardian Workspace Permission Matrix

Guardian workspace is now a formal workspace.

Guardian routes:

```text
/app/guardian/home
/app/guardian/messages/inbox
/app/guardian/messages/unread
/app/guardian/messages/:messageId
/app/guardian/students
/app/guardian/students/:studentId
/app/guardian/students/:studentId/courses
/app/guardian/students/:studentId/attendance
/app/guardian/students/:studentId/homework
/app/guardian/students/:studentId/leave-request
/app/guardian/students/:studentId/payments
```

API routes:

```text
/api/guardian/**
/api/guardian/students/**
/api/guardian/messages/**
/api/guardian/attendance/**
/api/guardian/homework/**
/api/guardian/payments/**
```

## 18.1 Guardian Identity Source

Guardian 身份来源：

```text
org.guardian_assignment.guardian_person_id = current_user.person_id
```

数据范围：

```sql
SELECT ga.student_person_id
FROM org.guardian_assignment ga
WHERE ga.guardian_person_id = :currentPersonId
  AND (ga.valid_to IS NULL OR ga.valid_to >= CURRENT_DATE);
```

## 18.2 Guardian Permissions

| Object | Action | Scope | guardian |
|---|---|---|---:|
| linked student profile | read | linked_student | ✅ |
| linked student courses | read | linked_student | ✅ |
| linked student attendance | read | linked_student | ✅ |
| linked student homework | read | linked_student | ✅ |
| linked student messages | read / reply | linked_student | ✅ |
| payment / contract summary | read | linked_student | ⚠️ configurable |
| leave request | add | linked_student | ✅ |
| termination request | add | linked_student | ⚠️ configurable |
| other students | read | any | ❌ |
| admin workspace | access | any | ❌ |

## 18.3 Guardian Workspace Binding

推荐绑定：

```text
auth.user_workspace_binding.workspace_type = 'guardian'
auth.user_workspace_binding.related_resource_id = NULL
```

理由：

```text
一个 guardian 可能关联多个学生。
数据范围应通过 org.guardian_assignment 解析，而不是通过 related_resource_id 绑定单个学生。
```

---

# 19. Email / Notification / Messages Permission Matrix

## 19.1 Domain Definitions

```text
Messages
= user-facing inbox and message detail

Email
= SMTP accounts, templates, send logs, delivery status

Notifications
= in-app alerts, reminders, broadcasts
```

这三类能力不能混淆。

## 19.2 Email Infrastructure Permissions

| Action | principal | operations | marketing | sales | finance | academic_director | teacher | it_admin |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| send email | ✅ | ✅ | ✅ campaign | ✅ assigned | ❌ | ⚠️ academic | ⚠️ own class future | ✅ |
| view email logs | ✅ all | ✅ all | ✅ campaign | ✅ own | ⚠️ billing only | ⚠️ academic | ⚠️ own | ✅ all |
| manage templates | ❌ | ✅ | ✅ marketing | ❌ | ❌ | ⚠️ academic templates | ❌ | ✅ |
| manage email accounts | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| view delivery status | ❌ | ✅ | ✅ campaign | ✅ own | ❌ | ⚠️ academic | ⚠️ own | ✅ |
| retry failed email | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

## 19.3 Notification Permissions

| Action | principal | operations | marketing | sales | academic_director | teacher | student | guardian | it_admin |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| notification: read | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| notification: modify | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| notification.broadcast: send | ✅ | ✅ | ❌ | ❌ | ⚠️ academic only | ❌ | ❌ | ❌ | ✅ |

## 19.4 Personal Messages Permissions

| Action | principal | operations | sales | academic_director | teacher | student | guardian |
|---|---:|---:|---:|---:|---:|---:|---:|
| view own inbox | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| send internal message | ✅ | ✅ | ✅ | ✅ | ✅ own class | ⚠️ to teacher/admin | ⚠️ to school/teacher |
| view unread | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| archive own message | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

---

# 20. Audit Permission Matrix

| Audit Object | principal | operations | finance | it_admin | saas_operator | system_maintainer |
|---|---:|---:|---:|---:|---:|---:|
| view school audit logs | ✅ | ❌ | ⚠️ finance only | ✅ | ❌ | ⚠️ break-glass |
| view user login logs | ✅ school | ❌ | ❌ | ✅ school | ✅ platform | ✅ platform |
| view role assignment logs | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| view payment logs | ✅ | ❌ | ✅ | ✅ | ❌ | ⚠️ technical only |
| view platform audit logs | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| export audit logs | ⚠️ | ❌ | ⚠️ finance only | ✅ | ✅ | ✅ |

## 20.1 Audit Principles

Sensitive operations must create audit logs:

```text
role assignment
role revocation
user creation
password reset
account deactivation
prospect conversion
contract activation
payment modification
refund / void
platform user creation
platform role assignment
system_maintainer break-glass access
```

---

# 21. Recommended Permission Object Naming and Session Context

## 21.1 Permission Object Naming

### Org Objects

```text
org.person
org.institution
org.campus
org.institution_member
org.institution_role_assignment
org.guardian_assignment
auth.user_account
auth.user_account.principal
auth.platform_role_assignment
org.institution_role_assignment.principal
```

### CRM Objects

```text
crm.prospect
crm.student_profile
crm.marketing_activity
crm.marketing_activity_participant
crm.task
crm.conversion
crm.conversion_stage
crm.sales_contract
crm.sales_payment
crm.renewal_reminder
crm.churn_warning
crm.communication
crm.tag
crm.tag_assignment
```

### Academic Objects

```text
academic.course
academic.class
academic.class_member
academic.schedule_attempt
academic.schedule_baseline
execution.class_session
execution.attendance
execution.makeup
contract.contract
```

### Communication Objects

```text
message
email.send
email.log
email.template
email.account
email.delivery
notification
notification.broadcast
email.initial_password
```

### Guardian Objects

```text
linked_student.profile
linked_student.course
linked_student.attendance
linked_student.homework
linked_student.message
linked_student.payment_summary
linked_student.leave_request
```

## 21.2 Principal Session Context Example

```json
{
  "current_workspace": "admin",
  "available_workspaces": ["admin", "academic"],
  "account_scope": "institution",
  "institution_id": "uuid",
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
  },
  "data_scope": {
    "institution_id": "uuid",
    "campus_ids": [],
    "assigned_only": false,
    "own_classes_only": false,
    "linked_student_person_ids": []
  }
}
```

## 21.3 Guardian Session Context Example

```json
{
  "current_workspace": "guardian",
  "available_workspaces": ["guardian"],
  "account_scope": "institution",
  "institution_id": "uuid",
  "person_id": "guardian-person-uuid",
  "user_account_id": "uuid",
  "role_codes": [],
  "platform_role_codes": [],
  "member_type": [],
  "relationship_codes": ["mother"],
  "permissions_by_workspace": {
    "guardian": {
      "linked_student.profile": ["read"],
      "linked_student.course": ["read"],
      "linked_student.attendance": ["read"],
      "linked_student.homework": ["read"],
      "linked_student.message": ["read", "add"],
      "linked_student.leave_request": ["add"],
      "linked_student.payment_summary": ["read"]
    }
  },
  "data_scope": {
    "institution_id": "uuid",
    "linked_student_person_ids": [
      "student-person-uuid-1",
      "student-person-uuid-2"
    ]
  }
}
```

## 21.4 Platform User Session Context Example

```json
{
  "current_workspace": null,
  "available_workspaces": [],
  "account_scope": "platform",
  "institution_id": null,
  "person_id": "platform-person-uuid",
  "user_account_id": "uuid",
  "role_codes": [],
  "platform_role_codes": ["saas_operator"],
  "permissions_by_workspace": {},
  "platform_permissions": {
    "org.education_institution": ["read", "add", "modify", "deactivate"],
    "auth.user_account.principal": ["add"],
    "org.institution_role_assignment.principal": ["assign"],
    "email.initial_password": ["send"]
  },
  "data_scope": {
    "scope": "platform"
  }
}
```

---

# 22. Implementation Rules, Open Decisions, and QA Gates

## 22.1 Frontend Rules

Frontend uses permissions for:

```text
show / hide menu items
enable / disable buttons
route guards
field-level visibility
bulk operation visibility
workspace switcher options
```

Frontend must not be the only enforcement layer.

## 22.2 Backend Rules

Backend must enforce permissions using:

```text
Spring Security
@PreAuthorize
service-level scope checks
repository-level institution_id filters
```

All sensitive operations must validate:

```text
user has workspace access
+
user has action permission
+
user belongs to institution if institution-scoped
+
target object belongs to same institution
+
scope rule allows operation
```

## 22.3 Database Rules

Database should support permission enforcement by having relevant fields:

```text
institution_id
created_by
assigned_to
person_id
student_person_id
guardian_person_id
entity_type
entity_id
account_scope
```

## 22.4 No Cross-Institution Leakage

Every institution-scoped query must include:

```text
institution_id = current_user.institution_id
```

or an equivalent join-based institution check.

## 22.5 Finalized Decisions

**v1.5 Default** — Configurability is noted but not required in the initial release.

### 22.5.1 Can principal create another principal?

**Decision: No.** Only SaaSOperator or specially authorized platform/system flow can create another principal. Principal cannot delegate this power.

Configurable: Future.

### 22.5.2 Can operations assign roles?

**Decision: No by default.** operations can view role assignments. Role assignment requires principal action. If delegated_by_principal is configured in a future release, operations may assign limited operational roles.

Configurable: Future.

### 22.5.3 Can sales convert prospect to student?

**Decision: Yes for assigned prospects.** Default mode: sales can convert assigned prospects to students. Controlled mode (sales submits conversion request, operations/principal approves) is a future configuration option.

Configurable: Future.

### 22.5.4 Can finance modify payments?

**Decision: Yes before settlement.** finance can add/modify payment records before settlement. Void/refund requires principal approval.

Configurable: Future.

### 22.5.5 Can teacher send emails directly?

**Decision: No for v1.5.** Reserve permissions only. Enable `teacher_send_email` for own classes/students in a later phase.

Configurable: Future.

### 22.5.6 Should platform users enter Classing SPA?

**Decision: No by default.** Platform users use SaaSOperator. If `system_maintainer` needs technical access to Classing SPA, define a separate technical entry with strong audit. Not a v1.5 concern.

## 22.6 DDL QA Gates

Required grep checks after DDL revision:

```text
workspace_type_enum contains: academic, teacher, student, guardian, admin
workspace_type_enum does not contain: academic_admin, management

org_institution_role_enum contains: principal, academic_director, people_admin, finance, it_admin, marketing, operations, sales
org_institution_role_enum does not contain: system_maintainer

platform_role_enum contains: saas_operator, system_maintainer

auth.user_account supports account_scope = institution/platform

auth.platform_role_assignment exists

auth.user_workspace_binding comments include guardian
```

### CRM Table QA Gates

```text
Every crm.* business table MUST have institution_id NOT NULL, either directly on the table
or (for polymorphic join tables) via a composite FK to the owning table.

Tables that MUST have direct institution_id NOT NULL:
  crm.prospect
  crm.student_profile (composite PK with person_id)
  crm.marketing_activity
  crm.marketing_activity_participant (formerly activity_participant)
  crm.task
  crm.conversion
  crm.conversion_stage
  crm.sales_contract
  crm.sales_payment
  crm.renewal_reminder
  crm.churn_warning
  crm.communication
  crm.tag
  crm.tag_assignment

Tables that MUST NOT have globally UNIQUE constraints on business identifiers:
  crm.sales_contract: UNIQUE(institution_id, contract_no) — NOT UNIQUE(contract_no) alone

email.email_log MUST have (created directly in V2_2's CREATE TABLE, after email.email_account):
  email_account_id UUID REFERENCES email.email_account(id)  (nullable for default SMTP)
  idx_email_log_email_account ON email.email_log(email_account_id)
email.email_log MUST NOT be in V2_1 — V2_1 only has email_template + email_rate_limit; email_log moved to V2_2
  so the FK to email.email_account can be defined in CREATE TABLE without ALTER.

Polymorphic tables MUST have institution_id for direct multi-tenant filtering:
  crm.communication:  INDEX ON (institution_id, entity_type, entity_id)
  crm.tag_assignment:  INDEX ON (institution_id, entity_type, entity_id)
```

## 22.7 Route QA Gates

Required frontend routes:

```text
/app/academic/*
/app/teacher/*
/app/student/*
/app/guardian/*
/app/admin/*
```

Forbidden old conceptual mapping:

```text
academic_admin
management
```

## 22.8 Final Principle

The system follows this hierarchy:

```text
SaaSOperator
  = platform-level school initialization

Platform Roles
  = saas_operator / system_maintainer

Principal
  = school-level administrator

Operations
  = school-level operational executor

Sales
  = prospect and contract conversion

Marketing
  = campaign and lead generation

Academic Director
  = academic planning and execution

Teacher
  = own class teaching workflow

Student
  = self-service learning workflow

Guardian
  = linked-student family access
```

The most important boundaries are:

```text
SaaSOperator creates the school and first principal.
Principal creates and manages the school's internal organization.
Sales / Marketing / Operations manage CRM leads under principal's institution.
Guardian accesses only linked students.
All prospect records live in crm.prospect.
Platform roles do not live in org.institution_role_assignment.
```
