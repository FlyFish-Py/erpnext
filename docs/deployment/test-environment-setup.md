# ERPNext 测试环境部署文档

| 项目 | 内容 |
|---|---|
| 部署日期 | 2026-07-02 |
| 环境定位 | **测试环境**（正式环境后续在 PVE 另分 VM 按本文档复刻） |
| 部署结果 | ✅ 成功上线，浏览器可正常访问登录 |
| 访问地址 | 内网 `http://192.168.1.114`（外网映射见 §6 待办） |
| 登录账号 | `Administrator`（密码为建站时设置，未记录在文档中） |

---

## 1. 环境与版本清单

### 1.1 硬件 / 网络

| 项 | 值 |
|---|---|
| 宿主 | PVE（Proxmox）虚拟机 |
| 操作系统 | Debian 13.5 (trixie)，DVD 安装 |
| 资源 | 16 核 CPU / 31G 内存 / 186G 磁盘 |
| 内网 IP | 192.168.1.114（**当前为 DHCP 动态获取**，待做静态保留，见 §6） |
| 公网 IP | 183.249.172.67（中国移动，动态） |
| 端口映射 | WAN 18822 → 22（SSH，路由器规则 Game01） |

### 1.2 软件版本

| 组件 | 版本 | 备注 |
|---|---|---|
| Python | 3.14.6 | frappe v16 硬性要求 `>=3.14,<3.15`；系统自带 3.13 不满足，用 uv 安装独立版本 |
| Node.js | 24.18.0 | frappe v16 要求 `>=24`，NodeSource 源安装 |
| MariaDB | 11.8.6 | Debian 13 官方源 |
| Redis | 8.0.2 | Debian 13 官方源（bench 生产模式自管两个实例：cache 13000 / queue 11000） |
| bench | 5.31.0 | pipx 安装 |
| uv | 0.11.26 | pipx 安装，负责 venv 和 Python 3.14 |
| frappe 框架 | v16.25.0（version-16 分支） | 官方仓库 frappe/frappe |
| ERPNext | v16.26.1（version-16 分支） | **二开 fork：FlyFish-Py/erpnext** |
| yarn | 1.22.22 | npm 全局安装 |

### 1.3 站点信息

| 项 | 值 |
|---|---|
| bench 目录 | `/home/frappe/frappe-bench` |
| 站点名 | `erp-test.local`（默认站点，`dns_multitenant off`，任意 Host/IP 均可访问） |
| 数据库名 | `erpnext_test` |
| 运行模式 | 生产模式（supervisor 守护 + nginx 反代 80 端口） |
| 调度器 | 已启用（enable-scheduler） |

---

## 2. 关键决策记录

1. **二开基线 = version-16 稳定分支**，不用 develop（无发布保障）。fork 当时只含 develop 分支，已执行 `git push origin refs/remotes/upstream/version-16:refs/heads/version-16` 把上游 version-16 推入 fork，作为二开主线。
2. **单机全家桶**：MariaDB / Redis / 应用同装一台 VM，不在 PVE 层拆分服务虚拟机（同一物理机拆分无高可用收益，徒增运维负担）。环境隔离靠"测试/正式各一台 VM"实现。
3. **bench 直装（非 Docker）**：贴合二开高频迭代（git pull + migrate + restart），且中文社区资料最全。
4. **数据库选 MariaDB**：ERPNext 官方主流；Postgres 在 v16 仍属次级支持。
5. 服务器在国内，**全程配置国内镜像**（apt/pip/uv/npm/yarn/Python 下载），否则多处会超时失败。

---

## 3. 部署步骤（成功路径）

> 以下为踩坑修正后的**正确顺序**。⚠️ 标注的步骤顺序不可颠倒。
> 命令前缀：`[root]` = root 用户执行；`[frappe]` = frappe 用户执行。

### 3.1 修复 APT 源（DVD 安装的系统默认只有光盘源）

```bash
# [root]
cp /etc/apt/sources.list /etc/apt/sources.list.bak 2>/dev/null
echo "" > /etc/apt/sources.list

cat > /etc/apt/sources.list.d/debian.sources <<'EOF'
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/debian
Suites: trixie trixie-updates trixie-backports
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOF

apt update
```

### 3.2 系统依赖 + Node 24

```bash
# [root]
apt install -y git curl pkg-config sudo cron \
  python3-dev python3-venv python3-pip pipx \
  mariadb-server libmariadb-dev libmariadb-dev-compat \
  redis-server \
  xvfb libfontconfig1 fonts-noto-cjk \
  nginx supervisor build-essential

# Node.js 24（frappe v16 要求 >= 24）
curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
apt install -y nodejs

npm config set registry https://registry.npmmirror.com
npm install -g yarn
```

