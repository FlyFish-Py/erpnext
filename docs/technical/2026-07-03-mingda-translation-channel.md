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

## 3.1 补充：系统清单类列表的数据列翻译（2026-07-03 追加，机制两次演进）

「构建」等工作区的系统清单列表默认原样显示字段数据。处理原则：

- **记录名列（编号）= 系统标识符**（绑定代码目录/表名/API 路径），**一律不翻译、不改名**。
- **普通数据列**（模块、单据类型引用、类型枚举等）：用列表格式化器把值过一遍翻译词库（`__()`）再显示。

实现机制演进：最初做成 Client Script 固件（每张表一条记录），随覆盖面扩大改为 **mingda 全局脚本** `public/js/mingda_list_translations.js`（`app_include_js` 挂载）——所有"表 → 翻译列"集中在一张登记表里，新增翻译列加一行即可。当前覆盖：

| 列表 | 翻译的列 |
|---|---|
| 模块定义 | 模块名称 |
| 单据类型 / 页面 / 工作区 / 数据面板 | 模块 |
| 报表 | 单据类型、报表类型、模块 |
| 工作流 / 通知 | 单据类型 |
| Python 脚本 | 脚本类型、引用单据类型 |
| 客户端脚本 | 单据类型、视图 |
| 自定义字段 | 单据类型、字段类型 |
| 属性设置 / 打印格式 | 单据类型 |

> 若测试站点曾同步过早期的两条 Client Script 固件（`模块定义列表-模块名称翻译`、`单据类型列表-模块列翻译`），请在"客户端脚本"列表中手动删除，避免与全局脚本重复。

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
