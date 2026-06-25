# CRM Integration Architecture v1.5 — Final Requirements Frozen

**Date**: 2026-06-24  
**Status**: v1.5 Final Requirements Frozen — implementation handoff baseline  
**Scope**: Wonders Academy 官网 → 统一业务系统（React SPA）→ Spring Boot API 架构下 CRM 模块整合方案  
**Participating Projects**: WondersAcademyWebsite（静态站）、ClassingSystem Backend（Spring Boot + Kotlin）、CRM Frontend（React）、SaaSOperator（独立管理工具）

---

## 1. Overview

### 1.1 Background

Wonders Academy 技术体系由四个项目组成：

| Project | Directory | Tech Stack | Responsibility |
|---------|-----------|------------|----------------|
| **ClassingSystem Backend** | `D:\dev\LLYProjects\ClassingSystem\Backend\BE_Classing` | Spring Boot 3.5 + Kotlin + JPA + Flyway + PostgreSQL | 统一后端：Auth、Org、CRM、Email、Messaging、Academic 等所有业务模块 |
| **ClassingSystem DB** | `D:\dev\LLYProjects\ClassingSystem\DB` | PostgreSQL + Flyway | 数据库脚本与 docker-compose |
| **SaaSOperator** | `D:\dev\LLYProjects\ClassingSystem\SaaSOperator` | 静态 HTML/JS + 独立登录 | 平台运营管理工具，**不纳入 `/app/admin/*` 路由体系** |
| **CRM Frontend** | `D:\dev\CRM` | React + Vite + Ant Design + Zustand + TanStack Query | 客户关系管理 SPA，**需迁移到 `/app/admin/*` 路由体系** |
| **Wonders Academy Website** | `D:\dev\LLYProjects\WondersAcademyWebsite` | Vanilla HTML/CSS/JS | 官网展示、招生转化、Legacy 线索管理 |

已有架构文档：

| Document | Version | Key Content |
|----------|---------|-------------|
| `Frontend_Architecture_v1.5_Final_Requirements_Frozen.md` | Final — implementation handoff baseline | React SPA + Ant Design + Zustand + TanStack Query |
| `Wonders_Website_Architecture_Evolution_v1.5_Final_Requirements_Frozen.md` | Final — implementation handoff baseline | 官网/SPA/Legacy 三层架构，Workspace 路由 |
| `Workspace_Route_Map_v1.5_Final_Requirements_Frozen.md` | Final — implementation handoff baseline | `/app/academic/*` `/app/teacher/*` `/app/student/*` `/app/admin/*` 路由分配 |

### 1.2 Goals

将 CRM 前端 React SPA 整合进 ClassingSystem 统一架构：

1. **Database Unification**: CRM 数据表并入 ClassingSystem PostgreSQL 实例，与 `org.person` / `auth.user_account` / `crm.prospect` 等表共享同一数据库
2. **Backend Unification**: CRM 业务逻辑迁移至 Spring Boot + Kotlin，统一 API 体系
3. **Frontend Unification**: CRM React 前端适配为 `/app/admin/crm/*` 下的 SPA 模块
4. **Auth Unification**: 复用 `auth.user_account` + JWT 刷新令牌体系
5. **Mobile Ready**: 基于 React 生态，后续通过 React Native 覆盖手机/平板

### 1.3 Final Architecture

```
wondersacademy.pl
│
├─ /                              Static Website（unchanged）
├─ /admin-login.html             Legacy（to be deprecated）
├─ /admin-prospect.html          Legacy（to be deprecated）
│
├─ /app/                         React SPA Unified Business System
│   ├─ /login                    Unified Login Entry
│   ├─ /academic/*               Academic Workspace
│   ├─ /teacher/*                Teacher Workspace
│   ├─ /student/*                Student Workspace
│   ├─ /guardian/*               Guardian Workspace
│   └─ /admin/*
│       ├─ /messages/*           Personal inbox
│       ├─ /org/*                Institution / person / member / room management
│       ├─ /crm/
│       │   ├─ /prospects        Lead Pool（admin manual entry）
│       │   ├─ /students         Enrolled Students（← current CRM student list）
│       │   ├─ /marketing-activities       Marketing Activity Management
│       │   ├─ /tasks            Task Board
│       │   ├─ /conversions      Conversion Pipeline
│       │   ├─ /conversion-stages  Conversion Stage Config
│       │   ├─ /sales-contracts  Contracts & Renewals
│       │   ├─ /renewals         Renewal Reminders
│       │   ├─ /communications   Communication History
│       │   ├─ /tags             Tag Management
│       │   └─ /churn-warnings   Churn Warnings
│       ├─ /email/
│       │   ├─ /accounts         SMTP Accounts
│       │   ├─ /templates        Email Templates
│       │   ├─ /logs             Send History（email.email_log）
│       │   └─ /delivery-status  Per-recipient status（email.email_delivery）
│       ├─ /notifications/       In-app notifications / broadcast（messaging.notification）
│       └─ /audit                Audit Log
│
└─ /api/                         Spring Boot Unified API
    ├─ /api/session/context         # Workspace + role + permissions
    ├─ /api/public/**
    ├─ /api/auth/**
    ├─ /api/admin/org/**
    ├─ /api/admin/crm/**
    │   ├─ /prospects/**
    │   ├─ /students/**
    │   ├─ /students/:id/guardians
    │   ├─ /marketing-activities/**
    │   ├─ /tasks/**
    │   ├─ /conversions/**
    │   ├─ /conversion-stages/**
    │   ├─ /sales-contracts/**
    │   ├─ /renewals/**
    │   ├─ /churn-warnings/**
    │   ├─ /communications/**
    │   ├─ /tags/**
    │   └─ /tag-assignments/**
    ├─ /api/admin/email/**
    ├─ /api/admin/notifications/**
    ├─ /api/academic/**
    ├─ /api/teacher/**
    ├─ /api/student/**
    └─ /api/guardian/**
```

---

## 2. Current CRM Details

### 2.1 Database Models（Prisma Schema）

本节描述当前 CRM 源系统（NestJS + Prisma）现状，不代表目标 ClassingSystem 集成后的最终表名与枚举值。
当前 CRM 使用 16 个数据模型，全部定义在 `apps/server/prisma/schema.prisma`：

| Model | Description | Key Fields |
|-------|-------------|------------|
| **User** | System User | name, email, passwordHash, role(sales) |
| **Student** | Student Profile | firstName, lastName, gender, age, dateOfBirth, grade, gradeYear, gradeMonth, daySchool, daySchoolGrade, daySchoolYear, daySchoolMonth, campus, phone, email, wechat, address, nationality, religion, taboos, source, status(lead→archived), notes |
| **Parent** | Parent Info | firstName, lastName, age, relationship, phone, email, wechat, address, nationality, religion, isPrimary, occupation, influence |
| **StudentParent** | Student-Parent Join | studentId + parentId（composite PK） |
| **Tag** | Tag Definition | name, color, type(student/activity/contract) |
| **StudentTag** | Student-Tag Join | studentId + tagId（composite PK） |
| **Activity** | Activity | title, type(trial_class/open_day/lecture/event), startDate, endDate, timezone, location, transportation, organization, capacity, assignedTo, status(draft→cancelled) |
| **ActivityParticipant** | Activity Participation | activityId + studentId, status(registered/checked_in/cancelled) |
| **ConversionStage** | Conversion Stage | name, order, color, probability |
| **Conversion** | Sales Opportunity | studentId + stageId, amount, probability, expectedCloseDate, assignedTo |
| **Contract** | Contract | studentId, contractNo, type, items(Json), totalAmount, paidAmount, remainingSessions, startDate, endDate, status(active→refunded) |
| **Payment** | Payment | contractId, amount, method(cash/transfer/wechat/alipay/pos), receiptNo, paidAt, installmentNo |
| **RenewalReminder** | Renewal Reminder | contractId + studentId, triggerAt, status(pending→cancelled) |
| **ChurnWarning** | Churn Warning | studentId, riskLevel(low/medium/high), reason, autoCreated, resolvedAt |
| **Communication** | Communication Record | type(email/call/meeting/sms), subject, content, direction(inbound/outbound), studentId, parentId |
| **EmailAccount** | SMTP Account | name, email, smtpHost, smtpPort, smtpUser, smtpPassEnc(AES), isDefault |
| **Email** | Email Record | emailAccountId, fromAddr, toAddrs[], ccAddrs[], subject, bodyHtml, bodyText, attachments(Json), status(draft→failed) |
| **Task** | Task | title, description, dueDate, priority(low→urgent), status(pending→cancelled), assignedTo, relatedToType, relatedToId, reminderAt |
| **Attachment** | Attachment | filename, originalName, mimeType, size, path, relatedToType, relatedToId |
| **AuditLog** | Audit Log | userId, action, entityType, entityId, oldValues(Json), newValues(Json), ipAddress |

### 2.2 Backend API Endpoints（NestJS）

约 80+ 个端点，按模块分组：

| Module | Prefix | Count | Typical Endpoints |
|--------|--------|-------|-------------------|
| Auth | `/api/auth` | 2 | POST login, GET me |
| Students | `/api/students` | 5 | CRUD + list with filter |
| Parents | `/api/parents` | 5 | CRUD + byStudent |
| Activities | `/api/activities` | 8 | CRUD + register/checkin/createTask |
| Conversions | `/api/conversions` | 7 | CRUD + stages + move |
| Contracts | `/api/contracts` | 6 | CRUD + payments |
| Renewals | `/api/renewals` | 6 | CRUD + resolve/dismiss |
| Warnings | `/api/warnings` | 6 | CRUD + resolve |
| Communications | `/api/communications` | 6 | read + add_communication + byStudent |
| Tasks | `/api/tasks` | 5 | CRUD + byUser |
| Tags | `/api/tags` | 6 | CRUD + student assign/unassign/list |
| EmailAccounts | `/api/email-accounts` | 5 | CRUD |
| Emails | `/api/emails` | 6 | CRUD + send |

### 2.3 Frontend Pages

| Page | Route | Features |
|------|-------|----------|
| StudentList | `/students` | Table + search + tag filter + create/edit/delete |
| StudentDetail | `/students/:id` | Detail card + related info Tabs（contracts/activities/communications） |
| ActivityList | `/activities` | Activity list + create/edit/delete |
| ActivityDetail | `/activities/:id` | Detail + participant management |
| ContractList | `/contracts` | Contract list + create/edit/delete |
| ContractDetail | `/contracts/:id` | Contract detail + payment records |
| RenewalList | `/renewals` | Renewal reminder list + create/edit/delete |
| WarningBoard | `/warnings` | Churn warning board + create/edit/delete |
| TaskBoard | `/tasks` | Task board + create/edit |
| ConversionPipeline | `/conversions` | Conversion pipeline + edit |
| TagsManagement | sidebar modal | Tag CRUD |

### 2.4 Tag System

- Tables: `tags`（id, name, color, type）+ `student_tags`（studentId, tagId）
- Sidebar: dynamic tag submenu items, click → `/students?tag=xxx`
- StudentList: tag badges per row, filter by tag
- Create Student: tag selection in create modal

---