### 3.3 创建运维用户 frappe

```bash
# [root]（bench 拒绝在 root 下运行）
adduser --gecos "" frappe
usermod -aG sudo frappe
```

### 3.4 MariaDB 配置

```bash
# [root] Frappe 必需的 utf8mb4 字符集
cat > /etc/mysql/mariadb.conf.d/99-frappe.cnf <<'EOF'
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
EOF
systemctl restart mariadb

# 给 root 设密码（替换 <DB密码>；建站时要用）
mariadb -e "ALTER USER 'root'@'localhost' IDENTIFIED VIA mysql_native_password USING PASSWORD('<DB密码>'); FLUSH PRIVILEGES;"
```

### 3.5 frappe 用户环境（镜像 + bench + Python 3.14）

```bash
# [frappe]（su - frappe 切换）

# pip 镜像
mkdir -p ~/.config/pip
cat > ~/.config/pip/pip.conf <<'EOF'
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
EOF

# npm/yarn 镜像（每个用户单独一份）
npm config set registry https://registry.npmmirror.com
yarn config set registry https://registry.npmmirror.com

# uv 镜像（uv 不读 pip.conf，需单独配置）
echo 'export UV_DEFAULT_INDEX=https://pypi.tuna.tsinghua.edu.cn/simple' >> ~/.bashrc
echo 'export UV_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple' >> ~/.bashrc
echo 'export UV_PYTHON_INSTALL_MIRROR=https://registry.npmmirror.com/-/binary/python-build-standalone' >> ~/.bashrc
exec $SHELL -l

# bench + uv
pipx install frappe-bench
pipx install uv
pipx ensurepath
exec $SHELL -l

# Python 3.14（frappe v16 硬性要求）
uv python install 3.14
```

### 3.6 初始化 bench（拉取 frappe 框架）

```bash
# [frappe]
cd ~
bench init --frappe-branch version-16 --python $(uv python find 3.14) frappe-bench
# 成功标志：SUCCESS: Bench frappe-bench initialized
```

### 3.7 安装 ERPNext（从二开 fork）

```bash
# [frappe]
cd ~/frappe-bench
bench get-app erpnext https://github.com/FlyFish-Py/erpnext.git --branch version-16
```

> GitHub 国内访问不稳定，连接超时就重试；持续不通换加速代理：
> `bench get-app erpnext https://ghfast.top/https://github.com/FlyFish-Py/erpnext.git --branch version-16`
> 走代理安装后校验完整性：`git -C apps/erpnext rev-parse HEAD` 应与 fork 上 version-16 的提交一致。

### 3.8 ⚠️ 先开生产模式（必须在建站之前）

> ERPNext 建站收尾时要连 Redis Queue（端口 11000），生产模式不先起来建站必失败。

```bash
# [root] bench/ansible 装在 frappe 用户目录，sudo 环境找不到，先做全局软链接
ln -s /home/frappe/.local/bin/bench /usr/local/bin/bench
ln -s /home/frappe/.local/share/pipx/venvs/frappe-bench/bin/ansible /usr/local/bin/ansible
ln -s /home/frappe/.local/share/pipx/venvs/frappe-bench/bin/ansible-playbook /usr/local/bin/ansible-playbook
ln -s /home/frappe/.local/share/pipx/venvs/frappe-bench/bin/ansible-galaxy /usr/local/bin/ansible-galaxy

# [frappe] 生产模式（自动配置 supervisor/nginx/fail2ban）
sudo bench setup production frappe

# supervisor 配置软链接（setup production 未自动创建，手动补）
sudo ln -s /home/frappe/frappe-bench/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf
sudo supervisorctl reread && sudo supervisorctl update
sudo supervisorctl status   # 应见 redis/web/workers 共 7 个进程 RUNNING
```

### 3.9 建站与站点配置

```bash
# [frappe] 交互输入：MySQL root 密码（§3.4 设置的）→ 设 Administrator 网页登录密码
bench new-site erp-test.local --db-name erpnext_test --install-app erpnext

bench use erp-test.local
bench config dns_multitenant off
bench --site erp-test.local enable-scheduler
```

### 3.10 nginx 配置与修复

```bash
# [frappe] 生成站点 nginx 配置（问覆盖答 y）
sudo bench setup nginx

# 修复一：bench 配置引用了 Debian nginx 没有的 log_format "main"，补全局定义
sudo tee /etc/nginx/conf.d/a-log-format.conf > /dev/null <<'EOF'
log_format main '$host $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';
EOF

# 修复二：移除 Debian 默认站点（与 bench 配置抢 80 端口 default_server）
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t && sudo systemctl restart nginx
bench restart
```

