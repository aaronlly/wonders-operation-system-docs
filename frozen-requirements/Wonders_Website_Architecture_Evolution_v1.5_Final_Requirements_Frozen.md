# Wonders Website Architecture Evolution v1.5 — Final Requirements Frozen

**Date**: 2026-06-24  
**Status**: v1.5 Final Requirements Frozen — implementation handoff baseline

---

## 1. Purpose

This document defines the evolution of the Wonders Academy public website, legacy admin pages, and unified business platform.

It preserves the established architectural decision:

```text
Public Website（static）
+
Unified Business Platform（React SPA）
+
Spring Boot API
+
PostgreSQL
```

The public website and internal business platform must remain separated by responsibility.

---

## 2. Current State

Current production/static website contains:

```text
/
 /enrollment-2027/
 /summer-camp/
 /pages/*
 /admin-login.html
 /admin-prospect.html
```

Current legacy admin flow:

```text
/admin-login.html
    ↓
/admin-prospect.html
```

The legacy prospect page is not commercialized as a stable production back-office. It is replaced by the CRM module in the target architecture.

---

## 3. Target Website Structure

```text
wondersacademy.pl
│
├─ /                              Public website
├─ /enrollment-2027/              Enrollment pages
├─ /summer-camp/                  Summer camp pages
├─ /pages/                        Other static pages
│
├─ /app/                          React SPA business platform
│   ├─ login
│   ├─ academic
│   ├─ teacher
│   ├─ student
│   ├─ guardian
│   └─ admin
│       ├─ messages
│       ├─ org
│       ├─ crm
│       ├─ email
│       ├─ notifications
│       └─ audit
│
├─ /admin-login.html              Legacy, to be deprecated
├─ /admin-prospect.html           Legacy, to be deprecated
│
└─ /api/
    ├─ public
    ├─ auth
    ├─ session/context          # Workspace + role + permissions
    ├─ admin
    ├─ academic
    ├─ teacher
    ├─ student
    └─ guardian
```

---

## 4. Public Website Boundary

The public website remains static and is responsible for:

- brand presentation
- course introduction
- enrollment conversion
- advertising landing pages
- SEO
- public forms

The public website must not contain internal business workflow pages.

Public forms submit into:

```text
/api/public/**
```

For CRM lead submission, public APIs write to:

```text
crm.prospect          ← after V2_3 migration; replaces legacy prospect.prospect
```

> **Migration note:** The existing `prospect.prospect` table（V2_0）is deprecated and replaced by `crm.prospect`（V2_3）. Because the system is pre-production, no data migration is needed. The public website form API must be redirected to `crm.prospect` after V2_3 deployment. Legacy `/admin-prospect.html` page redirects to `/app/admin/crm/prospects`.

#### Public Form `institution_id` Resolution

Since public forms are anonymous (no authenticated user), `crm.prospect.institution_id` must be resolved server-side:

- **Single-school deployment** (Wonders Academy): default `institution_code = WONDERS_ACADEMY`, resolve via `SELECT id FROM org.education_institution WHERE code = :institutionCode`.
- **Multi-school deployment** (future): `institution_code` resolved from host/domain, landing page configuration, or explicit `public_form_config` table.

**API contract**:
```text
POST /api/public/prospects
  → resolve public_form_config.institution_code
  → lookup org.education_institution.id
  → insert crm.prospect(institution_id = resolved id)
```

---

## 5. Unified Login Entry

The homepage admin button must become a general login entry:

```text
Login → /app/login
```

Login flow:

```text
/app/login
  ↓
POST /api/auth/login
  ↓
GET /api/session/context
  ↓
route to workspace
```

Default landing rule:

1. If `last_used_workspace` exists and is still allowed, use it.
2. Otherwise route by available workspace and role.
3. For CRM sales/marketing users, route to:

```text
/app/admin/crm/prospects
```

---

## 6. Workspaces

The unified platform keeps five workspaces:

```text
academic
teacher
student
guardian
admin
```

Guardian identity is resolved through `org.guardian_assignment`, not through institution roles.

No separate sales workspace is added at this stage.

Sales is a role under Admin:

```text
org.institution_role_assignment.role_code = 'sales'
```

---

## 7. Admin Workspace Domains

Admin contains the following domains:

```text
/app/admin/messages/*      Personal inbox
/app/admin/org/*           Organization and permissions
/app/admin/crm/*           CRM business domain
/app/admin/email/*         Email infrastructure（accounts, templates, logs, delivery）
/app/admin/notifications/* In-app notifications / broadcast
/app/admin/audit           Audit
```

Important distinction:

```text
/app/admin/messages/*
```
is a user-facing inbox.

```text
/app/admin/email/*
```
is infrastructure and operations for SMTP accounts, templates, send logs, and delivery status.

```text
/app/admin/notifications/*
```
is in-app notification / broadcast (messaging schema — separate from email infrastructure).

---

## 8. CRM Target Entry

The unique prospect route is:

```text
/app/admin/crm/prospects
```

Removed:

```text
/app/admin/prospects
/api/admin/prospects/**
```

Target CRM canonical prospect table（replaces old `prospect.prospect`）:

```text
crm.prospect
```

---

## 9. Marketing Activity Boundary

CRM marketing activities are represented by:

```text
crm.marketing_activity
```

This domain covers:

- campaigns
- open days
- trial classes
- recruitment lectures
- parent information sessions
- conversion events

It does not cover:

- formal academic class sessions
- timetable sessions
- attendance-bearing lessons
- academic execution records

Those belong to Academic / Execution domains.

---

## 10. Email & Notification Evolution

Two horizontal layers:

### 10.1 Email Infrastructure（email schema）

Extends existing `email.email_template` / `email.email_rate_limit`（V2_1）and newly created `email.email_log`（V2_2, with `email_account_id` FK to `email.email_account`）with:

- `email.email_account`（SMTP configuration）
- `email.email_delivery`（per-recipient delivery tracking）

```text
/app/admin/email/*
/api/admin/email/**
```

Consumed by:
- CRM follow-up communications
- marketing campaign emails
- teacher-to-parent emails
- academic notices
- auth password reset

### 10.2 In-App Notifications（messaging schema）

Standalone `messaging` schema for in-app broadcast:

```text
/app/admin/notifications/*
/api/admin/notifications/**
```

Consumed by:
- task assignment alerts
- renewal reminders
- churn warnings
- system announcements

Teacher notification permissions:

```text
teacher_send_message
teacher_view_own_messages
teacher_view_own_delivery_status
```

---

## 11. Legacy Decommissioning

Temporary legacy pages:

```text
/admin-login.html
/admin-prospect.html
```

Final redirects:

```text
/admin-login.html     → /app/login
/admin-prospect.html  → /app/admin/crm/prospects
```

---

## 12. Deployment Routing

Recommended Nginx routing:

```text
/                       → static website
/app/*                  → React SPA history fallback
/api/*                  → Spring Boot API
/admin-login.html       → static legacy page until deprecated
/admin-prospect.html    → static legacy page until deprecated
```

After deprecation:

```text
/admin-login.html       → 302/301 /app/login
/admin-prospect.html    → 302/301 /app/admin/crm/prospects
```

---

## 13. Development Priority

1. `/app/login`
2. `GET /api/session/context`
3. SPA shell and Admin layout
4. `/app/admin/crm/prospects`
5. CRM core modules
6. Email infrastructure and notification module
7. Legacy removal