## 3. Data Model Integration

### 3.0 Canonical Identity Model

The canonical student identity across the system is `org.person.id`.

| Concept | Canonical ID | Owner | Notes |
|---------|--------------|-------|-------|
| Lead / Prospect | `crm.prospect.id` | CRM | Lightweight pre-conversion lead record |
| Student | `org.person.id` | org / ClassingSystem | Formal person identity used by academic, contract, attendance, and resource domains |
| CRM Student Profile | `crm.student_profile.person_id` | CRM | Extension profile only; not a separate student identity |
| Guardian / Parent | `org.person.id` | org | Linked to student by `org.guardian_assignment` |

Principle:

```text
crm.student_profile is not the student master table.
The master student identity is org.person.id.
```

---

### 3.1 Core Data Flow

```
Website Form / Admin Manual Entry
       │
       ▼
┌─────────────────────────────────────┐
│     crm.prospect（Lead Pool）   │
│                                     │
│  status: NEW → CONTACTED → QUALIFIED│
│  Source Tracking: source_page,       │
│                  courses, IP, UA     │
│  Contact: legal_full_name, email,    │
│           phone, guardian_name       │
│  ⭐Can associate Conversion(deal)    │
│  ⭐Can associate Communication       │
│     (via crm.communication table)     │
│  institution_id resolved server-side  │
└──────────┬──────────────────────────┘
           │ Click "Convert to Student"
           ▼
┌─────────────────────────────────────┐
│  Conversion Dialog（one-time fill）  │
│                                     │
│  1. Name Confirmation（firstName/   │
│     lastName from legal_full_name,  │
│     manually adjustable）            │
│  2. Person Required Fields（gender,  │
│     DOB, address, id_document, etc） │
│  3. CRM Extension Fields（taboos,    │
│     grade, daySchool, campus, etc）  │
└──────────┬──────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│  org.person（Student Formal Record）          │
│  + org.institution_member（member_type=student）│
│  + core.resource（Scheduling）                 │
│  + crm.student_profile（CRM Ext）              │
│  + optional: guardian org.person              │
│  + optional: org.guardian_assignment          │
│  + optional: guardian auth.user_account       │
│                                              │
│  student_profile.status: enrolled             │
│  → can change to lost / archived              │
│                                              │
│  crm.prospect.person_id = person.id           │
│  crm.prospect.status = CONVERTED              │
└──────────────────────────────────────────────┘
```

> **institution_id security rule for public form submission**: Anonymous public website lead submission MUST resolve `institution_id` server-side. For `/api/public/prospects`: (1) request is unauthenticated; (2) `institution_id` resolved from `public_form_config.institution_code`, domain, or default `institution_code = 'WONDERS_ACADEMY'`; (3) NEVER trust client-side hidden fields for institution_id — validate server-side to prevent cross-institution injection. See `Wonders_Website_Architecture_Evolution` §Public Form institution_id Resolution for details.

### 3.2 Field Mapping Details

#### Student → `org.person` + `crm.student_profile`

| CRM Student Field | Target Table | Target Field | Mapping Rule |
|-------------------|--------------|--------------|--------------|
| `id` | `person.id` | UUID | Auto-generated |
| `firstName` | `person.given_name` | text NULL | Split from legal_full_name, confirm in conversion dialog |
| `lastName` | `person.family_name` | text NULL | Split from legal_full_name, confirm in conversion dialog |
| — | `person.legal_full_name` | text NOT NULL | Inherit from crm.prospect.legal_full_name |
| — | `person.preferred_name` | text NULL | Optional nickname |
| `gender` | `person.gender` | enum NOT NULL | ⚠️ CRM optional → conversion dialog required |
| `dateOfBirth` | `person.date_of_birth` | date NOT NULL | ⚠️ CRM optional → conversion dialog required |
| `age` | — | — | Deprecated; compute from DOB |
| `phone` | `person.primary_phone` | text NOT NULL | ⚠️ CRM optional → conversion dialog required |
| `email` | `person.email` | text NULL | ✅ Direct mapping |
| `wechat` | `person.wechat_id` | text NULL | ✅ Direct mapping |
| `address` | `person.address` | text NOT NULL | ⚠️ CRM optional → conversion dialog required |
| `nationality` | `person.nationality` | text NOT NULL | Default "Poland", adjustable |
| `religion` | `person.religion` | text NOT NULL | Default "None", adjustable |
| — | `person.postal_code` | text NOT NULL | ⚠️ New; conversion dialog required |
| — | `person.country_code` | text NOT NULL | Default "PL" |
| — | `person.id_document_type` | text NOT NULL | ⚠️ New; conversion dialog required |
| — | `person.id_document_number` | text NOT NULL | ⚠️ New; conversion dialog required |
| `grade` | `student_profile.grade` | VARCHAR | CRM grade level |
| `gradeYear` | `student_profile.grade_year` | INT | |
| `gradeMonth` | `student_profile.grade_month` | INT | |
| `daySchool` | `student_profile.day_school` | VARCHAR | |
| `daySchoolGrade` | `student_profile.day_school_grade` | VARCHAR | |
| `daySchoolYear` | `student_profile.day_school_year` | INT | |
| `daySchoolMonth` | `student_profile.day_school_month` | INT | |
| `campus` | `student_profile.campus` | VARCHAR | |
| `source` | `student_profile.source` | VARCHAR | Inherited from crm.prospect source |
| `status` | `student_profile.status` | VARCHAR | enrolled/lost/archived |
| `taboos` | `student_profile.taboos` | TEXT | |
| `notes` | `student_profile.notes` | TEXT | |
| `tags` | `crm.tag_assignment` | — | Many-to-many via join table |

#### Parent → `org.person` + `org.guardian_assignment`

| CRM Parent Field | Target Table | Target Field | Mapping Rule |
|------------------|--------------|--------------|--------------|
| `id` | `person.id` | UUID | Auto-generated |
| `firstName` | `person.given_name` | text NULL | |
| `lastName` | `person.family_name` | text NULL | |
| `relationship` | `guardian_assignment.relationship_type` | enum | father/mother/guardian |
| `isPrimary` | `guardian_assignment.is_primary_guardian` | boolean | |
| `phone` | `person.primary_phone` | text NOT NULL | |
| `email` | `person.email` | text NULL | |
| `wechat` | `person.wechat_id` | text NULL | |
| `address` | `person.address` | text NOT NULL | |
| `nationality` | `person.nationality` | text NOT NULL | |
| `religion` | `person.religion` | text NOT NULL | |
| `occupation` | `person` N/A | — | Can extend person or put in notes |
| `influence` | — | — | CRM-specific, can put in notes |

#### User → `auth.user_account`

| CRM User Field | Target Table | Target Field | Mapping Rule |
|----------------|--------------|--------------|--------------|
| `id` | `user_account.id` | UUID | Import existing data |
| `name` | `user_account.login_name` | text NOT NULL | |
| `email` | — | — | Available from `person.email` |
| `passwordHash` | `user_password.password_hash` | text | Import into password history |
| `role` | `org.institution_role_assignment.role_code` | enum.org_institution_role_enum | Map CRM `sales` role → add `'sales'` to `V0_2__enums.sql` `CREATE TYPE` list, then assign via `org.institution_role_assignment` |

### 3.3 New Table Definitions

以下表通过两个 Flyway 迁移文件创建（email extend + messaging 在前，crm 在后，以满足 FK 依赖）。
注意：`email.email_template`、`email.email_rate_limit` 已在 `V2_1__email_schema_and_templates.sql` 中存在。
表 `email.email_account`、`email.email_log`（含 `email_account_id`）、`email.email_delivery` 和 `messaging` 站内通知表则在 `V2_2__extend_email_and_messaging.sql` 中创建。
> **设计说明：** `email.email_log` 从 V2_1 移至 V2_2，使其 `email_account_id` 字段能在 `CREATE TABLE` 中直接引用 `email.email_account`（V2_2 前部创建），无需 `ALTER TABLE`。在开发/预生产阶段可以直接修改迁移文件。

#### V2_2__extend_email_and_messaging.sql

```sql
-- ============================================================
-- Part 1: Extend existing email schema with SMTP account + log + delivery
--         email.email_template / email.email_rate_limit
--         already exist in V2_1__email_schema_and_templates.sql
--         email.email_log was moved from V2_1 to V2_2 so that
--         email_account_id can be defined in CREATE TABLE
--         (references email.email_account created above).
-- ============================================================
CREATE TABLE email.email_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    email           VARCHAR(200) NOT NULL,
    smtp_host       VARCHAR(200) NOT NULL,
    smtp_port       INT DEFAULT 587,
    smtp_user       VARCHAR(200) NOT NULL,
    smtp_pass_enc   TEXT NOT NULL,
    smtp_secure     BOOLEAN DEFAULT false,
    is_default      BOOLEAN DEFAULT false,
    is_active       BOOLEAN DEFAULT true,
    institution_id  UUID REFERENCES org.education_institution(id),  -- NULL = platform account; NOT NULL = school account
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Default account rules:
-- 1. At most 1 account per institution may have is_default = true (enforced by partial unique index)
-- 2. A platform-level account (institution_id IS NULL) with is_default = true serves as the
--    platform's default for system-generated emails (e.g., password reset, invitation)
-- 3. A school-level account (institution_id IS NOT NULL) with is_default = true is used for
--    all outbound emails originating from that institution
-- 4. If no default exists for an institution, email sending MUST return a config error
--    (do NOT fall back to platform or another institution's default)
-- Constraint: ensure at most one default per institution (school-level)
CREATE UNIQUE INDEX uq_email_account_default_per_institution
    ON email.email_account (institution_id)
    WHERE is_default = true AND institution_id IS NOT NULL;

-- Constraint: ensure at most one platform-level default (institution_id IS NULL)
CREATE UNIQUE INDEX uq_email_account_default_platform
    ON email.email_account ((1))
    WHERE is_default = true AND institution_id IS NULL;

CREATE TABLE email.email_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_code   VARCHAR(50) NOT NULL,
    recipient_type  VARCHAR(50),
    recipient_id    UUID,
    recipient_email VARCHAR(255) NOT NULL,
    subject         TEXT NOT NULL,
    body            TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    retry_count     INT DEFAULT 0,
    max_retries     INT DEFAULT 5,
    created_at      TIMESTAMPTZ DEFAULT now(),
    sent_at         TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,
    last_retry_at   TIMESTAMPTZ,
    error_message   TEXT,
    sent_by         UUID,
    source_system   VARCHAR(50),
    institution_id  UUID,
    email_account_id UUID REFERENCES email.email_account(id),
    business_type   VARCHAR(50)
);
CREATE INDEX idx_email_log_recipient     ON email.email_log(recipient_id, created_at DESC);
CREATE INDEX idx_email_log_status        ON email.email_log(status, retry_count);
CREATE INDEX idx_email_log_created_at    ON email.email_log(created_at);
CREATE INDEX idx_email_log_template      ON email.email_log(template_code);
CREATE INDEX idx_email_log_institution   ON email.email_log(institution_id, recipient_id, template_code);
CREATE INDEX idx_email_log_email_account ON email.email_log(email_account_id);

CREATE TABLE email.email_delivery (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_log_id    UUID NOT NULL REFERENCES email.email_log(id) ON DELETE CASCADE,
    recipient_email VARCHAR(200) NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending',
    status_at       TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- ============================================================
-- Part 2: In-app notification schema（站内通知 / broadcast）
--         email infra lives in email schema, NOT here
--
-- Product semantics:
--   message    = User-to-user conversations, reply-capable（personal inbox, /app/admin/messages/*）
--                Separate from notification; belongs in a future messaging schema extension.
--   notification = System-triggered event alerts（task_assigned / renewal_reminder / churn_alert）
--                Stored in messaging.notification; each recipient gets a notification_recipient row.
--   broadcast  = Institution/role-level announcement, sent by principal / operations / it_admin.
--                For v1.5, stored as messaging.notification(type = 'broadcast') with
--                multiple notification_recipient rows. May be upgraded to a dedicated
--                messaging.broadcast_message model when rich-text scheduling/targeting is needed.
-- ============================================================
CREATE SCHEMA IF NOT EXISTS messaging;

CREATE TABLE messaging.notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES org.education_institution(id),
    type            VARCHAR(50) NOT NULL,          -- task_assigned/renewal_reminder/churn_alert/broadcast
    title           VARCHAR(200) NOT NULL,
    body            TEXT,
    link            VARCHAR(500),                  -- deep link to relevant page
    created_by      UUID REFERENCES auth.user_account(id),
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE messaging.notification_recipient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL REFERENCES messaging.notification(id) ON DELETE CASCADE,
    person_id       UUID NOT NULL REFERENCES org.person(id) ON DELETE CASCADE,
    is_read         BOOLEAN DEFAULT false,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE(notification_id, person_id)
);
```