### 3.11 权限修复（页面无样式问题）

```bash
# [frappe] Debian 13 新建用户家目录默认 700，nginx(www-data) 读不到静态资源
chmod 755 /home/frappe
```

### 3.12 验证清单

- [x] `sudo supervisorctl status`：7 个进程全部 RUNNING
- [x] `systemctl is-active nginx mariadb supervisor`：均 active
- [x] 内网浏览器 `http://192.168.1.114`：登录页样式正常
- [x] Administrator 登录成功

---

## 4. 踩坑记录

| # | 现象 | 原因 | 解决 |
|---|---|---|---|
| 1 | `apt install` 全部找不到包 | DVD 安装的系统 APT 只有光盘源 | 换清华镜像源（§3.1） |
| 2 | `bench init` 报 `FileNotFoundError: 'uv'` | 新版 bench 用 uv 建虚拟环境，未预装 | `pipx install uv` |
| 3 | `uv pip install frappe` 版本解析失败 | frappe v16 要求 Python `>=3.14,<3.15`，系统只有 3.13 | `uv python install 3.14`，init 时 `--python $(uv python find 3.14)` |
| 4 | `bench get-app` 连接 GitHub 超时 | 国内网络访问 GitHub 不稳定（时通时断） | 重试即通；备用方案 ghfast.top 加速代理 |
| 5 | 建站尾段报 `Error 111 connecting to 127.0.0.1:11000` | ERPNext 安装收尾要连 Redis Queue，而服务未启动 | **先** `bench setup production` **再**建站（§3.8） |
| 6 | 建站失败回滚卡死（DROP DATABASE 不动） | 安装进程残留 DB 连接持有元数据锁（Sleep 状态） | `SHOW PROCESSLIST` 找到 Sleep 的连接 → `KILL <id>` |
| 7 | `setup production` 报找不到 `bench`/`ansible` | pipx 装在用户目录，sudo/子进程的 PATH 里没有 | 软链接到 `/usr/local/bin`（§3.8） |
| 8 | `supervisorctl status` 无输出 | setup production 未创建 conf.d 软链接 | 手动软链 + `reread`/`update`（§3.8） |
| 9 | `nginx -t` 报 `unknown log format "main"` | bench 模板引用的 log_format 在 Debian nginx 中未定义 | 补 `a-log-format.conf` 全局定义（§3.10） |
| 10 | 页面能打开但无样式、图片碎裂 | Debian 13 家目录默认 700，nginx 无权读 `/home/frappe/...` 静态资源（error.log 大量 Permission denied） | `chmod 755 /home/frappe`（§3.11） |

---

## 5. 日常运维速查

```bash
# 服务状态 / 重启（frappe 用户）
sudo supervisorctl status          # 7 个进程
bench restart                      # 重启 web + workers（代码变更后）
sudo systemctl restart nginx       # nginx

# 日志位置
/home/frappe/frappe-bench/logs/    # 应用日志（web.log、worker.log、schedule.log）
/var/log/nginx/error.log           # nginx 错误（静态资源 403/404 看这里）

# 站点备份（输出在 sites/erp-test.local/private/backups/）
bench --site erp-test.local backup

# 更新代码后的标准动作（二开迭代用，后续单独出流程文档）
cd ~/frappe-bench/apps/erpnext && git pull
bench build --app erpnext
bench --site erp-test.local migrate
bench restart
```

**进程架构**：nginx(:80) → gunicorn(:8000) / socketio(:9000)；redis cache(:13000) + queue(:11000)；workers（short/long）+ schedule 由 supervisor 守护。

---

## 6. 待办事项

| 事项 | 说明 | 优先级 |
|---|---|---|
| 路由器给 VM 做 DHCP 静态保留 | 当前 192.168.1.114 是动态获取，IP 变了端口映射即失效 | 高 |
| 外网 Web 端口映射 | 建议 WAN 18080 → 192.168.1.114:80（**勿用 80/443**，运营商封锁）；访问 `http://183.249.172.67:18080` | 按需 |
| SSH 加固 | 改密钥登录、禁用 root 密码登录（fail2ban 已启用） | 中 |
| HTTPS + 域名/DDNS | 公网 IP 动态，长期外网使用需 DDNS；正式环境必须 HTTPS | 正式环境前 |
| 正式环境复刻 | PVE 另分 VM，按本文档执行；差异点：站点名/数据库名、资源配置、HTTPS | 规划中 |
