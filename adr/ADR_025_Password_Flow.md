# Password Flow Design Decision — v2.4

> **Status**: Finalized  
> **Applies to**: All role specs, UI IA Spec, OrgMemberService, Login.tsx, PasswordChange.tsx

---

## 1. Password Creation

| Scenario | Trigger | Password Source |
|----------|---------|----------------|
| New member created by principal/people_admin | OrgMemberList → `POST /api/admin/org/members` | User provides OR system auto-generates 12-char random |
| Principal created by SaaSOperator | `POST /api/institutions` | System-generated (PrincipalEmailService) |

### Auto-generation rules

```text
12 characters, mixed case letters + digits
Excludes ambiguous: 0, O, I, l
Format: ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789
```

`initialPassword` is returned in the API response ONLY when auto-generated. User-provided passwords are NOT echoed back.

---

## 2. Password Change

### First-time login

```
Login → must_change_password = true → force redirect to /app/admin/password-change
```

User MUST change password before accessing any workspace page.

### Self-service password change (any role)

```
User Menu → 修改密码 → POST /api/auth/change-password
```

Uses `authentication.principal` — any authenticated user can change their OWN password. No admin privileges required.

### Admin reset of other user's password

| Role | Can reset others' passwords |
|------|:---:|
| principal | ✅ All users in institution |
| people_admin | ✅ All users in institution |
| it_admin | ✅ Normal users only (NOT principal, people_admin, finance, academic_director, operations) |
| others | ❌ |

Endpoint: `POST /api/admin/org/users/:id/reset-password`

---

## 3. Password Storage

```
bcrypt, cost factor 10
hash_algorithm = 'bcrypt'
Stored in auth.user_password (append-only — old records deactivated via is_active = FALSE)
```

---

## 4. UI Rules

- Password is NEVER displayed after creation (except the one-time initial password response)
- `password_hash` is NEVER exposed to any UI
- `must_change_password` flag is checked on login and on every route guard
- Password change page validates confirmation match
- After password change, session is invalidated and user must re-login
