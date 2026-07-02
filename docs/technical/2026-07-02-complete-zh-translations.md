# 技术记录：补全 ERPNext 中文翻译（zh.po 100% 覆盖）

| 项目 | 内容 |
|---|---|
| 日期 | 2026-07-02 |
| 类型 | 本地化改进（首个二开改动） |
| 改动文件 | `erpnext/locale/zh.po`（1544 处 msgstr 补全，3 处误标修正） |
| 分支 | version-16 |

## 1. 背景

测试环境部署完成后，界面语言切换为 zh（简体中文）时仍有大量英文残留。排查确认根源：ERPNext 的中文翻译由社区（Crowdin）贡献，**上游覆盖率只有 84.5%**——9963 个词条中有 1544 条没有中文翻译，原样显示英文。这不是部署问题。

> 注意：登录页 "Forgot Password" 等属于 **frappe 框架**的词条（frappe/frappe 仓库，不在本 fork 范围），本次未处理，见 §5 遗留事项。

## 2. 方案

直接补全 fork 中的 `erpnext/locale/zh.po`（gettext PO 格式），提交到 version-16 二开主线。相比"数据库自定义翻译（Translation DocType）"方案，PO 文件随代码走 git，可审阅、可回滚、部署即生效，符合项目的文档化/版本化要求。

**术语约定**：严格沿用现有 84.5% 译文的风格（SAP 风格），保证整体一致：

| 英文 | 译法 | 英文 | 译法 |
|---|---|---|---|
| Item | 物料 | Stock Entry | 物料移动 |
| Delivery Note | 销售出库（单） | Purchase Receipt | 采购入库（单） |
| Journal Entry | 日记账凭证 | Payment Entry | 收付款凭证 |
| Batch | 批号 | Work Order | 生产工单 |
| Subcontracting | 委外 | Phantom Item | 虚拟件 |
| Party | 往来单位 | Account | 科目（会计）/账户（银行） |
| Reconcile | 对账（银行）/核销（付款） | Posting Date | 记账日期 |
| Backflush | 倒冲 | Blanket Order | 框架订单 |

## 3. 实施方法

脚本辅助 + 人工翻译，分 6 批完成（每批约 260 条）：

1. 脚本解析 PO，提取全部空 msgstr 词条 → `untranslated.json`
2. 从已有译文抽取术语对照表，统一用词
3. 分批人工翻译，生成 msgid→msgstr 映射 JSON
4. 脚本合并回 zh.po，**合并时逐条校验占位符**（`{0}`、`{name}`、`%s` 等在原文/译文中必须一致，不一致则拒绝写入）——全程 0 条占位符错误
5. `msgfmt -c` 做 gettext 格式校验

**校验结果**：翻译后 msgfmt 报 12 处格式错误 → 对比原文件（本就有 9 处上游遗留错误）定位出新增 3 处，均为上游对含 `%` 普通文本的词条误标 `#, python-format` 所致（如 `0% rate` 被解析成 `%r` 格式符）。修正方式：移除这 3 条的误标记。最终恢复到与上游相同的 9 处遗留错误（不影响编译，装机时已验证），**本次改动零新增错误**。

## 4. 部署步骤（测试环境）

```bash
# [frappe 用户]
cd ~/frappe-bench/apps/erpnext
git pull upstream version-16          # 该仓库的 upstream 远程即我们的 fork

cd ~/frappe-bench
bench compile-po-to-mo                # PO 编译为 MO
bench --site erp-test.local clear-cache
bench restart
```

浏览器强制刷新（Ctrl+Shift+R）后生效。

## 5. 遗留事项

| 事项 | 说明 |
|---|---|
| frappe 框架英文残留 | 登录页、桌面通用控件等词条在 frappe/frappe 仓库。方案待定：fork frappe 补全其 zh.po，或用站点级 Translation 记录覆盖。需要先量化框架侧缺口。 |
| 上游 9 处 PO 格式错误 | 旧译文中的 python-format 误标（第 254/260/266/38221/38226/38252/38293/40071/62085 行），上游问题，不影响编译，暂不处理。 |
| 新增词条的持续维护 | 以后从上游合并新版本时，`bench update-po-files` 可能带来新未翻译词条，按本文方法增量补全。 |
