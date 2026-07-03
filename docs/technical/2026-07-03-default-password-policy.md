# 技术方案：默认密码与首登强制改密（REQ-001）

| 项目 | 内容 |
|---|---|
| 日期 | 2026-07-03 |
| 对应需求 | [REQ-001](../requirements/2026-07-03-default-password-policy.md) |
| 代码仓库 | `FlyFish-Py/mingda`（新建自定义 App，本需求为首个功能） |
| 不改动 | erpnext / frappe 核心代码零修改，全部通过 hooks 挂载 |

## 1. 方案选型

| 方案 | 说明 | 结论 |
|---|---|---|
| A. 借力系统自带"每 N 天强制改密" | 代码最少，但必须开启周期改密 | ❌ 用户明确不要周期改密 |
| B. 纯自定义（选定） | User 加自定义标记字段 + 四个挂钩点 | ✅ |

## 2. 实现设计（App：mingda）

### 2.1 自定义字段（patch 创建）

`User.mingda_must_reset_password`（Check，只读）——"须强制改密"标记。

### 2.2 挂钩点（mingda/hooks.py）

| 挂钩 | 函数 | 作用 |
|---|---|---|
| `doc_events User.before_insert` | `disable_welcome_email` | 置 `send_welcome_email=0`，阻断欢迎邮件（frappe 在 on_update 阶段检查该字段） |
| `doc_events User.after_insert` | `apply_default_password` | 底层写入默认密码（绕过强度校验）+ 置标记=1 |
| `doc_events User.before_validate` | `clear_flag_on_manual_password_change` | 管理员在用户表单手动设密码（new_password 字段）时清标记 |
| `on_session_creation` | `redirect_if_password_reset_required` | 登录成功时若有标记，用 `User._reset_password(send_email=False, password_expired=True)` 生成改密链接并写入 `response["redirect_to"]`（与 frappe 核心过期改密同一套前端流程） |
| `auth_hooks`（每请求） | `enforce_password_reset` | 兜底强制：带标记用户的 GET 页面请求一律重定向到 `/update-password`（白名单放行改密页、静态资源、改密/登出 API），防止绕过登录跳转直接访问 |
| `override_whitelisted_methods` | 包装 `frappe.core.doctype.user.user.update_password` | 核心改密成功后（会话已切换为目标用户）清除标记。覆盖两条改密路径：登录跳转的 key 链接、`/update-password` 页面用旧密码改密 |

### 2.3 关键机制依据（frappe v16 源码核对）

- 欢迎邮件：`user.py` on_update 中 `cint(self.send_welcome_email)` 为 0 即跳过
- `update_password()`（whitelisted）：改密成功后执行 `login_manager.login_as(user)`，因此包装器里 `frappe.session.user` 即目标用户，可安全清标记；改密失败（key 无效/旧密码错）时会话不变，不会误清
- 新密码强度：用户自设新密码仍走核心 `test_password_strength`，密码策略不被绕过
- 默认密码写入用 `frappe.utils.password.update_password`（底层直写哈希），不触发强度校验

### 2.4 可配置项

站点 `site_config.json` 可选配置 `"mingda_default_password": "xxx"` 覆盖默认的 `123456`。

## 3. 部署步骤

```bash
# [frappe 用户]
cd ~/frappe-bench
bench get-app https://github.com/FlyFish-Py/mingda.git
bench --site erp-test.local install-app mingda   # 自动执行 patch 建字段
bench restart
```

## 4. 测试用例（对应验收标准）

1. 新建系统用户 → 无欢迎邮件 → `123456` 可登录 → 被带到改密页
2. 改密前手动输入 `/app` 地址 → 被拦回改密页
3. 设置弱密码 → 被强度策略拒绝；设置合规密码 → 成功进入系统
4. 登出重登 → 不再要求改密
5. 新建门户用户（Website User）→ 同样生效
6. 管理员在用户表单直接设新密码 → 该用户登录不被强制改密
7. Administrator 登录 → 不受影响
