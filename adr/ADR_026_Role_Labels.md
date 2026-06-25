# Role Label Mapping — v2.4

> **Status**: Finalized  
> **Applies to**: All frontend components (role dropdowns, tags, user menu, sidebar)

---

## 1. Principle

Internal role codes (snake_case, English) are used for API calls and permissions.  
Customer-facing UI always displays Chinese labels.

---

## 2. Mapping Table

| Internal Code | Display Label (zh) | Color |
|--------------|-------------------|-------|
| `principal` | 校长 | `red` |
| `people_admin` | 人事管理员 | `purple` |
| `academic_director` | 教务长 | `geekblue` |
| `operations` | 运营 | `blue` |
| `sales` | 销售 | `green` |
| `marketing` | 市场 | `orange` |
| `finance` | 财务 | `gold` |
| `it_admin` | IT管理员 | `cyan` |

---

## 3. Implementation

```typescript
// src/api/roleLabels.ts
export const ROLE_LABELS: Record<string, string> = {
  principal: '校长',
  people_admin: '人事管理员',
  academic_director: '教务长',
  operations: '运营',
  sales: '销售',
  marketing: '市场',
  finance: '财务',
  it_admin: 'IT管理员',
}

export const ROLE_OPTIONS = Object.entries(ROLE_LABELS).map(([value, label]) => ({ value, label }))
```

---

## 4. Usage

- `OrgMemberList.tsx` — role dropdown
- `OrgRoleList.tsx` — role tags + assign dropdown
- `DashboardLayout.tsx` — user menu role display
- `Login.tsx` — role-based default page routing
