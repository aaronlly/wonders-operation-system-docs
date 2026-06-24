# ClassingSystem — 全链路手动调试指南

> **适用**: 本地 Docker Desktop + Windows PowerShell
> **流程**: SaaSOperator → 创建学校+校长 → 校长创建团队 → 全员验证

---

## 一、环境启动

### 终端 1: PostgreSQL

```powershell
docker stop classing-postgres; docker rm classing-postgres
docker volume rm db_classing_pgdata -f
docker-compose -f "D:\dev\LLYProjects\ClassingSystem\DB\docker-compose.yml" up -d
Start-Sleep -Seconds 5
docker exec classing-postgres pg_isready -U classing_prod
# → /var/run/postgresql:5432 - accepting connections
```

### 终端 2: 后端

```powershell
cd D:\dev\LLYProjects\ClassingSystem\Backend\BE_Classing
.\gradlew.bat bootJar
java -jar build\libs\classing-backend-0.0.1-SNAPSHOT.jar
# → Started BeClassingApplicationKt in X seconds
```

### 终端 3: 前端

```powershell
cd D:\dev\CRM
pnpm --filter @crm/web dev
# → http://localhost:5173
```

> 首次运行需 `pnpm install`。

---

## 二、SaaSOperator 流程（Postman / curl）

### Step 1: 初始化 SaaS 管理员

```
POST http://localhost:8080/api/admin/init
```

```powershell
Invoke-RestMethod -Uri http://localhost:8080/api/admin/init -Method Post
```

**响应**:
```json
{
  "success": true,
  "loginName": "saas_admin",
  "password": "saas_admin@2026"
}
```

### Step 2: SaaSOperator 登录

```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{"loginName": "saas_admin", "password": "saas_admin@2026"}
```

```powershell
$saas = Invoke-RestMethod -Uri http://localhost:8080/api/auth/login -Method Post `
  -ContentType "application/json" `
  -Body '{"loginName":"saas_admin","password":"saas_admin@2026"}'
$saasToken = $saas.accessToken
$saasToken
# → eyJhbGciOiJIUzI1NiJ9...
```

### Step 3: 创建校长候选人（Person）

```
POST http://localhost:8080/api/persons
Authorization: Bearer {saasToken}
Content-Type: application/json

{
  "username": "headmaster",
  "legalFullName": "Zhang Wei",
  "gender": "MALE",
  "dateOfBirth": "1985-06-15",
  "nationality": "PL",
  "primaryPhone": "600111222",
  "email": "zhang@wa.pl",
  "address": "School Street 1",
  "postalCode": "00-001",
  "countryCode": "PL",
  "idDocumentType": "passport",
  "idDocumentNumber": "Z123456789"
}
```

```powershell
$personBody = @{
    username = "headmaster"
    legalFullName = "Zhang Wei"
    gender = "MALE"
    dateOfBirth = "1985-06-15"
    nationality = "PL"
    primaryPhone = "600111222"
    email = "zhang@wa.pl"
    address = "School Street 1"
    postalCode = "00-001"
    countryCode = "PL"
    idDocumentType = "passport"
    idDocumentNumber = "Z123456789"
} | ConvertTo-Json

$person = Invoke-RestMethod -Uri http://localhost:8080/api/persons -Method Post `
  -Body $personBody -ContentType "application/json" `
  -Headers @{"Authorization"="Bearer $saasToken"}
$person.id
# → 记录这个 UUID = PRINCIPAL_PERSON_ID
# 例如: "a1b2c3d4-..."
```

### Step 4: 创建学校（同时任命校长、创建登录账号）

```
POST http://localhost:8080/api/institutions
Authorization: Bearer {saasToken}
Content-Type: application/json

{
  "name": "Wonders Academy",
  "legalName": "WA Education LLC",
  "countryCode": "PL",
  "primaryPhone": "123456789",
  "email": "school@wonders.pl",
  "principalId": "{PRINCIPAL_PERSON_ID}",
  "username": "headmaster"
}
```

```powershell
$instBody = @{
    name = "Wonders Academy"
    legalName = "WA Education LLC"
    countryCode = "PL"
    primaryPhone = "123456789"
    email = "school@wonders.pl"
    principalId = $person.id
    username = "headmaster"
} | ConvertTo-Json

$inst = Invoke-RestMethod -Uri http://localhost:8080/api/institutions -Method Post `
  -Body $instBody -ContentType "application/json" `
  -Headers @{"Authorization"="Bearer $saasToken"}
$inst
# → 返回 institution 信息，包含 institutionCode (如 EI000001)
```

**这一步内部发生了什么**：
- 创建 `org.education_institution`
- 创建 `org.institution_member`（principal → STAFF）
- 创建 `org.institution_role_assignment`（principal → PRINCIPAL）
- 创建 `auth.user_account`（loginName = "headmaster"）← **密码随机生成，通过邮件发送**
- 尝试发送凭证邮件（本地 SMTP 不可用时会静默失败）

### Step 5: 手动设置校长密码（邮件不可用时）

```
# 查校长 account ID
SELECT id FROM auth.user_account WHERE login_name = 'headmaster';
```

```powershell
docker exec -it classing-postgres psql -U classing_prod -d classing_prod `
  -c "SELECT id FROM auth.user_account WHERE login_name = 'headmaster';"
# → 记录这个 UUID = HEADMASTER_ACCOUNT_ID

# 插入 bcrypt 密码（密码 = admin）
docker exec -it classing-postgres psql -U classing_prod -d classing_prod `
  -c "INSERT INTO auth.user_password (user_account_id, password_hash, hash_algorithm, is_active) VALUES ('HEADMASTER_ACCOUNT_ID', '`$2a`$10`$9PjO4bRKYUHRqCzMjSRzZufeYtJ6WUXPo6QB8QsZ7Xon4O/3BeVGu', 'bcrypt', TRUE);"
