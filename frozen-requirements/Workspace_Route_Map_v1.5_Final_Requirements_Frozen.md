# Workspace Route Map v1.5 — Final Requirements Frozen

**Date**: 2026-06-24  
**Status**: v1.5 Final Requirements Frozen — implementation handoff baseline

---

## 1. Login

```text
/app/login
```

---

## 2. Academic Workspace

```text
/app/academic/dashboard
/app/academic/messages/*
/app/academic/resources/students
/app/academic/resources/students/:id
/app/academic/resources/teachers
/app/academic/resources/teachers/:id
/app/academic/resources/rooms
/app/academic/resources/rooms/:id
/app/academic/classes
/app/academic/classes/:classId
/app/academic/classes/:classId/members
/app/academic/classes/:classId/baseline
/app/academic/scheduling/attempts
/app/academic/scheduling/attempts/:attemptId
/app/academic/scheduling/confirm
/app/academic/baselines
/app/academic/baselines/:baselineId
/app/academic/execution/today
/app/academic/execution/today/:sessionId
/app/academic/execution/history
/app/academic/execution/makeups
/app/academic/review/study-plan
/app/academic/review/course-changes
/app/academic/review/material-changes
/app/academic/contracts
/app/academic/contracts/:contractId
/app/academic/contracts/:contractId/settlement
/app/academic/settings/time-rules
/app/academic/settings/scheduling-strategy
/app/academic/settings/notification-strategy
/app/academic/settings/audit-log
```

Formal class sessions remain under Academic / Execution.

---

## 3. Teacher Workspace

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

Teacher messaging permissions:

```text
teacher_send_message
teacher_view_own_messages
teacher_view_own_delivery_status
```

---

## 4. Student Workspace

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

---

## 5. Guardian Workspace

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

---

## 6. Admin Workspace

```text
/app/admin/messages/*
/app/admin/org/*
/app/admin/crm/*
/app/admin/email/*
/app/admin/notifications/*
/app/admin/audit
```

---

## 7. Admin Messages

```text
/app/admin/messages/inbox
/app/admin/messages/unread
/app/admin/messages/:messageId
```

---

## 8. Admin Organization

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

---

## 9. Admin CRM

```text
/app/admin/crm/prospects
/app/admin/crm/prospects/:id
/app/admin/crm/prospects/:id/convert

/app/admin/crm/students
/app/admin/crm/students/:id
/app/admin/crm/students/:id/guardians        # Student guardian links（admin only）

/app/admin/crm/marketing-activities
/app/admin/crm/marketing-activities/:id

/app/admin/crm/tasks

/app/admin/crm/conversions
/app/admin/crm/conversions/:id
/app/admin/crm/conversion-stages

/app/admin/crm/sales-contracts
/app/admin/crm/sales-contracts/:id
/app/admin/crm/sales-contracts/:id/payments

/app/admin/crm/renewals
/app/admin/crm/churn-warnings
/app/admin/crm/communications
/app/admin/crm/tags
```

> **Guardian management boundary**: Guardian Workspace (§5) is the guardian **self-service portal**. Guardian student links for admin staff are managed within the student detail page (`/app/admin/crm/students/:id/guardians`). Guardian person master data is managed via Admin Org (`/app/admin/org/persons/:personId`). There is no standalone `/app/admin/crm/guardians` route unless a future "school-wide guardian pool" page is required.

---
## 10. Admin Email

```text
/app/admin/email/accounts
/app/admin/email/templates
/app/admin/email/logs
/app/admin/email/delivery-status
```

## 11. Admin Notifications

```text
/app/admin/notifications/inbox
/app/admin/notifications/:id
/app/admin/notifications/broadcast     （principal, operations, it_admin）
```

---

## 12. Removed Routes

Removed:

```text
/app/admin/prospects
/api/admin/prospects/**
```

Use:

```text
/app/admin/crm/prospects
/api/admin/crm/prospects/**
```

---

## 13. API Route Map

```text
/api/public/**
/api/auth/**
/api/session/context                    # GET: workspace, role, permissions（no workspace prefix）
/api/admin/org/**
/api/admin/crm/prospects/**
/api/admin/crm/students/**
/api/admin/crm/students/:id/guardians
/api/admin/crm/marketing-activities/**
/api/admin/crm/tasks/**
/api/admin/crm/conversions/**
/api/admin/crm/conversion-stages/**
/api/admin/crm/sales-contracts/**
/api/admin/crm/renewals/**
/api/admin/crm/churn-warnings/**
/api/admin/crm/communications/**
/api/admin/crm/tags/**
/api/admin/crm/tag-assignments/**
/api/admin/email/**
/api/admin/notifications/**
/api/academic/**
/api/teacher/**
/api/student/**
/api/guardian/**
```

---

## 14. Legacy Redirects

```text
/admin-login.html     → /app/login
/admin-prospect.html  → /app/admin/crm/prospects
```
