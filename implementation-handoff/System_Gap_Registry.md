# System-Wide Gap Tracking & Regression Registry

**Created**: 2026-06-25  
**Purpose**: Track ALL gaps between spec and implementation that cannot be fixed immediately.  
**Usage**: Before each sprint, review this doc. Mark items as [FIXED] when resolved.

---

## CRITICAL — Multi-Tenant Data Isolation (missing `institution_id`)

| # | Entity | Status | Fix Sprint |
|---|---|---|---|
| G001 | `EmailAccountEntity` | missing `institution_id` | TBD |
| G002 | `StudentProfileEntity` | missing `institution_id` (spec requires composite PK) | TBD |
| G003 | `TaskEntity` | missing `institution_id` | TBD |
| G004 | `SalesPaymentEntity` | missing `institution_id` | TBD |
| G005 | `TagEntity` | missing `institution_id` | TBD |
| G006 | `ConversionEntity` | missing `institution_id` | TBD |
| G007 | `ConversionStageEntity` | missing `institution_id` | TBD |

---

## CRITICAL — Missing Controllers

| # | Controller | Spec | Status |
|---|---|---|---|
| G008 | `NotificationController` | `/api/admin/notifications/*` | None - entire in-app notification system missing |
| G009 | `EmailDeliveryController` | `/api/admin/email/delivery-status/*` | None - per-recipient delivery tracking missing |

---

## HIGH — Entity Field Mismatches

| # | Entity | Gap | Fix Sprint |
|---|---|---|---|
| G010 | `SalesContractEntity` | missing `type`, `items` (JSONB), `paid_amount`, `remaining_sessions`, `student_person_id` | TBD |
| G011 | `MarketingActivityEntity` | missing `timezone`, `transportation`, `organization`, `capacity` | TBD |
| G012 | `ChurnWarningEntity` | `person_id` vs `studentProfileId`, missing `auto_created`, `resolved_by` | TBD |
| G013 | `RenewalReminderEntity` | `person_id` vs `studentProfileId`, `trigger_at` vs `dueDate` type mismatch | TBD |

---

## HIGH — Email Domain

| # | Gap | Status |
|---|---|---|
| G014 | `EmailSenderService` uses hardcoded config instead of `EmailAccountEntity` | Blocks per-institution SMTP |
| G015 | `EmailLogEntity` missing `email_account_id` | Cannot link logs to accounts |
| G016 | `EmailTemplateController` wrong base path (`/api/email-templates` vs `/api/admin/email/templates`) | Route mismatch |
| G017 | `EmailLogController` permissions use `crm.communication` instead of `email.log` | Wrong permission domain |

---

## HIGH — Missing @PreAuthorize

| # | Controller | Endpoints |
|---|---|---|
| G018 | `OrgRoleController` | `listRoles()`, `assignRole()`, `revokeRole()` |
| G019 | `InstitutionController` | `listInstitutions()` |
| G020 | `MessagesController` | `inbox()`, `detail()` |
| G021 | `AuditController` | `list()` |

---

## HIGH — Email Permissions Missing in Permission Model

| # | Role | Missing |
|---|---|---|
| G022 | marketing | `email.template:read,modify` (CPE missing) |
| G023 | marketing | `email.log:read` (CPE missing) |

*Note: G022/G023 were NOT fixed in this session — only SC was updated, CPE still missing.*

---

## MEDIUM — Controller Gaps

| # | Gap | Status |
|---|---|---|
| G024 | `MarketingActivityController` uses `/activities` not `/marketing-activities` per spec | Path mismatch |
| G025 | `StudentProfileController` missing POST create, PUT update | Only GET/DELETE exist |
| G026 | `OrgMemberController` missing GET list, GET detail, PUT update, DELETE | Only POST exists |

---

## MEDIUM — Messaging / Notification

| # | Gap | Status |
|---|---|---|
| G027 | Broadcast uses separate tables instead of `messaging.notification` per spec | Schema divergence |
| G028 | `MessagesController` no institution scope filtering | Cross-tenant data leak |

---

## LOW — DTO Field Gaps

| # | Gap |
|---|---|
| G029 | `PersonResponse` missing `preferredName` |
| G030 | `InstitutionResponse` missing `secondaryPhone`, `wechatId`, `email1`, `email2`, `emailEnrollment`, `emailRecruitment` |

---

## LOW — Naming Inconsistencies

| # | Gap |
|---|---|
| G031 | `crm.renewal` vs spec `crm.renewal_reminder` |
| G032 | `crm.student` vs canonical `crm.student_profile` |
| G033 | `change_status` used instead of `complete`/`cancel` for task actions |

---

## DEFERRED — Teacher / Student / Guardian Workspaces (member_type-based)

These roles require a fundamentally different permission resolution path since they're based on `member_type`, not `institution_role`. The current `resolvePermissions()` only handles institution roles.

| # | Gap | Sprint |
|---|---|---|
| G034 | Teacher permissions (9 objects) entirely missing from SC/CPE | Sprint 9 |
| G035 | Student permissions (7 objects) entirely missing from SC/CPE | Sprint 10 |
| G036 | Guardian permissions (8 objects) entirely missing from SC/CPE | Sprint 11 |
| G037 | Guardian workspace not built in `buildAvailableWorkspaces` | Sprint 11 |

---

## DEFERRED — Academic Object Domain

| # | Gap | Sprint |
|---|---|---|
| G038 | No `academic.*` object permissions in any permission map | Sprint 8 |

---

## DEFERRED — Specification Reporting

| # | Gap | Sprint |
|---|---|---|
| G039 | `permissions_by_workspace` structure not returned — flat `permissions` map returned instead | TBD |
| G040 | `currentWorkspace` returns `"saas"` for platform roles instead of `null` per FA spec §21.4 | TBD |
| G041 | `people_admin` missing from `CrmPermissionEvaluator` (correct per spec, but risks errors) | TBD |