#### V2_3__crm_tables.sql

```sql
CREATE SCHEMA IF NOT EXISTS crm;

-- CRM Prospect（canonical lead pool — replaces old prospect.prospect）
-- Absorbs website form fields + CRM sales pipeline
-- Fields originate from current prospect.prospect（pre-production working code）
CREATE TABLE crm.prospect (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id              UUID NOT NULL REFERENCES org.education_institution(id),

    -- Student identity（from website form or manual entry）
    legal_full_name             TEXT NOT NULL,          -- prospect.prospect.name
    preferred_name              TEXT,
    gender                      VARCHAR(10),
    date_of_birth               DATE,
    nationality                 TEXT,
    religion                    TEXT,

    -- Student contact
    primary_phone               TEXT NOT NULL,
    email                       TEXT,
    wechat_id                   TEXT,
    address                     TEXT,

    -- Student school info
    grade_of_next_academic_year TEXT,
    day_school                  TEXT,
    day_school_grade            TEXT,

    -- Guardian info (primary guardian embedded in prospect)
    guardian_name               TEXT,
    guardian_gender             VARCHAR(10),
    guardian_date_of_birth      DATE,
    guardian_nationality        TEXT,
    guardian_phone              TEXT,
    guardian_email              TEXT,
    guardian_wechat             TEXT,
    guardian_address            TEXT,
    guardian_relationship_type  VARCHAR(30),            -- FATHER/MOTHER/LEGAL_GUARDIAN/...

    -- Website form context
    source                      TEXT NOT NULL,          -- homepage/enrollment-2026/course-chinese
    source_courses              TEXT,                   -- comma-separated course list
    source_language             TEXT,                   -- zh/en/pl
    source_url                  TEXT,
    source_channel              TEXT,                   -- website/referral/walk-in/phone
    user_agent                  TEXT,
    ip_address                  TEXT,

    -- CRM sales fields
    notes                       TEXT,
    status                      VARCHAR(20) NOT NULL DEFAULT 'new'
        CHECK (status IN ('new','contacted','trial_booked','trial_completed',
            'negotiating','qualified','converted','lost','dropped')),
    assigned_to                 UUID REFERENCES org.person(id),
    created_by                  UUID,

    -- Conversion links
    converted_to_person_id      UUID REFERENCES org.person(id),
    converted_at                TIMESTAMPTZ,

    -- Timestamps
    contacted_at                TIMESTAMPTZ,
    status_updated_at           TIMESTAMPTZ,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prospect_status ON crm.prospect(status);
CREATE INDEX idx_prospect_assigned ON crm.prospect(assigned_to);
CREATE INDEX idx_prospect_institution ON crm.prospect(institution_id);
CREATE INDEX idx_prospect_phone ON crm.prospect(primary_phone);

-- Additional guardians per prospect (beyond primary guardian)
CREATE TABLE crm.prospect_guardian (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prospect_id         UUID NOT NULL REFERENCES crm.prospect(id) ON DELETE CASCADE,
    guardian_name       TEXT NOT NULL,
    guardian_gender     VARCHAR(10),
    guardian_date_of_birth DATE,
    guardian_nationality TEXT,
    guardian_phone      TEXT,
    guardian_email      TEXT,
    guardian_wechat     TEXT,
    guardian_address    TEXT,
    relationship_type   VARCHAR(30),    -- FATHER/MOTHER/LEGAL_GUARDIAN/STEP_PARENT/GRANDPARENT/RELATIVE/OTHER
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_prospect_guardian_prospect ON crm.prospect_guardian(prospect_id);


-- CRM Student Extension Profile（complements org.person, institution-scoped）
-- Composite PK (institution_id, person_id) enables same person to have
-- different CRM profiles across institutions (multi-tenant isomorphic data).
CREATE TABLE crm.student_profile (
    institution_id      UUID NOT NULL REFERENCES org.education_institution(id),
    person_id           UUID NOT NULL REFERENCES org.person(id) ON DELETE CASCADE,
    grade               VARCHAR(100),
    grade_year          INT,
    grade_month         INT,
    day_school          VARCHAR(200),
    day_school_grade    VARCHAR(100),
    day_school_year     INT,
    day_school_month    INT,
    campus              VARCHAR(100),
    source              VARCHAR(50),
    status              VARCHAR(20) DEFAULT 'enrolled',  -- enrolled/lost/archived
    taboos              TEXT,
    notes               TEXT,
    created_by          UUID REFERENCES auth.user_account(id),
    created_at          TIMESTAMPTZ DEFAULT now(),
    updated_at          TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (institution_id, person_id)
);

-- Marketing Activity
CREATE TABLE crm.marketing_activity (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id    UUID NOT NULL REFERENCES org.education_institution(id),
    title             VARCHAR(200) NOT NULL,
    description       TEXT,
    type              VARCHAR(50) NOT NULL,       -- trial_class/open_day/lecture/event/campaign
    start_date        TIMESTAMPTZ NOT NULL,
    end_date          TIMESTAMPTZ,
    timezone          VARCHAR(50),
    location          VARCHAR(500),
    transportation    VARCHAR(200),
    organization      VARCHAR(200),
    capacity          INT,
    assigned_to       UUID REFERENCES org.person(id),
    status            VARCHAR(20) DEFAULT 'draft',  -- draft/published/in_progress/completed/cancelled
    created_by        UUID REFERENCES auth.user_account(id),
    created_at        TIMESTAMPTZ DEFAULT now(),
    updated_at        TIMESTAMPTZ DEFAULT now()
);

-- Marketing Activity Participants（polymorphic: supports both prospect and student）
-- institution_id enables direct multi-tenant filtering without joining to marketing_activity.
CREATE TABLE crm.marketing_activity_participant (
    institution_id        UUID NOT NULL REFERENCES org.education_institution(id),
    marketing_activity_id UUID NOT NULL REFERENCES crm.marketing_activity(id) ON DELETE CASCADE,
    person_id             UUID REFERENCES org.person(id) ON DELETE CASCADE,
    entity_type           VARCHAR(20) NOT NULL CHECK (entity_type IN ('prospect', 'student')),
    entity_id             UUID NOT NULL,
    status                VARCHAR(20) DEFAULT 'registered',  -- registered/checked_in/checked_out/cancelled
    registered_at         TIMESTAMPTZ DEFAULT now(),
    notes                 TEXT,
    PRIMARY KEY (marketing_activity_id, entity_type, entity_id)
);

-- Task（institution-scoped）
-- institution_id enables direct multi-tenant filtering without joining to related entity.
CREATE TABLE crm.task (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id   UUID NOT NULL REFERENCES org.education_institution(id),
    title            VARCHAR(200) NOT NULL,
    description      TEXT,
    due_date         TIMESTAMPTZ,
    priority         VARCHAR(20) DEFAULT 'medium',  -- low/medium/high/urgent
    status           VARCHAR(20) DEFAULT 'pending',  -- pending/in_progress/completed/cancelled
    assigned_to      UUID REFERENCES org.person(id),
    related_to_type  VARCHAR(50),                   -- prospect/student/guardian/marketing_activity/sales_contract
    related_to_id    UUID,
    reminder_at      TIMESTAMPTZ,
    completed_at     TIMESTAMPTZ,
    created_by       UUID REFERENCES auth.user_account(id),
    created_at       TIMESTAMPTZ DEFAULT now(),
    updated_at       TIMESTAMPTZ DEFAULT now()
);

-- Tag（institution-scoped）
CREATE TABLE crm.tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    color           VARCHAR(7) DEFAULT '#1890ff',
    type            VARCHAR(50) DEFAULT 'student',  -- prospect/student/guardian/marketing_activity/sales_contract
    institution_id  UUID NOT NULL REFERENCES org.education_institution(id),
    created_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_tag_institution ON crm.tag(institution_id);

-- Tag Assignment（polymorphic, institution-scoped）
-- institution_id enables direct multi-tenant filtering without joining to crm.tag.
-- Constraints:
--   entity_type CHECK: 防止脏数据
--   App-layer validation: 创建时校验 entity 存在
--   Indexes: (institution_id, entity_type, entity_id) 用于查询, (tag_id) 用于 FK 反向
CREATE TABLE crm.tag_assignment (
    tag_id      UUID NOT NULL REFERENCES crm.tag(id) ON DELETE CASCADE,
    entity_type VARCHAR(50) NOT NULL CHECK (entity_type IN ('prospect', 'student', 'guardian', 'marketing_activity', 'sales_contract')),
    entity_id   UUID NOT NULL,
    institution_id UUID NOT NULL REFERENCES org.education_institution(id),
    created_at  TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (tag_id, entity_type, entity_id)
);
CREATE INDEX idx_tag_assignment_institution ON crm.tag_assignment(institution_id, entity_type, entity_id);
CREATE INDEX idx_tag_assignment_tag ON crm.tag_assignment(tag_id);

-- Conversion Stage（configurable pipeline, institution-scoped）
CREATE TABLE crm.conversion_stage (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL,
    ord         INT NOT NULL,
    color       VARCHAR(7) DEFAULT '#1890ff',
    probability INT DEFAULT 0,
    institution_id UUID NOT NULL REFERENCES org.education_institution(id)
);
CREATE INDEX idx_conversion_stage_institution ON crm.conversion_stage(institution_id);

-- Sales Opportunity（deal tracking, institution-scoped）
CREATE TABLE crm.conversion (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES org.education_institution(id),
    entity_type         VARCHAR(20) NOT NULL,      -- prospect / student
    entity_id           UUID NOT NULL,
    stage_id            UUID REFERENCES crm.conversion_stage(id),
    amount              DECIMAL(12,2) DEFAULT 0,
    probability         INT,
    expected_close_date DATE,
    assigned_to         UUID REFERENCES org.person(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ DEFAULT now(),
    updated_at          TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_conversion_institution ON crm.conversion(institution_id);
CREATE INDEX idx_conversion_entity ON crm.conversion(entity_type, entity_id);

-- Sales Contract（CRM-side contract signing & payment collection, institution-scoped）
--   classing_contract_id → contract.contract.id after formal activation
--   contract_no is UNIQUE per institution (not globally unique — same contract number
--   may exist in different schools)
CREATE TABLE crm.sales_contract (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id        UUID NOT NULL REFERENCES org.education_institution(id),
    student_person_id     UUID NOT NULL REFERENCES org.person(id) ON DELETE CASCADE,
    classing_contract_id  UUID REFERENCES contract.contract(id),  -- linked after formal activation
    contract_no           VARCHAR(50) NOT NULL,
    type                  VARCHAR(50) NOT NULL,              -- course_package/membership/trial
    items                 JSONB,                             -- [{courseName, totalSessions, unitPrice}]
    total_amount          DECIMAL(12,2) NOT NULL,
    paid_amount           DECIMAL(12,2) DEFAULT 0,
    remaining_sessions    INT DEFAULT 0,
    start_date            DATE NOT NULL,
    end_date              DATE,
    status                VARCHAR(20) DEFAULT 'active',      -- active/expiring/expired/terminated/refunded
    notes                 TEXT,
    created_by            UUID REFERENCES auth.user_account(id),
    created_at            TIMESTAMPTZ DEFAULT now(),
    updated_at            TIMESTAMPTZ DEFAULT now(),
    UNIQUE (institution_id, contract_no)
);
CREATE INDEX idx_sales_contract_institution ON crm.sales_contract(institution_id);

-- Payment（institution-scoped）
CREATE TABLE crm.sales_payment (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id   UUID NOT NULL REFERENCES org.education_institution(id),
    sales_contract_id UUID NOT NULL REFERENCES crm.sales_contract(id) ON DELETE CASCADE,
    amount           DECIMAL(12,2) NOT NULL,
    method           VARCHAR(50) NOT NULL,          -- cash/transfer/wechat/alipay/pos
    receipt_no       VARCHAR(100),
    paid_at          TIMESTAMPTZ DEFAULT now(),
    installment_no   INT DEFAULT 1,
    installment_total INT DEFAULT 1,
    notes            TEXT,
    created_by       UUID REFERENCES auth.user_account(id),
    created_at       TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_sales_payment_institution ON crm.sales_payment(institution_id);

-- Renewal Reminder（institution-scoped）
CREATE TABLE crm.renewal_reminder (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES org.education_institution(id),
    sales_contract_id UUID NOT NULL REFERENCES crm.sales_contract(id),
    person_id   UUID NOT NULL REFERENCES org.person(id) ON DELETE CASCADE,
    trigger_at  TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',      -- pending/sent/completed/cancelled
    assigned_to UUID REFERENCES org.person(id),
    notes       TEXT,
    resolved_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_renewal_institution ON crm.renewal_reminder(institution_id);

-- Churn Warning（institution-scoped）
CREATE TABLE crm.churn_warning (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES org.education_institution(id),
    person_id   UUID NOT NULL REFERENCES org.person(id) ON DELETE CASCADE,
    risk_level  VARCHAR(20) DEFAULT 'medium',       -- low/medium/high
    reason      VARCHAR(50) NOT NULL,               -- contract_expiring/attendance_drop/long_inactive
    description TEXT,
    auto_created BOOLEAN DEFAULT true,
    resolved_at TIMESTAMPTZ,
    resolved_by UUID REFERENCES auth.user_account(id),
    created_at  TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_churn_institution ON crm.churn_warning(institution_id);

-- Communication Record（unified for prospect / student / guardian, institution-scoped）
--   related_student_person_id optional; when entity_type = 'guardian' and the
--   communication concerns a specific student, populate this field.
--   Not a v1.5 DDL column — defined here as a future extension point.
--   entity_type stays as original even after prospect→student conversion
--   Student view queries via crm.prospect.person_id join
--   This preserves the "this happened during lead stage" context
--   email_log_id links to email.email_log for email-type communications
--   institution_id enables direct multi-tenant filtering without joining to entity.
CREATE TABLE crm.communication (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id UUID NOT NULL REFERENCES org.education_institution(id),
    type        VARCHAR(50) NOT NULL,               -- email/call/meeting/note/sms
    subject     VARCHAR(200),
    content     TEXT,
    direction   VARCHAR(20) DEFAULT 'outbound',     -- inbound/outbound
    entity_type VARCHAR(20) NOT NULL CHECK (entity_type IN ('prospect', 'student', 'guardian', 'marketing_activity', 'sales_contract')),
    entity_id   UUID NOT NULL,
    email_log_id UUID REFERENCES email.email_log(id),  -- NULL for non-email communications
    created_by  UUID NOT NULL REFERENCES auth.user_account(id),
    created_at  TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_communication_institution_entity ON crm.communication(institution_id, entity_type, entity_id);

-- Expose legacy compatibility view for CRM migration period
CREATE VIEW crm.email_account AS SELECT * FROM email.email_account;
CREATE VIEW crm.email AS SELECT * FROM email.email_log;
```

