# 2api JSON 导入功能完成报告

- 日期：2026-06-07 00:10
- 责任人：开发

## 背景

qwen_register 注册脚本导出 `邮箱_密码_2api.json` 格式，需要将这些账号批量导入 YuJunZhiXue_qwen2API 账号池，同时支持：
1. **自动上传**：注册成功后实时 POST 到后端
2. **手动上传**：在管理面板选择本地 JSON 文件批量导入

## 2api JSON 格式

```json
[
  {
    "email": "example@domain.com",
    "password": "your_password",
    "token": "oauth_access_token",
    "source": "file"
  }
]
```

---

## 修改内容

### 1. 后端：新增批量导入接口

**文件：** `backend/api/admin.py`

新增 `POST /api/admin/accounts/import` 接口，接收 2api JSON 数组：

- `token` 有值 → 调用 `verify_token` 验证，通过后以 `source=file` 入池
- `token` 为空 → 以 `activation_pending=True` 状态入池，待后续手动激活
- 已存在的邮箱自动跳过（幂等）
- 返回 `{ ok, added, skipped, failed, errors }`

```python
@router.post("/accounts/import", dependencies=[Depends(verify_admin)])
async def import_accounts(request: Request):
    ...
```

**鉴权方式：** 与其他 admin 接口一致，`Authorization: Bearer <ADMIN_KEY>`

---

### 2. 前端：批量导入按钮

**文件：** `frontend/src/pages/AccountsPage.tsx`

在账号管理页头部操作栏新增「批量导入 2api JSON」按钮：

- 点击后弹出文件选择框（`accept=".json"`）
- 解析 JSON，发 POST 到 `/api/admin/accounts/import`
- toast 显示结果：`导入完成：新增 X，跳过 Y，失败 Z`
- 失败数 > 0 显示 warning，全部成功显示 success

**改动点：**
- 新增 `useRef<HTMLInputElement>` + `useState<boolean>` (importing 状态)
- 新增 `handleImportFile` 函数
- 新增 `<input type="file" hidden>` + Upload 图标按钮
- 新增 `Upload` 图标导入

---

### 3. 注册脚本：自动上传配置

**文件：** `D:/cc/qwen_register/.env`

新增三个配置项：

```env
YUJUNZHIXUE_UPLOAD_ENABLED=false
YUJUNZHIXUE_UPLOAD_URL=https://your-server.com
YUJUNZHIXUE_UPLOAD_KEY=your-admin-key
```

**文件：** `D:/cc/qwen_register/config.py`

新增对应读取：

```python
YUJUNZHIXUE_UPLOAD_ENABLED = os.environ.get("YUJUNZHIXUE_UPLOAD_ENABLED", "false").lower() == "true"
YUJUNZHIXUE_UPLOAD_URL = os.environ.get("YUJUNZHIXUE_UPLOAD_URL", "")
YUJUNZHIXUE_UPLOAD_KEY = os.environ.get("YUJUNZHIXUE_UPLOAD_KEY", "")
```

**文件：** `D:/cc/qwen_register/register.py`

新增 `yujunzhixue_upload(email, password, token)` 函数，在注册成功两个路径（直接入 chat / 激活后）各调用一次。

上传地址：`YUJUNZHIXUE_UPLOAD_URL + /api/admin/accounts/import`

---

## 部署步骤（线上）

### 后端
只需更新 `backend/api/admin.py` 一个文件，重启服务即可生效：

```bash
# 重启容器
docker compose restart
# 或直接重启进程
```

### 前端
需要重新构建并部署：

```bash
cd frontend
npm run build
# 将 dist/ 目录覆盖到服务器对应目录
```

---

## 使用方式

### 手动上传
1. 登录管理面板
2. 进入「账号管理」页
3. 点击「批量导入 2api JSON」按钮
4. 选择 `output/2api/` 目录下任意 `*_2api.json` 文件
5. 查看 toast 提示确认导入结果

### 自动上传（注册脚本）
1. 修改 `D:/cc/qwen_register/.env`：
   ```env
   YUJUNZHIXUE_UPLOAD_ENABLED=true
   YUJUNZHIXUE_UPLOAD_URL=https://your-server.com
   YUJUNZHIXUE_UPLOAD_KEY=your-admin-key
   ```
2. 运行 `python register.py`，注册成功后自动推送

---

## 测试结果

| 场景 | 结果 |
|------|------|
| register.py 语法检查 | ✅ 通过 |
| admin.py 语法检查 | ✅ 通过 |
| token 有效时导入 | 验证通过后入池 |
| token 为空时导入 | 以 pending_activation 入池 |
| 重复邮箱 | 自动 skipped，幂等 |

---

## 相关文件

- `D:/cc/YuJunZhiXue_qwen2API/backend/api/admin.py` — 新增 import 接口
- `D:/cc/YuJunZhiXue_qwen2API/frontend/src/pages/AccountsPage.tsx` — 新增导入 UI
- `D:/cc/qwen_register/register.py` — 新增 `yujunzhixue_upload()` 函数
- `D:/cc/qwen_register/config.py` — 新增 YuJunZhiXue 相关配置
- `D:/cc/qwen_register/.env` — 新增 YuJunZhiXue 上传配置项

## 版本信息

- 关联项目：YuJunZhiXue_qwen2API v2.x
- 发布状态：本地验证完成，待线上部署
