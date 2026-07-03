# 技术方案：模块名补翻与"明达翻译通道"

| 项目 | 内容 |
|---|---|
| 日期 | 2026-07-03 |
| 类型 | 本地化机制建设 |
| 改动 | erpnext fork `zh.po` 追加 4 条；mingda 新增 `locale/zh.po`（6 条）|

## 1. 背景

模块名在界面（角色"模块集合"、用户"允许的模块"等）显示英文。盘点 34 个模块名：24 个已有翻译，10 个真缺口——其中 4 个归属 erpnext、5 个归属 frappe 框架、1 个是 mingda 自身。EDI 经确认保留英文不翻。

> 注意：「构建 → 模块定义」列表里的模块名称列是**系统标识符**（绑定应用代码目录），永远显示英文，不属于翻译范畴。

## 2. 关键机制：明达翻译通道

frappe 运行时**合并所有已安装应用的翻译词库**，按应用安装顺序后者生效——mingda 最后安装，优先级最高。因此：

**框架层（frappe/hrms）缺失或错误的中文词条，统一放进 `mingda/locale/zh.po` 补充/覆盖，无需 fork frappe。**

这条通道自本次起成为项目标准做法，适用场景：

- frappe 框架未翻译的词条（本次的 Core/Desk/Geo/Printing/Automation；后续的登录页 "Forgot Password"、邮箱账户 "Outgoing" 标签页等）
- frappe 已有但**翻译错误**的词条（如 Login ID 被错译为"备用邮箱"）——同名 msgid 直接覆盖
- mingda 自身功能的词条

各应用词条归属原则：erpnext 的词条改 fork 的 `erpnext/locale/zh.po`；frappe/hrms 的词条走 mingda 通道（这两个应用我们不 fork）。

## 3. 本次改动清单

| 词条 | 译文 | 位置 |
|---|---|---|
| Utilities | 实用工具 | erpnext zh.po（追加，含维护注释标记） |
| ERPNext Integrations | ERPNext 集成 | 同上 |
| Telephony | 电话 | 同上 |
| Bulk Transaction | 批量交易 | 同上 |
| Core / Desk / Geo / Printing / Automation | 核心 / 桌面 / 地理 / 打印 / 自动化 | mingda locale/zh.po |
| Mingda | 明达 | mingda locale/zh.po |

## 3.1 补充：模块定义列表的"模块名称"列（2026-07-03 追加）

「构建 → 模块定义」列表默认原样显示字段数据。经确认按以下方式处理：

- **"编号"列**（记录名）：系统标识符，绑定应用代码目录，**不翻译、不改名**（改名会导致模块路径解析失败）。
- **"模块名称"列**：通过 mingda 固件（fixture）部署一个列表客户端脚本（`模块定义列表-模块名称翻译`），用列表格式化器把该列的值过一遍翻译词库（`__()`）再显示——复用 §3 补翻的模块名词条，Manufacturing 显示为"生产"等。

固件随 `bench migrate` / `install-app` 自动同步，进 git、正式环境自动带上。

## 4. 维护注意事项

- erpnext zh.po 里追加的模块名词条**不在上游 POT 中**（文件尾部有注释标记），若将来运行 `bench update-po-files` 重新生成，需检查这些词条是否被清除并补回。
- mingda 的 zh.po 与代码同仓库管理，部署即生效（`bench compile-po-to-mo` 会编译所有应用的 po）。

## 5. 部署

```bash
# 服务器（frappe 用户）
cd /home/frappe/frappe-bench/apps/erpnext && git pull upstream version-16
cd /home/frappe/frappe-bench/apps/mingda && git pull
cd /home/frappe/frappe-bench
bench compile-po-to-mo
bench --site erp-test.local clear-cache
bench restart
```

验证：角色页面"模块集合"处 Utilities/Telephony 等显示中文；模块定义列表仍显示英文（预期内）。