> **Note on `entity_type = 'guardian'`**: When `entity_type = 'guardian'`, `entity_id` points to the guardian's own `org.person.id`. `related_student_person_id` is an optional future extension field — **not in v1.5 DDL**. In v1.5, guardian communication queries will not support student-specific filtering; the field is reserved to avoid ambiguous multi-student guardian communication histories in a later release.

> **Parent-Child institution_id Consistency**: Several child tables duplicate `institution_id` for direct multi-tenant filtering (see §3.3 DDL). The application service layer MUST validate that the child's `institution_id` matches its parent object's `institution_id`. Affected pairs:
> - `crm.sales_payment.institution_id` ≡ `crm.sales_contract.institution_id`
> - `crm.renewal_reminder.institution_id` ≡ `crm.sales_contract.institution_id`
> - `crm.marketing_activity_participant.institution_id` ≡ `crm.marketing_activity.institution_id`
> - `crm.tag_assignment.institution_id` ≡ `crm.tag.institution_id`
> - `crm.conversion.institution_id` ≡ `crm.conversion_stage.institution_id` (when stage is set)
>
> Where practical, composite FK constraints may be added in a later hardening migration. For the initial implementation, service-layer validation is sufficient.

### 3.4 Person Required Fields Strategy

CRM optional vs Person NOT NULL 的矛盾通过三阶段数据质量解决：

| Phase | Data State | User Action |
|-------|-----------|-------------|
| **1. Lead**（prospect.NEW） | Name + contact only | Auto-submit from website / admin quick entry |
| **2. Follow-up**（prospect.CONTACTED/QUALIFIED） | Gradually enriched | Sales adds communications, marks Conversion deals |
| **3. Conversion**（Click "Convert to Student"） | ALL Person required fields filled at once | Dialog form + name confirmation + identity document |

Person 表的 NOT NULL 约束**保持不变**。

### 3.5 Name Splitting Rules

| Language | Splitting Logic | Example |
|----------|-----------------|---------|
| English / Polish | Split by last space | `"Jan Kowalski"` → `given_name="Jan"`, `family_name="Kowalski"` |
| Chinese | Heuristic first 1-2 chars as surname, or leave blank for manual fill | `"张三"` → prefill in dialog, not locked |
| Japanese / Korean | No auto-split | |
| Spanish / Arabic | No auto-split | |

**Core Principle**: System makes best-effort guess. **Conversion dialog is always editable.** Admin must confirm before proceeding.

### 3.6 Communication Record Unification

#### Current State

| Dimension | crm.prospect | CRM Communication |
|-----------|------------------|-------------------|
| Storage | Single field `communication TEXT` | Independent table, 12 columns |
| Usage | Ad-hoc notes before conversion | Multi-type, multi-direction, linked to student/parent |
| Problem | Unstructured, not queryable, no history | Data only lives in CRM, not available during prospect stage |

#### Solution: Unify into `crm.communication`

Do NOT keep two communication systems. Use `crm.communication` from the very first prospect touchpoint:

```
Prospect Phase（lightweight）          Student Phase（full）
───────────────────────────           ──────────────────────
crm.communication                     crm.communication
  entity_type = 'prospect'              entity_type = 'student'
  entity_id = crm.prospect.id               entity_id = person.id
  Supports all types:                    Supports all types:
  - note（quick note）                    - note
  - call（phone call）                    - call
  - meeting（in-person）                  - meeting
  - sms（text message）                   - sms
                                         - email（with full subject/body）
```

#### On Conversion: Preserve Context, Don't Rewrite

**Decision: Do NOT change `entity_type` from 'prospect' to 'student' on conversion.**

Reason: Changing `entity_type` loses the context that the communication happened during the lead stage. Preserving the original phase is essential for auditing and sales process analysis.

Instead, query communications for a student by including their prospect history:

```sql
-- View ALL communications for a student（post-conversion）
-- This includes communications that happened during the prospect phase
-- All multi-tenant queries MUST be institution-scoped
SELECT c.*, c.entity_type AS phase
FROM crm.communication c
WHERE c.institution_id = :institutionId
  AND (
       (c.entity_type = 'student' AND c.entity_id = :personId)
    OR (c.entity_type = 'prospect' AND c.entity_id IN (
        SELECT id FROM crm.prospect WHERE person_id = :personId
    ))
  )
ORDER BY c.created_at DESC;
```

This way:
- **Prospect page** sees `WHERE entity_type='prospect' AND entity_id=prospect.id`
- **Student page** sees the UNION query above — all communications from prospect phase AND student phase
- **Context is preserved**: the UI can show a badge "线索阶段" vs "正式学生阶段"

#### Legacy `crm.prospect.message` Field Handling

| Step | Action |
|------|--------|
| **1. Migration** | One-time ETL: parse `crm.prospect.message` text → `crm.communication(type='note', entity_type='prospect')` |
| **2. Retention** | Keep `message` column as free-text note during migration; migrated to `crm.communication` over time |
| **3. Future** | All communication goes through `crm.communication` |

#### Usage Scenarios

| Scenario | Action | Writes To |
|----------|--------|-----------|
| Quick note on prospect list | Click "Note" button | `crm.communication(type='note', entity_type='prospect')` |
| Phone call record | Click "Call" button | `crm.communication(type='call', ...)` |
| Send email | System auto-records | `crm.communication(type='email', email_log_id=email.email_log.id, ...)` |
| View communication history | Unified list component | UNION query across prospect + student |

#### Shared React Component

Prospect detail page and Student detail page reuse the same **CommunicationHistory** component, with an optional `includeProspectHistory` flag:

```tsx
// Prospect view
<CommunicationHistory entityType="prospect" entityId={prospect.id} />

// Student view（includes prospect-phase history automatically）
<CommunicationHistory entityType="student" entityId={person.id} includeProspectHistory />
```

