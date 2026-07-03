# BUG-002：默认密码登录后跳转报"301 未捕获异常"

| 项目 | 内容 |
|---|---|
| 日期 | 2026-07-03 |
| 严重级别 | 高（新用户登录后无法进入改密流程，卡在错误页） |
| 关联需求 | REQ-001 默认密码策略 |
| 关联 Bug | BUG-001 修复后遗留 |

## 1. 现象

新建账号用默认密码 `123456` 登录成功后，页面显示"服务器错误 301: 未捕获异常"。

## 2. 根因

`enforce_password_reset`（auth_hooks，每请求拦截）用 `raise frappe.Redirect` 实现重定向。但 `frappe.Redirect` 只在**网页渲染阶段**有对应的异常处理；auth 验证阶段抛出后落入 frappe 请求入口的通用异常处理（app.py：只有 werkzeug 的 `HTTPException` 会被转换成正常响应，其余一律渲染错误页），于是重定向变成了 301 错误页。

登录本身、默认密码写入、须改密标记全部正常——只是"拦截后跳去改密页"这最后一步的机制用错了。

## 3. 修复

改为抛 **werkzeug 的 `HTTPException`** 并携带重定向响应，frappe 入口会原样返回：

```python
from werkzeug.exceptions import HTTPException
from werkzeug.utils import redirect

raise HTTPException(response=redirect("/update-password"))
```

## 4. 经验教训

- 在 frappe 的 **auth/请求前置阶段**做重定向，必须用 werkzeug `HTTPException(response=redirect(...))`；`frappe.Redirect` 只适用于 website 页面渲染上下文。
- 自定义钩子写完后，测试要覆盖"从登录到目标页"的完整用户路径，不能只测钩子函数本身。
