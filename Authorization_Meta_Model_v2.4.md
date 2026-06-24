# ClassingSystem — Authorization Meta Model v2.4

> **Purpose**: Unified authorization architecture binding Identity → Role → Scope → Permission → Guard → API → UI.
> **Audience**: Backend, Frontend, DB, QA.
> **Status**: v2.4-aligned.

---

## 1. Full Authorization Chain

```text
Identity
  │  who is this person?  (org.person, auth.user_account)
  ↓
Authentication
  │  are they logged in?  (JWT, session context)
  ↓
Workspace Binding
  │  which workspaces can they enter?  (auth.user_workspace_binding)
  ↓
Institution Role (or Member Type)
  │  what role do they have in this school?  (org.institution_role_assignment OR org.institution_member.member_type)
  ↓
Resource Scope
  │  what data can they see?  (SELF | ASSIGNED | CAMPUS | INSTITUTION | PLATFORM)
  ↓
Permission Object × Action
  │  what exactly can they do?  (crm.prospect:read, org.person:add, ...)
  ↓
Guard (@PreAuthorize / Backend Service Rules)
  │  is this request allowed?  (scope + permission + institution check)
  ↓
API Response
  │  data filtered by scope
  ↓
UI Visibility
  │  sidebar menu, button state, route guard (from session context permissions_by_workspace)
```

---

## 2. Identity

Identity answers "who is this person?"

| Source | Table | Key |
|--------|-------|-----|
| Person record | `org.person` | person_id |
| Login account | `auth.user_account` | person_id → person |
| Platform scope | `auth.user_account.account_scope` | `institution` or `platform` |

Platform accounts (`account_scope = 'platform'`) have no `institution_id`. They enter SaaSOperator, not any workspace.

---

## 3. Authentication

| Step | Endpoint | Output |
|------|----------|--------|
| Login | `POST /api/auth/login` | JWT accessToken + refreshToken |
| Session | `GET /api/session/context` | roles, workspaces, permissions, scope |

---

## 4. Workspace Binding

Which workspaces a user can enter.

| Binding Type | Table | Used By |
|-------------|-------|---------|
| Institution role | `org.institution_role_assignment` | admin roles (principal, people_admin, etc.) |
| Member type | `org.institution_member` | teacher, student |
| Guardian link | `org.guardian_assignment` | guardian |
| Platform role | `auth.platform_role_assignment` | saas_operator, system_maintainer |
| Workspace binding | `auth.user_workspace_binding` | bridges member_type / guardian to workspace |

### How workspace binding is determined

```text
institution_role_assignment → admin
institution_role_assignment (academic_director) → academic + admin(limited)
member_type = 'teacher' → teacher
member_type = 'student' → student
guardian_assignment → guardian
platform_role_assignment → SaaSOperator (not a workspace)
```

---

## 5. Role Source

There are TWO distinct role sources. They are NOT interchangeable.

### 5.1 Institution Role (org.institution_role_assignment)

School-level management roles. Grant access to Admin workspace.

```text
principal, people_admin, academic_director,
finance, it_admin, marketing, operations, sales
```

### 5.2 Member Type (org.institution_member.member_type)

Teaching/learning roles. Grant access to Teacher/Student workspaces. NOT roles.

```text
teacher → Teacher workspace
student → Student workspace
staff  → Admin workspace via institution role
```

### Critical distinction

```text
teacher is NOT a role. It is a member_type.
student is NOT a role. It is a member_type.
Only institution_role_assignment entries are roles.
```

---

## 6. Resource Scope

| Level | Definition | Used By |
|-------|-----------|---------|
| **SELF** | Current user's own data only | student |
| **ASSIGNED** | Data explicitly assigned to current user | sales, teacher, guardian |
| **CAMPUS** | Data within current user's campus(es) | future |
| **INSTITUTION** | All data within current user's institution | principal, operations, people_admin, finance, it_admin, academic_director, marketing |
| **PLATFORM** | Cross-institution, platform-level access | saas_operator, system_maintainer |

### Scope per Role

| Role | scope_level | scope_source | scope_filter |
|------|------------|-------------|-------------|
| student | SELF | current_user.person_id | person_id = self |
| guardian | ASSIGNED | org.guardian_assignment | valid_from ≤ now ≤ valid_to |
| teacher | ASSIGNED | academic.class_member via core.resource | resource.person_id = self + member_role + left_at |
| sales | ASSIGNED | crm.prospect.assigned_to | assigned_to = self |
| marketing | INSTITUTION | institution_role_assignment | institution_id (campaign-scoped for email) |
| operations | INSTITUTION | institution_role_assignment | institution_id (excl. HR/technical) |
| finance | INSTITUTION | institution_role_assignment | institution_id |
| people_admin | INSTITUTION | institution_role_assignment | institution_id |
| academic_director | INSTITUTION | institution_role_assignment | institution_id |
| it_admin | INSTITUTION | institution_role_assignment | institution_id |
| principal | INSTITUTION | institution_role_assignment | institution_id |
| saas_operator | PLATFORM | platform_role_assignment | cross-institution |
| system_maintainer | PLATFORM | platform_role_assignment | cross-institution |

---

## 7. Permission Object × Action

Canonical permission tokens defined in Role Permission Matrix.

Format: `object.action`

```text
crm.prospect:read
org.person:add
email.log:read
notification.broadcast:send
```

Permissions are granted to roles, not to users directly.
Session context returns `permissions_by_workspace` as `{ object: [actions] }`.

---

## 8. Guard

### Backend

```text
@PreAuthorize("@crmSecurity.hasPermission('crm.prospect', 'read')")
+ scope filter (institution_id = current_user.institution_id)
+ data scope (assigned_to = current_user.person_id for sales)
```

### Frontend

```text
route guard:  hasWorkspace(workspace) → hasPermission(object, action)
menu visible:  permissions_by_workspace[workspace][object].includes(action)
button state:  same logic
```

---

## 9. API Response Filtering

Every API response MUST be filtered by:

1. **Institution scope** — `WHERE institution_id = current_user.institution_id`
2. **Data scope** — `WHERE assigned_to = current_user.person_id` (for ASSIGNED roles)
3. **Permission check** — caller has `object:action`

Server-side only. Client never sends scope filters.

---

## 10. UI Visibility

| Layer | Source | Logic |
|-------|--------|-------|
| Sidebar menu | `permissions_by_workspace` from session context | show group if ≥1 child visible |
| Button enable | same | disable if missing action |
| Route guard | `hasPermission` | redirect if denied |
| Field visibility | `permissions` + `data_scope` | hide sensitive fields |

---

## 11. Class Member Lifecycle

```text
joined_at ≤ now()
AND (left_at IS NULL OR left_at > now())
```

This supports future-dated teacher/student assignments (pre-enrollment) without prematurely granting access.

---

## 12. Messaging Server-Side Rule

```text
All message target resolution MUST be server-side.
Client may request logical targets only: own_class, linked_student_teacher, school_admin.
Client MUST NOT submit arbitrary person_id recipient lists.
Backend resolves targets from session context + data scope.
```
