# 项目文档索引

本目录存放 ERPNext 二次开发项目的全部过程文档。**约定：所有需求、技术方案、Bug 修复都必须留档**，随代码一起提交。

## 目录结构

| 目录 | 用途 | 状态 |
|---|---|---|
| `deployment/` | 环境与部署文档 | ✅ 使用中 |
| `requirements/` | 需求文档（每个需求一份） | ✅ 使用中 |
| `technical/` | 技术方案设计 | ✅ 使用中 |
| `bugfixes/` | Bug 修复记录（每个 Bug 一份） | ✅ 使用中 |

> 二开代码仓库：定制逻辑统一放在自定义应用 [FlyFish-Py/mingda](https://github.com/FlyFish-Py/mingda)，不修改 erpnext/frappe 核心。

## 文档列表

### 部署

- [测试环境部署文档](deployment/test-environment-setup.md) — 2026-07-02，Debian 13 / frappe v16 / ERPNext v16.26（fork version-16 分支），含完整步骤与 10 条踩坑记录

### 需求

- [REQ-001 新账号默认密码与首登强制改密](requirements/2026-07-03-default-password-policy.md) — 2026-07-03，全员默认密码 123456、不发欢迎邮件、首次登录强制改密

### 技术方案

- [补全 ERPNext 中文翻译](technical/2026-07-02-complete-zh-translations.md) — 2026-07-02，zh.po 覆盖率 84.5% → 100%（1544 条），含术语约定与部署步骤
- [REQ-001 默认密码策略实现](technical/2026-07-03-default-password-policy.md) — 2026-07-03，mingda 应用首个功能：hooks 挂载点设计、核心机制核对、测试用例
- [模块名补翻与"明达翻译通道"](technical/2026-07-03-mingda-translation-channel.md) — 2026-07-03，确立框架层翻译修补的标准通道（mingda 自带 zh.po 覆盖，无需 fork frappe）

### Bug 修复

- [BUG-001 安装 mingda 后全站 500](bugfixes/2026-07-03-BUG-001-install-patch-not-run.md) — 2026-07-03，全新安装不执行 patches 导致字段缺失 + auth 钩子无防御；含两条后续二开通用规则
- [BUG-002 登录后跳转报 301 未捕获异常](bugfixes/2026-07-03-BUG-002-redirect-uncaught-exception.md) — 2026-07-03，auth 阶段重定向必须用 werkzeug HTTPException，frappe.Redirect 仅限页面渲染上下文