---

## 4. Role-Based Access Under Admin

### 4.1 Decision: No Separate Sales Workspace (For Now)

CRM features stay under `/app/admin/crm/`, NOT as a separate `/app/sales/*` workspace. Rationale:

- Creating a separate sales workspace increases Top Bar, sidebar, routing, permission, and mobile adaptation cost
- CRM is not yet heavy enough to justify its own workspace
- Role-based access within Admin achieves the same separation (marketing staff can't see audit, sales staff can't see org settings)

### 4.2 Role Matrix

| Role | CRM Access | Org Access | Email / Notification Access | Audit Access |
|------|-----------|------------|------------------|--------------|
| `principal` | Read all | Full | Read + Send | Read |
| `academic_director` | Read students/contracts | Read | Read | None |
| `marketing` | Read/write prospects | None | Read email logs | None |
| `operations` | Full CRM | Full | Full | None |
| `finance` | Read sales contracts / payments | None by default | View billing-related logs | Finance audit only |
| `sales`（new） | CRM operational access（assigned by default; all configurable） | Read students only | Send email | None |
| `teacher` | None by default | None | Send/view own class-related messages | None |
| `it_admin` | Full | Full | Full | Full |

> **Note:** `'sales'` must be added to `enum.org_institution_role_enum` in `V0_2__enums.sql` by editing the `CREATE TYPE` list directly（no ALTER needed — system is pre-production）. Full enum values after update: `principal`, `academic_director`, `finance`, `it_admin`, `marketing`, `operations`, `sales`.
> 
> `system_maintainer` is no longer in `org_institution_role_enum` — it is a **platform role** assigned through `auth.platform_role_assignment` using `enum.platform_role_enum` (`saas_operator`, `system_maintainer`).

### 4.3 Target Route Structure

```
/app/admin
├── /messages/                ← Personal inbox / unread / message detail
├── /org/                     ← Org management（principal, it_admin）
│   ├── institutions
│   ├── persons
│   ├── members
│   └── rooms
│
├── /crm/                     ← CRM（sales, operations, marketing）
│   ├── /prospects            Lead Pool（NEW → CONTACTED → QUALIFIED）
│   ├── /students             Student CRM Profiles
│   ├── /students/:id/guardians  Student Guardian Links（admin only）
│   ├── /marketing-activities Marketing Activity Management
│   ├── /tasks                Task Board
│   ├── /conversions          Conversion Pipeline（deals + stages）
│   ├── /sales-contracts      Contract Signing & Payments
│   ├── /renewals             Renewal Reminders
│   ├── /churn-warnings       Churn Warnings
│   ├── /communications       Communication History
│   └── /tags                 Tag Management
│
├── /email/                   ← Email infrastructure（operations, it_admin）
│   ├── /accounts             SMTP Accounts
│   ├── /templates            Email Templates
│   ├── /logs                 Send History（email.email_log）
│   └── /delivery-status      Per-recipient status（email.email_delivery）
├── /notifications/           ← In-app notifications（all roles）
│
└── /audit                    ← Audit Log（it_admin, principal）
```

### 4.4 Email & Notification Permission Matrix

| Domain | Permission | teacher | sales | marketing | operations | it_admin | principal |
|--------|-----------|---------|-------|-----------|------------|----------|-----------|
| **Email** | Send email | ✅ Own classes/students | ✅ Own prospects/students | ✅ Campaigns | ✅ All | ✅ All | ✅ All |
| | View send logs | ✅ Own only | ✅ Own only | ✅ Campaigns | ✅ All | ✅ All | ✅ All |
| | Manage email accounts | ❌ | ❌ | ❌ | ✅ CRUD | ✅ CRUD | ❌ |
| | Manage templates | ❌ | ❌ | ✅ CRUD | ✅ CRUD | ✅ All | ❌ |
| | View delivery status | ✅ Own only | ✅ Own only | ✅ Campaigns | ✅ All | ✅ All | ❌ |
| | Retry failed emails | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| **Notification**（in-app） | Receive notifications | ✅ System-generated | ✅ System-generated | ✅ System-generated | ✅ All | ✅ All | ✅ All |
| | Mark as read | ✅ Own | ✅ Own | ✅ Own | ✅ Own | ✅ Own | ✅ Own |
| | Send broadcast | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |

Enforcement: API-level via Spring Security method annotations (`@PreAuthorize`). Frontend menu/hide driven by `userContext.permissions` from session.

**Teacher permission note:** Legacy placeholder names (`teacher_send_message`, `teacher_view_own_messages`, `teacher_view_own_delivery_status`) are used across docs for backward compatibility. These will be refined into separate email and notification permissions during implementation (e.g. `teacher_send_email`, `teacher_view_own_notifications`). The current names represent the teacher's ability to communicate with their own students/parents, which spans both email and in-app notification domains.

### 4.5 Future: Extract Sales Workspace

When CRM grows to include:
- Sales-specific dashboard（KPIs, pipeline charts, team performance）
- Independent mobile entry point
- Distinct UX from admin（different color scheme, navigation patterns）

Then extract:

```
/app/sales/*       ← CRM operational features（prospects, students, sales-contracts, marketing-activities, tasks）
/app/admin/crm/*   ← Only CRM configuration（tags, stages, churn config）
```

---

## 5. API Integration

### 5.1 API Structure

```
/api/session/context              ← Current workspace + role + permissions（auth schema）

/api/admin/crm/**
  ├── /prospects                ← Lead pool + convert to student
  ├── /students                 ← CRUD + search + filter by tag
  ├── /students/:id/guardians   ← Guardian links（admin only, org.guardian_assignment）
  ├── /students/:id/tags        ← Tag assign/unassign
  ├── /marketing-activities     ← CRUD + register/checkin
  ├── /tasks                    ← CRUD + byUser
  ├── /sales-contracts          ← CRUD + payments
  ├── /renewals                 ← CRUD + resolve/dismiss
  ├── /churn-warnings           ← CRUD + resolve
  ├── /communications           ← read + add_communication + byEntity
  ├── /conversions              ← Deals CRUD + stage move
  ├── /conversion-stages        ← Stage configuration（independent API）
  ├── /tags                     ← CRUD
  └── /tag-assignments          ← Assign/unassign（polymorphic: entity_type + entity_id）

/api/admin/email/**             ← Email infrastructure（email schema）
  ├── /logs                     ← Send records（email.email_log）
  ├── /logs/:id/send            ← Send / resend
  ├── /logs/:id/retry           ← Manual retry
  ├── /templates                ← Email templates（email.email_template）
  ├── /accounts                 ← SMTP account management（email.email_account）
  └── /delivery-status          ← Per-recipient delivery（email.email_delivery）

/api/admin/notifications/**     ← In-app notifications（messaging schema）
  ├── /inbox                    ← My notifications
  ├── /:id                      ← Single notification
  └── /broadcast                ← Send broadcast（principal, operations, it_admin）

Removed:
  /api/admin/prospects/**
```

### 5.2 Auth Unification

| Current CRM | Target（ClassingSystem auth） |
|-------------|------------------------------|
| Simple JWT | JWT + refresh token（15min/30d） |
| email + password login | login_name + password + institution_code |
| `users` table | `auth.user_account` + `auth.user_password` |
| role = "sales" | `org.institution_role_assignment.role_code = 'sales'`（under admin workspace） |

**Migration Strategy**:
1. Import current CRM users into `auth.user_account` + `auth.user_password`
2. Create corresponding `org.person` records for each user
3. Frontend auth flow change: `POST /api/auth/login` → JWT → `GET /api/session/context` → workspace routing

### 5.3 NestJS → Spring Boot Migration

Package Structure:

```
com/
  ├── admin/
  │   └── crm/
  │       ├── controller/
  │       │   ├── ProspectController.kt
  │       │   ├── StudentController.kt
  │       │   ├── MarketingActivityController.kt
  │       │   ├── TaskController.kt
  │       │   ├── SalesContractController.kt
  │       │   ├── CommunicationController.kt
  │       │   ├── ConversionController.kt
  │       │   ├── TagController.kt
  │       │   └── ChurnController.kt
  │       ├── service/
  │       ├── repository/
  │       ├── dto/
  │       └── mapper/
  │
  ├── email/
  │   ├── EmailService.kt
  │   ├── controller/
  │   │   ├── EmailLogController.kt
  │   │   ├── EmailTemplateController.kt
  │   │   ├── EmailAccountController.kt
  │   │   └── EmailDeliveryController.kt
  │   ├── service/
  │   ├── repository/
  │   └── dto/
  │
  └── messaging/
      ├── NotificationService.kt
      ├── controller/
      │   ├── NotificationController.kt
      │   └── BroadcastController.kt
      ├── service/
      ├── repository/
      └── dto/
```

**Migration Strategy**: Module-by-module. Start with the most-used module（Student Management）, then progressively cover all endpoints. The NestJS server can be decommissioned once all modules are migrated.

---

## 6. Frontend Integration

### 6.1 Adaptation Strategy

Current CRM React frontend needs the following adaptations to connect to the Spring Boot API:

| Item | Current（NestJS） | Target（Spring Boot） |
|------|------------------|----------------------|
| API baseURL | `http://localhost:3000/api` | `/api`（same-origin） |
| Auth header | `Bearer ${token}` | `Bearer ${accessToken}` |
| Login flow | `POST /api/auth/login` → token | `POST /api/auth/login` → accessToken + refreshToken + sessionContext |
| Current user | `authStore.user` | `authStore.user`（from session context） |
| createdBy | Login user email | `auth.user_account.id`（from JWT） |

### 6.2 React SPA Shell Integration

Following `Wonders_Website_Architecture_Evolution_v1.5_Final_Requirements_Frozen.md`, CRM modules live under Admin Workspace:

```
/app/admin/crm/prospects         → Prospect List & Conversion（merged from legacy + CRM）
/app/admin/crm/students          → StudentList（adapted from CRM）
/app/admin/crm/students/:id      → StudentDetail（adapted from CRM）
/app/admin/crm/marketing-activities    → MarketingActivityList（adapted from CRM）
/app/admin/crm/tasks             → TaskBoard（adapted from CRM）
/app/admin/crm/conversions       → ConversionPipeline（adapted from CRM）
/app/admin/crm/sales-contracts   → ContractList + Payment（adapted from CRM）
/app/admin/crm/renewals          → RenewalList（adapted from CRM）
/app/admin/crm/churn-warnings    → WarningBoard（adapted from CRM）
/app/admin/crm/communications    → CommunicationHistory（new unified component）
/app/admin/crm/tags              → Tag Management（from CRM sidebar modal）

/app/admin/email/accounts         → Email Account Management
/app/admin/email/templates        → Email Template Management
/app/admin/email/logs             → Email Send History
/app/admin/email/delivery-status  → Delivery Status Tracking
/app/admin/notifications/*        → In-app Notifications
```

Sidebar Menu:

