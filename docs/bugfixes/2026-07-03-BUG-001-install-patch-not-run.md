# BUG-001：安装 mingda 后全站 500（自定义字段未创建）

| 项目 | 内容 |
|---|---|
| 日期 | 2026-07-03 |
| 严重级别 | 高（测试环境全站不可用） |
| 关联需求 | REQ-001 默认密码策略 |
| 修复提交 | mingda 仓库（after_install 钩子 + 防御性判断） |

## 1. 现象

测试环境执行 `bench install-app mingda` 并重启后，**所有页面（含登录页）500**。日志核心报错：

```
MySQLdb.OperationalError: (1054, "Unknown column 'mingda_must_reset_password' in 'SELECT'")
  File "apps/mingda/mingda/overrides/auth.py", line 36, in enforce_password_reset
```

## 2. 根因分析

两个问题叠加：

1. **直接原因**：`User.mingda_must_reset_password` 自定义字段没有被创建。字段创建逻辑放在了 `patches.txt` 的补丁里，但 **frappe 在全新安装 App 时不会执行补丁**——它把 patches.txt 里的所有条目直接标记为"已执行"（补丁机制只服务于已装站点的版本升级迁移）。
2. **放大原因**：`auth_hooks` 每个请求都会执行，其中的字段查询没有任何防御——字段不存在时直接抛数据库异常，把"一个字段缺失"放大成"全站不可用"。

## 3. 紧急处置（当时执行）

```bash
cd /home/frappe/frappe-bench
bench --site erp-test.local execute mingda.patches.create_password_policy_fields.execute
bench --site erp-test.local clear-cache
```

手动执行补丁函数补建字段，站点即恢复。

## 4. 永久修复

1. 字段创建逻辑移至 `mingda/install.py`，并挂到 **`after_install` 钩子**（安装时必然执行）；patches.txt 保留同一逻辑（幂等），服务于"先装 App 后加字段"的存量站点升级场景。
2. `_needs_reset()` 增加防御：先用缓存的 meta 检查字段是否存在（`frappe.get_meta("User").has_field(...)`），字段缺失时直接放行不拦截，保证任何初始化异常都不会导致全站瘫痪。

## 5. 经验教训（后续二开通用规则）

- **新装 App 的初始化（自定义字段、默认数据）一律放 `after_install`，不要只放 patch**；patch 只负责存量站点迁移。
- **每请求执行的钩子（auth_hooks 等）必须防御性编程**：任何查询失败都要有兜底，宁可功能暂时失效也不能拖垮全站。
- 自定义 App 上线正式环境前，必须先在干净的测试站点完整走一遍 install 流程验证。
