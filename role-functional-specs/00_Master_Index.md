# ClassingSystem Role Function Specifications v2.0

> **Status**: Draft — Based on v1.5 Architecture Freeze
> **Purpose**: Individual role-level specs for frontend, backend, and DB implementation teams
> **Template**: All role specs follow the same 13-section template defined below.

---

## 1. Document Package Map

| # | Role | Spec File | Priority | Status |
|---|------|-----------|:---:|:---:|
| 1 | principal | *(covered in v1.5 Initial School Setup + Role Matrix)* | — | ✅ v1.5 |
| 2 | **people_admin** | `01_people_admin.md` | P0 | ✅ v1.0 |
| 3 | operations | `02_operations.md` | P0 | ✅ v1.0 |
| 4 | academic_director | `03_academic_director.md` | P0 | ✅ v1.0 |
| 5 | finance | `04_finance.md` | P1 | ✅ v1.0 |
| 6 | it_admin | `05_it_admin.md` | P1 | ✅ v1.0 |
| 7 | sales | `06_sales.md` | P1 | ✅ v1.0 |
| 8 | marketing | `07_marketing.md` | P1 | ✅ v1.0 |
| 9 | teacher | `08_teacher.md` | P0 | ✅ v1.0 |
| 10 | student | `09_student.md` | P0 | ✅ v1.0 |
| 11 | guardian | `10_guardian.md` | P0 | ✅ v1.0 |

### Priority Legend
- **P0**: Core school operations — unblock teacher/student/guardian end-user flows
- **P1**: School management — support org governance and CRM operations

---

## 2. Delivery Order (Business Dependency)

```
Group 1 — School Internal Management (admin workspace)
  1. people_admin    ← HR / personnel
  2. operations      ← daily operations
  3. academic_director ← academic governance
  4. finance         ← financial oversight
  5. it_admin        ← technical administration

Group 2 — CRM Operations (admin workspace)
  6. sales           ← prospect → conversion → contract
  7. marketing       ← leads, activities, campaigns

Group 3 — Teaching Execution (teacher workspace)
  8. teacher         ← classes, homework, attendance

Group 4 — External Users (student/guardian workspaces)
  9. student         ← self-service learning
  10. guardian       ← linked student monitoring
```

**Rationale**: Teacher / student / guardian depend on Academic, Class, Schedule, Contract, Homework, Attendance, Message, Payment objects that must exist before end-user workspaces can be delivered.

---

## 3. Resource Scope Model

Every role MUST declare its data scope level, source, and filter. This unifies all scope descriptions across role specs.

| Level | Definition | Used By |
|-------|-----------|---------|
| **SELF** | Current user's own data only | student |
| **ASSIGNED** | Data explicitly assigned to current user | sales, teacher, guardian |
| **CAMPUS** | Data within current user's campus(es) | future: campus-level roles |
| **INSTITUTION** | All data within current user's institution | principal, operations, people_admin, finance, it_admin, academic_director, marketing |
| **PLATFORM** | Cross-institution, platform-level access | saas_operator, system_maintainer |

### Scope Mapping Table

| Role | scope_level | scope_source | scope_filter |
|------|------------|-------------|-------------|
| student | SELF | `current_user.person_id` | `person_id = self` |
| guardian | ASSIGNED | `org.guardian_assignment` valid range | `guardian_person_id = self AND valid_from <= now() AND (valid_to IS NULL OR valid_to >= now())` |
| teacher | ASSIGNED | `academic.class_member` via `core.resource` | `resource.person_id = self AND member_role IN ('teacher','assistant') AND left_at IS NULL` |
| sales | ASSIGNED | `crm.prospect.assigned_to` | `assigned_to = self OR config: all` |
| marketing | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` |
| operations | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` (excludes HR privacy & technical data) |
| finance | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` |
| people_admin | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` |
| academic_director | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` |
| it_admin | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` |
| principal | INSTITUTION | `org.institution_role_assignment` | `institution_id = self.institution_id` |
| saas_operator | PLATFORM | `auth.platform_role_assignment` | cross-institution |
| system_maintainer | PLATFORM | `auth.platform_role_assignment` | cross-institution |

---

## 4. Role Spec Template (13 Sections)

Every role spec MUST contain all 13 sections (including Resource Scope declaration):

| # | Section | Description |
|---|---------|-------------|
| **1** | Role Positioning | What this role is, what it is NOT, where it fits in the org |
| **2** | Workspace Access | Which workspaces this role can enter |
| **3** | Default Landing Page | First page after login |
| **4** | Sidebar Menu | All menu items visible to this role |
| **5** | Core User Stories | "As a [role], I want to [action] so that [outcome]" |
| **6** | Page-Level Functions | What each page does, what controls are shown, what buttons are enabled |
| **7** | API Requirements | Every API endpoint this role calls, grouped by domain |
| **8** | Backend Service Rules | Business logic rules enforced server-side |
| **9** | Database Read / Write Objects | Tables read and written by this role's operations |
| **10** | Permission Object / Action Mapping | Canonical `object.action` token matrix |
| **11** | Notification / Email / Message Interaction | What triggers what notification to whom |
| **12** | Acceptance Tests | Verifiable test scenarios for QA |

### Section 13 — Out of Scope (mandatory)

What this role explicitly CANNOT do. Critical for preventing scope creep.

---

## 4. v1.5 Reference Documents

These are the frozen architecture baseline. All role specs are derived from these:

| Doc | Version | Status |
|-----|---------|:---:|
| `Wonders_Academy_Role_Permission_Matrix_v1.0.md` | v1.5 | Frozen |
| `Workspace_Route_Map_v1.5_Final_Requirements_Frozen.md` | v1.5 | Frozen |
| `CRM_Integration_Architecture_v1.5_Final_Requirements_Frozen.md` | v1.5 | Frozen |
| `Frontend_Architecture_v1.5_Final_Requirements_Frozen.md` | v1.5 | Frozen |
| `Initial_School_Setup_and_Principal_Administration_Flow_v1.0.md` | v1.5 | Frozen |
| `Development_Plan_v3.0.md` | v1.5 | Frozen |

---

## 5. Implementation Hand-off

Each role spec is designed to be handed to:

| Team | Uses Sections | Delivers |
|------|--------------|----------|
| **Frontend** | §2–§6, §11 | Pages, sidebar, components, notification UI |
| **Backend** | §7–§10 | Controllers, services, @PreAuthorize, repositories |
| **Database** | §9 | Migration scripts, seed data, QA gates |
| **QA** | §5, §12 | Test cases, acceptance criteria |

---

## 6. Version History

| Version | Date | Change |
|---------|------|--------|
| v2.0 | 2026-06-24 | All 10 role specs drafted (01-10) |