```

> ⚠️ 把 `HEADMASTER_ACCOUNT_ID` 替换为实际的 UUID
> 如果已有一条 `password_status = 'initial'` 的记录，需要先 `UPDATE` 而不是 `INSERT`：
> ```sql
> UPDATE auth.user_password SET password_hash = '$2a$10$9PjO4bRKYUHRqCzMjSRzZufeYtJ6WUXPo6QB8QsZ7Xon4O/3BeVGu', hash_algorithm = 'bcrypt', is_active = TRUE WHERE user_account_id = 'HEADMASTER_ACCOUNT_ID';
> ```

---

## 三、校长流程（浏览器）

### Step 6: 校长登录前端

1. 浏览器打开 `http://localhost:5173`
2. 在**邮箱**框输入 `headmaster`
3. 在**密码**框输入 `admin`
4. 点击**登录**

→ 自动跳转到 `/app/admin/students`（客户管理页）
→ 左侧菜单显示全部 admin 功能

### Step 7: 校长 → 创建销售

1. 左侧菜单点击 **成员管理**（`/app/admin/org/members`）
2. 点击 **添加成员** 按钮
3. 填写：
   | 字段 | 值 |
   |------|-----|
   | 登录名 | `sales01` |
   | 初始密码 | `Sales123!` |
   | 姓名 | `Li Ming` |
   | 角色 | 销售 |
   | 电话 | `601000111` |
   | 地址 | `Test Address` |
   | 邮编 | `00-001` |
   | 国家代码 | `PL` |
   | 证件类型 | `passport` |
   | 证件号 | `SALES001` |
4. 点击确定

→ 创建成功提示

### Step 8: 校长 → 创建 people_admin

同上，角色选 **people_admin**（如果下拉中有）或直接 POST：

```powershell
$principalLogin = Invoke-RestMethod -Uri http://localhost:8080/api/auth/login -Method Post `
  -ContentType "application/json" -Body '{"loginName":"headmaster","password":"admin"}'
$principalToken = $principalLogin.accessToken
$instId = $principalLogin.user.institutionId

$body = @{
    institutionId = $instId
    loginName = "hr_admin"
    initialPassword = "HrTest123"
    roleCode = "people_admin"
    legalFullName = "HR Admin"
    dateOfBirth = "1992-03-15"
    primaryPhone = "602000222"
    address = "HR Office"
    postalCode = "00-002"
    countryCode = "PL"
    idDocumentType = "passport"
    idDocumentNumber = "HR001"
} | ConvertTo-Json

Invoke-RestMethod -Uri http://localhost:8080/api/admin/org/members -Method Post `
  -Body $body -ContentType "application/json" `
  -Headers @{"Authorization"="Bearer $principalToken"}
```

---

## 四、团队验证

### Step 9: Sales 登录（浏览器）

1. 退出校长账号（右上角退出按钮）
2. 用 `sales01` / `Sales123!` 登录
3. 验证：
   - 左侧菜单显示**线索管理、客户管理、转化、合同、任务**
   - 左侧菜单**不**显示邮箱账户、成员管理
   - 打开线索管理页 → 创建一条线索 → 列表可见

### Step 10: people_admin 登录（浏览器）

1. 用 `hr_admin` / `HrTest123` 登录
2. 验证：
   - 左侧菜单显示**成员管理、人员档案、用户账号、角色分配**
   - 左侧菜单**不**显示任何 CRM 页面
   - 打开成员管理 → 可以查看/创建成员

### Step 11: 权限边界验证（curl）

```powershell
# people_admin 尝试访问 CRM → 应返回 403
$paLogin = Invoke-RestMethod -Uri http://localhost:8080/api/auth/login -Method Post `
  -ContentType "application/json" -Body '{"loginName":"hr_admin","password":"HrTest123"}'
$paToken = $paLogin.accessToken
$instId = $paLogin.user.institutionId

Invoke-WebRequest -Uri "http://localhost:8080/api/admin/crm/prospects?institutionId=$instId" `
  -Headers @{"Authorization"="Bearer $paToken"} -SkipHttpErrorCheck | Select StatusCode
# → 403

# sales 尝试访问邮件账户 → 应返回 403
$sLogin = Invoke-RestMethod -Uri http://localhost:8080/api/auth/login -Method Post `
  -ContentType "application/json" -Body '{"loginName":"sales01","password":"Sales123!"}'
$sToken = $sLogin.accessToken

Invoke-WebRequest -Uri "http://localhost:8080/api/admin/email/accounts" `
  -Headers @{"Authorization"="Bearer $sToken"} -SkipHttpErrorCheck | Select StatusCode
# → 403
```

---

## 五、CRM 核心流程验证（浏览器）

以 `sales01` 登录：

| 操作 | 路径 | 预期 |
|------|------|------|
| 查看线索列表 | `/app/admin/crm/prospects` | 空表或无数据 |
| 新建线索 | 点击"添加线索"按钮 | 弹出表单，填写后创建成功 |
| 查看线索详情 | 点击表格行 | 进入详情页，可添加沟通记录 |
| 修改状态 | 在详情页修改状态 | new → contacted → trial_booked → ... |
| 转化线索 | 点击"转化" | 创建 student + profile |
| 查看学生 | `/app/admin/crm/students` | 新转化的学生出现 |

---

## 六、完全重置

```powershell
# 停后端
Get-Process -Name java -ErrorAction SilentlyContinue | Stop-Process -Force

# 停前端 (Ctrl+C 在 Vite 终端)

# 清数据库
docker stop classing-postgres
docker rm classing-postgres
docker volume rm db_classing_pgdata -f
```

要重新开始，从**一、环境启动**重新执行。
