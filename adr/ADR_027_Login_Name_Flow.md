# Login Name Flow — v2.4 (ADR-027)

> **Status**: Finalized
> **Applies to**: OrgMemberService, AuthController, OrgMemberList, DashboardLayout, PasswordChange

---

## 1. Login Name Generation

### Automatic Generation

```
Format: {countryCode}{idDocumentNumber}
Example: PLZ123456, CNP999888
```

Login name is NOT user-specified. It is derived from the person's country code and ID document number, both of which are mandatory fields on the member creation form.

### Preview

After filling country code + ID document number, the form shows a real-time preview:

```
── 登录名预览: PLZ123456 ──
```

---

## 2. Login Name Modification

### Rule

| Condition | Value |
|-----------|-------|
| Who can modify | Self only (`PUT /api/auth/login-name`) |
| How many times | **Exactly once** |
| How to detect "already changed" | Current login name does NOT match `{countryCode}{idDocumentNumber}` pattern |
| Duplicate check | Same institution, unique `login_name` |
| After modification | Must re-login |

### Backend Endpoint

```
PUT /api/auth/login-name
Body: { "newLoginName": "zhangwei" }
```

Backend checks:
1. `login_name` matches generation pattern? If no → 403 "已经修改过"
2. `newLoginName` unique in same institution?
3. Update `login_name` field in `auth.user_account`
4. Invalidate all active refresh tokens → force re-login

---

## 3. First-Time Login Prompt

After first login (`must_change_password = true`), user is forced to change password. After password change succeeds, a modal appears:

```
┌──────────────────────────────────────────────┐
│           重要提示                            │
│                                              │
│  您当前的登录名为: PLZ123456                  │
│  此登录名由系统自动生成（国家代码+证件号）。    │
│                                              │
│  ⚠ 您只有一次机会修改登录名。                │
│  修改后请务必牢记，并与密码一起妥善保存。      │
│                                              │
│  [ 去修改登录名 ]    [ 暂不修改，进入系统 ]    │
└──────────────────────────────────────────────┘
```

---

## 4. Modification UI

Accessible via Top Bar → User Menu → "修改登录名" (only visible when `login_name` matches generation pattern).

```
┌──────────────────────────────────┐
│  修改登录名                        │
│                                  │
│  ⚠ 仅可修改一次，请谨慎操作。      │
│  请务必牢记新登录名和密码。          │
│                                  │
│  当前登录名: PLZ123456            │
│  新登录名:   [_______________]    │
│                                  │
│  [确认修改]  [取消]                │
└──────────────────────────────────┘
```

---

## 5. Member Creation Form

### Field Order

```
1. memberType (员工/教师/学生)
2. countryCode* + idDocumentNumber* → loginName preview
3. familyName + givenName + preferredName (given_name / family_name fields)
4. legalFullName*
5. gender* + dateOfBirth*
6. nationality* + religion
7. primaryPhone* + email + wechatId
8. address* + postalCode*
9. idDocumentType*
10. roleCode (only for staff)
```

### Batch Mode

Table layout: countryCode, idDocumentNumber, loginName (read-only preview), memberType, familyName, givenName, preferredName, legalFullName, gender, primaryPhone, dateOfBirth, email.

Each row → one member. Submit all at once → `POST /api/admin/org/members/batch`.

---

## 6. Three-Language Support

Locale files at `src/locales/{zh,en,pl}.json`:

```json
{
  "member": {
    "create": "创建新成员" / "Create Member" / "Utwórz członka",
    "batch": "批量创建" / "Batch Create" / "Tworzenie wsadowe",
    "countryCode": "国家代码" / "Country Code" / "Kod kraju",
    "idDocumentNumber": "证件号" / "ID Document No." / "Nr dokumentu",
    "loginNamePreview": "登录名预览" / "Login Name Preview" / "Podgląd nazwy logowania",
    ...
  }
}
```

Language switcher in User Menu: 中文 | English | Polski.
Selection saved to `localStorage`, instant refresh via React Context.