```
Admin
├── Messages           （/app/admin/messages/*）
├── CRM
│   ├── Lead Pool        （/app/admin/crm/prospects）       [sales, marketing, operations]
│   ├── Students         （/app/admin/crm/students）        [sales, operations, academic_director]
│   ├── Conversion Pipeline （/app/admin/crm/conversions）  [sales, operations]
│   ├── Sales Contracts  （/app/admin/crm/sales-contracts） [sales, operations]
│   ├── Marketing Activities （/app/admin/crm/marketing-activities） [sales, operations, marketing]
│   ├── Renewals         （/app/admin/crm/renewals）        [sales, operations]
│   ├── Churn Warnings   （/app/admin/crm/churn-warnings）  [operations, principal]
│   ├── Task Board       （/app/admin/crm/tasks）           [all CRM roles]
│   ├── Communications   （/app/admin/crm/communications）  [all CRM roles]
│   └── Tags             （/app/admin/crm/tags）            [operations, it_admin]
│
├── Email
│   ├── Accounts         （/app/admin/email/accounts）      [operations, it_admin]
│   ├── Templates        （/app/admin/email/templates）     [operations, marketing]
│   ├── Logs             （/app/admin/email/logs）          [all]
│   └── Delivery Status  （/app/admin/email/delivery-status）[operations, it_admin]
│
├── Notifications        （/app/admin/notifications/*）     [all roles]
├── Organization         （/app/admin/org/*）               [principal, it_admin]
└── Audit Log            （/app/admin/audit）               [principal, it_admin]
```

---

## 7. Implementation Roadmap

### Current State（Pre-Phase 0）

Before starting, the existing codebase has these resources ready:

| Module | Backend Status | Frontend Status |
|--------|---------------|-----------------|
| `com/email/` | ✅ Fully implemented: EmailTemplateEntity, EmailLogEntity, EmailRateLimitEntity, EmailSenderService, EmailTemplateService, EmailTemplateController + 4-layer pattern with DTOs | — |
| `com/messaging/` | ⬜ Scaffold exists（empty controller/service/entity/dto/repository dirs） | — |
| `com/prospect/` | ⬜ Old `prospect.prospect` code exists — must be migrated to `com.admin.crm.prospect` | — |
| `com/admin/crm/` | ✅ Exists（from Sprint 0–3）— full backend: prospect, tag, communication, conversion, task, marketing, contract, payment, renewal, churn_warning, student, guardian. Needs v1.5 alignment: DDL（institution_id）, permissions（@PreAuthorize）, routing（workspace prefix）, tenant scope | ✅ 16 modules from NestJS, need route/permission adaptation |
| CRM Frontend（NestJS） | — | ✅ 16 modules: students, parents, activities, contracts, conversions, renewals, warnings, tasks, communications, tags, email-accounts, emails, auth, users |
| ClassingSystem Auth | ✅ `com.security/` + `com.auth/` with JWT + refresh token | — |

### Phase 0 — Database & Foundation（Week 1）

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Write `V2_2__extend_email_and_messaging.sql` | `email.email_account`, `email.email_log`（含 `email_account_id` FK）, `email.email_delivery`, `messaging.notification`, `messaging.notification_recipient`（email_log 从 V2_1 移来，使其 `email_account_id` 在 CREATE TABLE 中直接引用 email_account） |
| 3-4 | Write `V2_3__crm_tables.sql` | All `crm.*` tables with columns, CHECK constraints, indexes, FK references |
| 5 | Review and align existing `com.admin.crm` package to v1.5 conventions | Normalize controller / service / repository structure, package naming, DTO patterns |
| 5 | Auth adaptation | Verify `sales` role exists in `enum.org_institution_role_enum`； `GET /api/session/context` returns role/permissions |
| 6 | Integration smoke test | Flyway run → verify all tables + API boots without error |

### Phase 1 — Backend CRM Core（Week 2-3）

| Week | Module | Task | Migration Note |
|------|--------|------|----------------|
| 2 | **Prospect** | Migrate from `com/prospect/` to `com.admin.crm.prospect` | Map `prospect.prospect` fields → `crm.prospect`（`name`→`legal_full_name`, add `grade_of_next_academic_year`, use `institution_id` FK） |
| 2 | **Communication** | Align existing with v1.5 DDL, institution scope, and append-only policy | polymorphic `entity_type`/`entity_id` query, `email_log_id` FK; append-only — no modify, archive, or physical DELETE in v1.5 |
| 2 | **Tag** | Align existing `TagEntity` + `TagAssignmentEntity` | CHECK constraint + app-layer entity existence validation; DELETE restricted: only if no assignments |
| 3 | **Conversion** | Align existing pipeline + stages | Stage move logic, prospect-bound deal tracking; archive instead of DELETE |
| 3 | **Task** | Align existing polymorphic task | `related_to_type`/`related_to_id`, person assignment; DELETE restricted to draft/cancelled |
| 3 | **Student Profile** | Align existing extension table for `org.person` | grade, daySchool, campus, taboos, status; archive instead of DELETE |

### Phase 2 — Backend CRM Extended（Week 4-5）

| Week | Module | Key Features |
|------|--------|--------------|
| 4 | **Marketing Activity** | CRUD + participant（`entity_type='prospect'\|'student'`）+ register/checkin |
| 4 | **Sales Contract** | `crm.sales_contract` + `crm.sales_payment`, `classing_contract_id` link to `contract.contract` |
| 5 | **Renewal** | Reminder CRUD + resolve/dismiss |
| 5 | **Churn Warning** | CRUD + resolve |
| 5 | **Permission Annotation** | Add `@PreAuthorize` guards per §4.2 Role Matrix |

### Phase 3 — Frontend API Adaptation（Week 6）

| Day | Task | Scope |
|-----|------|-------|
| 1 | Switch axios baseURL → Spring Boot | `apps/client/vite.config.ts`, axios instance |
| 2 | Auth flow replacement | Old JWT → refresh token + `GET /api/session/context` |
| 3 | createdBy from JWT payload | All create/update calls |
| 4-5 | API path remapping | `/api/students` → `/api/admin/crm/students` etc., per Appendix A mapping |
| 6 | Smoke test | Each module: at least one CRUD cycle passing |

### Phase 4 — Frontend CRM Pages（Week 7-8）

| Week | Page | Components |
|------|------|-----------|
| 7 | **Prospect List + Conversion Dialog** | Table + Convert button → dialog（name confirm + required fields + grade prefill） |
| 7 | **Student List + Detail** | Table with tag badges + detail page with Tabs |
| 7 | **CommunicationHistory** | Shared component, `includeProspectHistory` param |
| 8 | **Marketing Activities** | List + detail + register/checkin UI |
| 8 | **Sales Contracts** | List + detail + payment records |
| 8 | **Tasks / Renewals / Tags** | Reuse existing Ant Design components, update endpoint + field names |

### Phase 5 — Email & Notifications（Week 9）

| Day | Task | Detail |
|-----|------|--------|
| 1-2 | `EmailAccountEntity` + `EmailDeliveryEntity` | `com.email` already has template/log/rateLimit — add account + delivery |
| 3-4 | `NotificationEntity` + `NotificationRecipientEntity` | `com.messaging`（currently empty scaffold）— implement notification CRUD + broadcast |
| 5 | Wire CRM communication → EmailService | `crm.communication(type='email')` auto-creates `email.email_log` |
| 5-6 | Wire permission annotations | `@PreAuthorize` per §4.4 Email & Notification Permission Matrix |

### Phase 6 — Decommission & Polish（Week 10）

| Day | Task |
|-----|------|
| 1-2 | PWA + responsive optimization |
| 3-4 | Delete NestJS `apps/server` modules（keep `apps/client`） |
| 3-4 | Delete old `com/prospect/` package |
| 5-6 | Redirect legacy pages: `/admin-prospect.html` → 301 → `/app/admin/crm/prospects`; `/admin-login.html` → `/app/login` |

---

## 8. Architecture Decision Records（ADR）

| ID | Decision | Choice | Rationale |
|----|----------|--------|-----------|
| ADR-001 | Frontend Framework | React | Architecture docs already specify + all existing CRM code is React |
| ADR-002 | Mobile Platform | React Native | Architecture docs final goal |
| ADR-003 | Person NOT NULL Constraints | Keep as-is | Compliance requirements; lightweight entry via prospect phase |
| ADR-004 | Name Splitting | System guess + manual confirm dialog | Multi-language names cannot be 100% automated |
| ADR-005 | Route Naming | `/app/admin/crm/*` | Semantically clear, independent business domain |
| ADR-006 | Lead & Student Management | Unified（Option B） | Eliminate duplicate status, single data flow |
| ADR-007 | CRM Status Removal | Deprecate lead/prospect status | Unify under crm.prospect.status |
| ADR-008 | Database | Merge into ClassingSystem instance | Single DB single backend, avoid dual deployment |
| ADR-009 | CRM Backend | Migrate to Spring Boot | Unified backend tech stack |
| ADR-010 | CRM Frontend | Keep React, adapt API | Preserve UI code, replace API layer |
| ADR-011 | Communication on Conversion | Keep `entity_type='prospect'`, join via `crm.prospect.person_id` | Preserves "this happened during lead stage" context |
| ADR-012 | Prospect Legacy Field | Keep `crm.prospect.message` as free-text note during migration; migrate to `crm.communication` over time | Old `prospect.prospect.communication` + `message` absorbed into `crm.prospect.message` |
| ADR-013 | Contract Model | Two-table: `crm.sales_contract` + `contract.contract` | Clean separation between sales signing and fulfillment |
| ADR-014 | Email Module | Extend existing `email` schema（email_account / email_delivery）; `messaging` = in-app notifications only | Keep email infra in `email` schema（V2_1 already has template/log/rate_limit）; avoid duplicating into `messaging` |
| ADR-015 | Tag Assignment Constraints | CHECK + app validation + indexes | Polymorphic tables need defense against dirty data |
| ADR-016 | Communication Entity Types | `prospect`, `student`, `guardian`, `marketing_activity`, `sales_contract` | Cover all CRM-touchable entities |
| ADR-017 | Tag Assignment Entity Types | `prospect`, `student`, `guardian`, `marketing_activity`, `sales_contract` | Align with communication entity types for consistency |
| ADR-018 | Sales Role | Add `sales` as `role_code` via `org.institution_role_assignment` under admin workspace | Granular access control without creating new workspace |
| ADR-019 | Future Sales Workspace | Extract when CRM has dedicated dashboard + mobile entry + distinct UX | Avoid premature splitting; reversible architecture |
| ADR-020 | Prospect Table | Use `crm.prospect` as canonical prospect table | CRM owns the lead lifecycle; avoid split between `prospect` and `crm` schemas |
| ADR-021 | Marketing Activity Naming | Use `crm.marketing_activity` instead of `crm.activity` | Prevent confusion with formal class sessions and academic execution records |
| ADR-022 | Teacher Messaging | Include teacher send/view-own permissions in messaging model | Teachers will need class-related parent/student messaging |
| ADR-023 | Admin Messages vs Notifications/Email | `/app/admin/messages/*` is personal inbox; `/app/admin/email/*` is email infra; `/app/admin/notifications/*` is in-app broadcast | Avoid route and concept collision across three distinct surfaces |

---

## Appendix A: Full API Endpoint Mapping

| # | Current CRM Module | Current NestJS Endpoint | Target Spring Boot Endpoint | Target Controller | Permission Object | Permission Action | Notes |
|---|-------------------|------------------------|----------------------------|-------------------|-------------------|------------------|-------|
| 1 | **Auth** | POST /api/auth/login | POST /api/auth/login | — (global AuthController) | — | — | Uses ClassingSystem JWT |
| 2 | | GET /api/auth/me | GET /api/auth/me | — (global AuthController) | — | — | Returns userContext with permissions |
| 3 | | POST /api/auth/register | N/A (→ org invitation flow) | — | — | — | Registration replaced by institution invitation |
| 4 | | POST /api/auth/change-password | POST /api/auth/change-password | — | — | — | Delegated to ClassingSystem auth |
| 5 | | POST /api/auth/reset-password | POST /api/auth/reset-password | — | — | — | Delegated to ClassingSystem auth |
| 6 | **Students** | GET /api/students | GET /api/admin/crm/students | StudentController | crm.student_profile | read | Filtered by institution |
| 7 | | GET /api/students/:id | GET /api/admin/crm/students/:id | StudentController | crm.student_profile | read | personId lookup |
| 8 | | POST /api/students | POST /api/admin/crm/students | StudentController | crm.student_profile | add | Requires org.person + crm.student_profile |
| 9 | | PUT /api/students/:id | PUT /api/admin/crm/students/:id | StudentController | crm.student_profile | modify | |
| 10 | | PUT /api/students/:id/archive | PUT /api/admin/crm/students/:id/archive | StudentController | crm.student_profile | archive | Set crm.student_profile.status = archived; no physical DELETE |
| 11 | **Guardians（Student-scoped only）** | GET /api/parents/by-student/:studentId | GET /api/admin/crm/students/:id/guardians | GuardianController | org.guardian_assignment | read | Student-scoped guardian list; no standalone `/api/admin/crm/guardians` list endpoint |
| 12 | | POST /api/parents（student-scoped） | POST /api/admin/crm/students/:id/guardians | GuardianController | org.guardian_assignment | add | Creates org.person + org.guardian_assignment; no global guardian create |
| 13 | | DELETE /api/parents/:id（student-scoped） | DELETE /api/admin/crm/students/:id/guardians/:personId | GuardianController | org.guardian_assignment | revoke | Unlink guardian from student (revoke the assignment); guardian person data kept in org. No physical DELETE — the relationship record is voided with valid_to. |
| 14 | | **Guardian person detail → use `/api/admin/org/persons/:personId`** | | — | org.person | read | Guardian person fields managed via org module |
| 17 | **Activities** | GET /api/activities | GET /api/admin/crm/marketing-activities | MarketingActivityController | crm.marketing_activity | read | Renamed + kebab-case |
| 18 | | GET /api/activities/:id | GET /api/admin/crm/marketing-activities/:id | MarketingActivityController | crm.marketing_activity | read | |
| 19 | | POST /api/activities | POST /api/admin/crm/marketing-activities | MarketingActivityController | crm.marketing_activity | add | |
| 20 | | PUT /api/activities/:id | PUT /api/admin/crm/marketing-activities/:id | MarketingActivityController | crm.marketing_activity | modify | |
| 21 | | PUT /api/activities/:id/cancel | PUT /api/admin/crm/marketing-activities/:id/cancel | MarketingActivityController | crm.marketing_activity | cancel | Cancel activity; no physical DELETE |
| 22 | | POST /api/activities/:id/register | POST /api/admin/crm/marketing-activities/:id/participants | MarketingActivityController | crm.marketing_activity | register | Register prospect/student via entity_type |
| 23 | | POST /api/activities/:id/checkin | PUT /api/admin/crm/marketing-activities/:id/participants/:participantId/checkin | MarketingActivityController | crm.marketing_activity | checkin | |
| 24 | | POST /api/activities/:id/createTask | POST /api/admin/crm/tasks (with related_to_type=marketing_activity) | TaskController | crm.task | add | Replaced by generic task creation |
| 25 | **Conversions** | GET /api/conversions/stages | GET /api/admin/crm/conversion-stages | ConversionController | crm.conversion_stage | read | |
| 26 | | POST /api/conversions/stages | POST /api/admin/crm/conversion-stages | ConversionController | crm.conversion_stage | add | |
| 27 | | PUT /api/conversions/stages/:id | PUT /api/admin/crm/conversion-stages/:id | ConversionController | crm.conversion_stage | modify | |
| 28 | | DELETE /api/conversions/stages/:id | DELETE /api/admin/crm/conversion-stages/:id | ConversionController | crm.conversion_stage | delete | Restricted — only allowed if no conversion references this stage |
| 29 | | GET /api/conversions | GET /api/admin/crm/conversions | ConversionController | crm.conversion | read | |
| 30 | | GET /api/conversions/:id | GET /api/admin/crm/conversions/:id | ConversionController | crm.conversion | read | |
| 31 | | POST /api/conversions | POST /api/admin/crm/conversions | ConversionController | crm.conversion | add | |
| 32 | | PUT /api/conversions/:id | PUT /api/admin/crm/conversions/:id | ConversionController | crm.conversion | modify | |
| 33 | | PUT /api/conversions/:id/archive | PUT /api/admin/crm/conversions/:id/archive | ConversionController | crm.conversion | archive | Archive deal; no physical DELETE |
| 34 | | PUT /api/conversions/:id/move | PUT /api/admin/crm/conversions/:id/move | ConversionController | crm.conversion | move_stage | Change stage + probability |
| 35 | **Contracts** | GET /api/contracts | GET /api/admin/crm/sales-contracts | SalesContractController | crm.sales_contract | read | Renamed; kebab-case |
| 36 | | GET /api/contracts/:id | GET /api/admin/crm/sales-contracts/:id | SalesContractController | crm.sales_contract | read | |
| 37 | | POST /api/contracts | POST /api/admin/crm/sales-contracts | SalesContractController | crm.sales_contract | add | |
| 38 | | PUT /api/contracts/:id | PUT /api/admin/crm/sales-contracts/:id | SalesContractController | crm.sales_contract | modify | |
| 39 | | PUT /api/contracts/:id/void-draft | PUT /api/admin/crm/sales-contracts/:id/void-draft | SalesContractController | crm.sales_contract | void_draft | Void only draft contracts; active contracts use terminate; no physical DELETE |
| 40 | | POST /api/contracts/:id/payments | POST /api/admin/crm/sales-contracts/:id/payments | SalesContractController | crm.sales_payment | add | |
| 41 | **Payments** | (via contracts) | GET /api/admin/crm/sales-contracts/:id/payments | SalesContractController | crm.sales_payment | read | Payments as sub-resource |
| 42 | | (via contracts) | GET /api/admin/crm/sales-payments/:id | SalesContractController | crm.sales_payment | read | |
| 43 | **Renewals** | GET /api/renewals | GET /api/admin/crm/renewals | RenewalController | crm.renewal_reminder | read | |
| 44 | | GET /api/renewals/:id | GET /api/admin/crm/renewals/:id | RenewalController | crm.renewal_reminder | read | |
| 45 | | POST /api/renewals | POST /api/admin/crm/renewals | RenewalController | crm.renewal_reminder | add | |
| 46 | | PUT /api/renewals/:id | PUT /api/admin/crm/renewals/:id | RenewalController | crm.renewal_reminder | modify | |
| 47 | | PUT /api/renewals/:id/cancel | PUT /api/admin/crm/renewals/:id/cancel | RenewalController | crm.renewal_reminder | cancel | Cancel reminder; no physical DELETE |
| 48 | | PUT /api/renewals/:id/resolve | PUT /api/admin/crm/renewals/:id/resolve | RenewalController | crm.renewal_reminder | resolve | |
| 49 | | PUT /api/renewals/:id/dismiss | PUT /api/admin/crm/renewals/:id/dismiss | RenewalController | crm.renewal_reminder | dismiss | |
| 50 | **Warnings** | GET /api/warnings | GET /api/admin/crm/churn-warnings | ChurnWarningController | crm.churn_warning | read | Renamed; kebab-case |
| 51 | | GET /api/warnings/:id | GET /api/admin/crm/churn-warnings/:id | ChurnWarningController | crm.churn_warning | read | |
| 52 | | POST /api/warnings | POST /api/admin/crm/churn-warnings | ChurnWarningController | crm.churn_warning | add | |
| 53 | | PUT /api/warnings/:id | PUT /api/admin/crm/churn-warnings/:id | ChurnWarningController | crm.churn_warning | modify | |
| 54 | | N/A — use resolve | N/A | — | — | — | No DELETE; use PUT /api/admin/crm/churn-warnings/:id/resolve (row 55) |
| 55 | | PUT /api/warnings/:id/resolve | PUT /api/admin/crm/churn-warnings/:id/resolve | ChurnWarningController | crm.churn_warning | resolve | |
| 56 | **Communications** | GET /api/communications | GET /api/admin/crm/communications | CommunicationController | crm.communication | read | |
| 57 | | GET /api/communications/:id | GET /api/admin/crm/communications/:id | CommunicationController | crm.communication | read | |
| 58 | | POST /api/communications | POST /api/admin/crm/communications | CommunicationController | crm.communication | add_communication | Communication records are append-only (audit trail); no modify or archive in v1.5 |
| 59 | | — | — | — | — | — | Communication modify is NOT allowed in v1.5 (history is immutable) |
| 60 | | — | — | — | — | — | Communication archive is NOT needed in v1.5 (no physical DELETE) |
| 61 | | GET /api/communications/by-student/:studentId | GET /api/admin/crm/communications?entity_type=student&entity_id=:personId | CommunicationController | crm.communication | read | Unified query param |
| 62 | **Tasks** | GET /api/tasks | GET /api/admin/crm/tasks | TaskController | crm.task | read | |
| 63 | | GET /api/tasks/:id | GET /api/admin/crm/tasks/:id | TaskController | crm.task | read | |
| 64 | | POST /api/tasks | POST /api/admin/crm/tasks | TaskController | crm.task | add | |
| 65 | | PUT /api/tasks/:id | PUT /api/admin/crm/tasks/:id | TaskController | crm.task | modify | |
| 66 | | DELETE /api/tasks/:id | DELETE /api/admin/crm/tasks/:id | TaskController | crm.task | delete | Restricted to draft/cancelled status; completed tasks archived, not deleted |
| 67 | | GET /api/tasks/by-user/:userId | GET /api/admin/crm/tasks?assigned_to=:personId | TaskController | crm.task | read | Unified query param |
| 68 | **Tags** | GET /api/tags | GET /api/admin/crm/tags | TagController | crm.tag | read | |
| 69 | | GET /api/tags/:id | GET /api/admin/crm/tags/:id | TagController | crm.tag | read | |
| 70 | | POST /api/tags | POST /api/admin/crm/tags | TagController | crm.tag | add | |
| 71 | | PUT /api/tags/:id | PUT /api/admin/crm/tags/:id | TagController | crm.tag | modify | |
| 72 | | DELETE /api/tags/:id | DELETE /api/admin/crm/tags/:id | TagController | crm.tag | delete | Restricted — only allowed if no tag_assignments reference this tag |
| 73 | | POST /api/tags/:id/assign | POST /api/admin/crm/tag-assignments | TagAssignmentController | crm.tag_assignment | add | entity_type/entity_id polymorphic |
| 74 | | DELETE /api/tags/:id/unassign/:studentId | DELETE /api/admin/crm/tag-assignments/:tagId/:entityType/:entityId | TagAssignmentController | crm.tag_assignment | revoke | |
| 75 | | GET /api/tags/:id/students | GET /api/admin/crm/tags/:id/assignments | TagAssignmentController | crm.tag_assignment | read | |
| 76 | **EmailAccounts** | GET /api/email-accounts | GET /api/admin/email/accounts | EmailAccountController | email.account | read | Extended from existing email schema |
| 77 | | GET /api/email-accounts/:id | GET /api/admin/email/accounts/:id | EmailAccountController | email.account | read | |
| 78 | | POST /api/email-accounts | POST /api/admin/email/accounts | EmailAccountController | email.account | add | |
| 79 | | PUT /api/email-accounts/:id | PUT /api/admin/email/accounts/:id | EmailAccountController | email.account | modify | |
| 80 | | DELETE /api/email-accounts/:id | DELETE /api/admin/email/accounts/:id | EmailAccountController | email.account | delete | |
| 81 | **Emails** | GET /api/emails | GET /api/admin/email/logs | EmailLogController | email.log | read | Mapped to existing email.email_log |
| 82 | | GET /api/emails/:id | GET /api/admin/email/logs/:id | EmailLogController | email.log | read | |
| 83 | | POST /api/emails | POST /api/admin/email/logs | EmailLogController | email.log | add | Uses email.email_log directly |
| 84 | | — | PUT /api/admin/email/logs/:id | EmailLogController | email.log | modify | |
| 85 | | — | N/A | — | — | — | No DELETE — email logs are audit records |
| 86 | | POST /api/emails/:id/send | POST /api/admin/email/logs/:id/send | EmailLogController | email.send | send | Triggers SMTP send via email_account |
| 87 | **Templates** | (none in current CRM) | GET /api/admin/email/templates | EmailTemplateController | email.template | read | Already exists in email schema |
| 88 | | — | POST /api/admin/email/templates | EmailTemplateController | email.template | add | |
| 89 | | — | PUT /api/admin/email/templates/:id | EmailTemplateController | email.template | modify | |
| 90 | | — | DELETE /api/admin/email/templates/:id | EmailTemplateController | email.template | delete | |
| 91 | **Delivery** | (none in current CRM) | GET /api/admin/email/delivery-status | EmailDeliveryController | email.delivery | read | New table in email schema |
| 92 | | — | GET /api/admin/email/delivery-status/:id | EmailDeliveryController | email.delivery | read | |
| 93 | | — | POST /api/admin/email/logs/:id/retry | EmailLogController | email.log | retry | Retry failed email |

## Appendix B: Entity Relationship Overview

### B.1 Current CRM ER（Prisma 16 Models）

Source: `apps/server/prisma/schema.prisma`

```
User 1 ── N Student             （createdBy）
User 1 ── N Task                （assignedTo / createdBy）
User 1 ── N Communication       （createdBy）
│
Student 1 ── N StudentParent    （composite PK: studentId + parentId）
Parent  1 ── N StudentParent
Student N ── N Tag              （via StudentTag）
Student 1 ── N Contract
Student 1 ── N Payment           （via Contract）
Student 1 ── N Communication
Student 1 ── N RenewalReminder
Student 1 ── N ChurnWarning
Student 1 ── N ActivityParticipant
Activity 1 ── N ActivityParticipant
Student 1 ── N Conversion
ConversionStage 1 ── N Conversion
EmailAccount 1 ── N Email
│
Contract 1 ── N Payment
Contract 1 ── N RenewalReminder
│
Task ── student  （relatedToType/relatedToId）
Attachment ── polymorphic （relatedToType/relatedToId）
AuditLog ── entityType/entityId
```

Key observations of current CRM:
- All 16 models live in a single private `public` schema
- Student is the central hub connected to 12+ other models
- Tags use `student_tags` junction table (student-only, not polymorphic)
- Activities, Tasks, Communications, Attachments each have their own polymorphic patterns
- No `org`, `auth`, `messaging`, or `contract` schema separation

### B.2 Target Integrated ER（ClassingSystem + CRM）

After migration, CRM models distribute across 5 schemas:

```
── org schema ──────────────────────────────────
org.person                                          ← central person entity
    ↑
    ├── crm.prospect.person_id                     （filled after conversion）
    ├── crm.student_profile.person_id              （CRM student extension）
    ├── org.guardian_assignment.student_person_id  （student side）
    ├── org.guardian_assignment.guardian_person_id （guardian side）
    ├── crm.sales_contract.student_person_id
    ├── crm.renewal_reminder.person_id
    ├── crm.churn_warning.person_id
    └── auth.user_account.person_id

── auth schema ──────────────────────────────────
auth.user_account
    └── org.institution_role_assignment            （role_code = 'sales' etc.）

── crm schema ───────────────────────────────────
crm.prospect ──→ crm.communication                 （entity_type='prospect'）
crm.prospect ──→ crm.tag_assignment                （entity_type='prospect'）
crm.prospect ──→ crm.conversion                    （deal tracking）

crm.student_profile ──→ crm.communication          （entity_type='student'）
crm.student_profile ──→ crm.tag_assignment         （entity_type='student'）
crm.student_profile ──→ crm.sales_contract
crm.student_profile ──→ crm.renewal_reminder
crm.student_profile ──→ crm.churn_warning
crm.student_profile ──→ crm.conversion

org.person（guardian person, linked via org.guardian_assignment）
    ├── crm.communication              （entity_type='guardian', entity_id = guardian's org.person.id）
    └── crm.tag_assignment             （entity_type='guardian', entity_id = guardian's org.person.id）
Note: `org.guardian_assignment` only defines the student↔guardian relationship; it is NOT the entity target for polymorphic references.

crm.marketing_activity
    ├── crm.marketing_activity_participant          （entity_type='prospect'|'student'）
    ├── crm.communication                           （entity_type='marketing_activity'）
    └── crm.tag_assignment                          （entity_type='marketing_activity'）

crm.sales_contract
    ├── crm.sales_payment
    ├── crm.renewal_reminder
    ├── crm.communication                           （entity_type='sales_contract'）
    └── crm.tag_assignment                          （entity_type='sales_contract'）

crm.communication.email_log_id ──→ email.email_log

crm.tag_assignment        polymorphic: entity_type/entity_id → prospect/student/guardian/marketing_activity/sales_contract
crm.communication         polymorphic: entity_type/entity_id → prospect/student/guardian/marketing_activity/sales_contract
crm.task                  polymorphic: related_to_type/related_to_id → prospect/student/guardian/marketing_activity/sales_contract

── email schema（extends V2_1）─────────────────
email.email_account
    └── email.email_log（via email_log.email_account_id）
        └── email.email_delivery（via delivery.email_log_id）

── messaging schema（in-app notifications only）─
messaging.notification
    └── messaging.notification_recipient

── contract schema ─────────────────────────────
contract.contract（fulfillment side）
    ←── crm.sales_contract.classing_contract_id   （one-to-one after activation）
```

Cross-schema relationship summary:

| Source Table | FK / PK | Target Table | Cardinality | Notes |
|-------------|---------|-------------|-------------|-------|
| crm.prospect | institution_id | org.education_institution | N:1 | Multi-tenant scope |
| crm.prospect | person_id | org.person | 0..1 | Filled after conversion |
| crm.student_profile | (institution_id, person_id) composite PK | org.education_institution + org.person | 1 per institution | Same person can have different profiles across institutions |
| crm.sales_contract | institution_id | org.education_institution | N:1 | Multi-tenant scope |
| crm.sales_contract | student_person_id | org.person | 1 | |
| crm.sales_contract | classing_contract_id | contract.contract | 0..1 | Linked after formal activation |
| crm.sales_payment | institution_id | org.education_institution | N:1 | Must match parent crm.sales_contract.institution_id |
| crm.sales_payment | sales_contract_id | crm.sales_contract | N | |
| crm.renewal_reminder | institution_id | org.education_institution | N:1 | Must match parent crm.sales_contract.institution_id |
| crm.renewal_reminder | sales_contract_id | crm.sales_contract | 1 | |
| crm.renewal_reminder | person_id | org.person | 1 | |
| crm.churn_warning | institution_id | org.education_institution | N:1 | Multi-tenant scope |
| crm.churn_warning | person_id | org.person | 1 | |
| crm.marketing_activity_participant | institution_id | org.education_institution | N:1 | Must match parent crm.marketing_activity.institution_id |
| crm.marketing_activity_participant | marketing_activity_id | crm.marketing_activity | 1 | |
| crm.communication | institution_id | org.education_institution | N:1 | Multi-tenant scope |
| crm.communication | email_log_id | email.email_log | 0..1 | |
| email.email_delivery | email_log_id | email.email_log | N | |
| org.guardian_assignment | student_person_id | org.person | N | |
| org.guardian_assignment | guardian_person_id | org.person | 1 | |

## Appendix C: Reference DDL Files

Existing ClassingSystem Flyway migrations（not copied here — refer to source files for full DDL）:

| Migration File | Path | Content |
|---------------|------|---------|
| `V0_2__enums.sql` | `.../db/migration/V0_2__enums.sql` | `enum.org_institution_role_enum`（principal, academic_director, finance, it_admin, marketing, operations, sales — `'sales'` added directly to CREATE TYPE）; `enum.workspace_type_enum`（academic, teacher, student, guardian, admin — replaces old academic_admin/management）; **new** `enum.platform_role_enum`（saas_operator, system_maintainer）; **new** `enum.auth_account_scope_enum`（institution, platform） |
| `V2_0__classing_schema_baseline.sql` | `.../db/migration/V2_0__classing_schema_baseline.sql` | `org.person`, `org.institution_member`, `org.guardian_assignment`, `core.resource`, `auth.user_account` + core business tables. **Note:** Legacy `CREATE TABLE prospect.prospect` has been removed from this baseline — replaced by `crm.prospect` in V2_3（pre-production rebuild, no data migration）. |
| `V2_1__email_schema_and_templates.sql` | `.../db/migration/V2_1__email_schema_and_templates.sql` | `email.email_template`, `email.email_rate_limit` |
| `V2_2__extend_email_and_messaging.sql` | New migration | `email.email_account`, `email.email_log`（含 `email_account_id` — 从 V2_1 移来，使 FK 在 CREATE TABLE 中直接引用 email_account）, `email.email_delivery`, `messaging.notification`, `messaging.notification_recipient` |
| `V2_3__crm_tables.sql` | New migration | All `crm.*` tables defined in §3.3 above |

Full DDL text for `V2_0__classing_schema_baseline.sql` is maintained at:
`D:\dev\LLYProjects\ClassingSystem\Backend\BE_Classing\src\main\resources\db\migration\V2_0__classing_schema_baseline.sql`
