<img width="1840" height="869" alt="image" src="https://github.com/user-attachments/assets/9648c32b-6d44-4e33-9c3c-0cdce6078834" />
# TG 机器人商城部署教程（Ubuntu 22.04 + 宝塔）

这是按实际部署成功路径重新整理的教程。请从头按顺序操作，不要混用旧教程。

## 一、只使用这一套架构

```text
浏览器
  -> 宝塔 Nginx（80/443）
  -> 反向代理 127.0.0.1:5002
  -> systemd 服务 tg-bot
  -> Gunicorn + Flask + Telegram Bot
  -> SQLite 数据库
```

重要规则：

1. 宝塔负责安装 Nginx、Python 3.11、创建网站、反向代理和 SSL。
2. `systemd` 负责启动和守护 Python 项目。
3. **不要在宝塔 Python 项目管理器中添加或启动 `tg-bot` 项目。**
4. **不要勾选宝塔 Supervisor/进程守护。**
5. **不要在终端长期手动运行 `bash deploy/start.sh`。**
6. 以上三种方式会和 systemd 重复启动，导致 `5002` 端口冲突。
7. 项目使用 SQLite，不需要安装 MySQL。

## 二、重装前备份（没有旧数据可跳过）

重装系统会清空服务器数据。如果需要保留旧后台账号、机器人、商品和订单，重装前
下载以下两项到本地：

```text
/www/wwwroot/tg-bot/.env
/www/wwwroot/tg-bot/instance/
```

说明：

- `.env` 保存后台配置和 Flask 密钥。
- `instance/mall.db` 是 SQLite 主数据库。
- `instance/backups/` 是数据库备份。

确认已经下载成功后才能重装系统。需要全新空数据库时，不恢复这些文件。

## 三、重装系统并开放端口

1. 在云服务器控制台重装 `Ubuntu 22.04 x64`。
2. 记录服务器公网 IP、root 密码或 SSH 密钥。
3. 云厂商安全组放行：

```text
22      SSH（修改过 SSH 端口则放行实际端口）
80      HTTP
443     HTTPS
宝塔安装完成后显示的面板端口
```

不要开放 `5002`。这个端口只允许服务器本机的 Nginx 访问。

## 四、安装宝塔

用 SSH 登录服务器，执行：

```bash
apt update
apt upgrade -y
apt install -y curl wget unzip ca-certificates
```

执行宝塔官方安装命令：

```bash
if [ -f /usr/bin/curl ]; then
    curl -sSO https://download.bt.cn/install/install_panel.sh
else
    wget -O install_panel.sh https://download.bt.cn/install/install_panel.sh
fi
bash install_panel.sh docscenter
```

询问是否安装到 `/www` 时输入 `y`。安装结束后保存面板地址、端口、安全入口、账号
和密码。如果忘记，SSH 终端执行：

```bash
bt default
```

## 五、宝塔只安装两个运行环境

登录宝塔后：

1. 进入 **软件商店**，安装 `Nginx` 稳定版。
2. 安装宝塔 Python 管理工具。
3. 进入 Python 工具的 **版本管理**，安装 Python `3.11.x`。
4. 等待 Python 3.11 显示安装完成。
5. 不要在 Python 项目管理器里点击“添加项目”。

本项目不需要安装 PHP、MySQL、MariaDB、Redis 或 phpMyAdmin。

## 六、上传正确的部署包

使用 `tg-bot-baota-ubuntu-20260917.zip`，不要上传原始下载源码或旧部署包。

1. 宝塔左侧点击 **文件**。
2. 进入 `/www/wwwroot`。
3. 上传 ZIP。
4. 解压 ZIP。
5. 最终目录必须是 `/www/wwwroot/`。

终端执行以下命令检查目录层级：

```bash
ls -l /www/wwwroot/tg-bot/app.py
ls -l /www/wwwroot/tg-bot/deploy/setup.sh
ls -l /www/wwwroot/tg-bot/deploy/install-systemd.sh
```

三条都显示文件才正确。下面这种双层目录是错误的：

```text
/www/wwwroot/tg-bot/tg-bot/app.py
```

## 七、一条命令部署 Python 服务

在宝塔终端或 SSH 中执行：

```bash
cd /www/wwwroot/tg-bot
bash deploy/setup.sh
```

这个脚本会自动：

- 寻找宝塔的 Python 3.11。
- 创建独立 `.venv`。
- 安装全部 Python 依赖和 Gunicorn。
- 创建 `.env` 和随机管理员密码。
- 设置 `www` 用户权限。
- 创建并启动 `tg-bot.service`。
- 设置服务器重启后自动启动。
- 检查 `http://127.0.0.1:5002/login`。

安装依赖可能需要几分钟，不要中途关闭终端。成功时最后必须看到：

```text
Deployment completed successfully.
Service: active
Local URL: http://127.0.0.1:5002/login (HTTP 200)
```

`Preflight check passed` 也是程序输出，不是下一条命令。

### 如果脚本找不到 Python 3.11

在宝塔 Python 版本管理里查看解释器实际路径。常见路径为：

```text
/www/server/python_manager/versions/3.11.4/bin/python3.11
```

然后执行（版本号按实际情况修改）：

```bash
cd /www/wwwroot/tg-bot
PYTHON_BIN=/www/server/python_manager/versions/3.11.4/bin/python3.11 \
    bash deploy/setup.sh
```

## 八、确认后端确实运行

执行：

```bash
systemctl is-enabled tg-bot
systemctl is-active tg-bot
ss -lntp | grep ':5002'
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
    http://127.0.0.1:5002/login
```

正确结果必须包含：

```text
enabled
active
127.0.0.1:5002
HTTP 200
```

如果不是这些结果，先执行：

```bash
cd /www/wwwroot/tg-bot
bash deploy/doctor.sh | tee /root/tg-bot-doctor.txt
```

该检查不会输出后台密码和 Telegram Token。

## 九、查看后台账号和密码

仅在自己的服务器终端执行：

```bash
cd /www/wwwroot/tg-bot
grep -E '^(ADMIN_USERNAME|ADMIN_PASSWORD)=' .env
```

妥善保存，不要发到群里或公开截图。注意：数据库已经创建管理员后，直接修改 `.env`
不会修改数据库里的旧密码，应进入后台使用修改密码功能。

## 十、创建独立的网站目录

执行：

```bash
mkdir -p /www/wwwroot/tg-bot-site
chown -R www:www /www/wwwroot/tg-bot-site
```

这个目录只用于宝塔 Nginx 站点。**不要把 `/www/wwwroot/tg-bot` 源码目录设为网站
根目录**，否则可能暴露 `.env` 和 SQLite 数据库，还会生成锁定的 `.user.ini`。

## 十一、在宝塔创建站点

进入宝塔 **网站 -> 添加站点**：

```text
域名：你的域名（推荐），没有域名可暂填服务器公网 IP
根目录：/www/wwwroot/tg-bot-site
PHP：纯静态
FTP：不创建
数据库：不创建
```

保存后，网站列表里的“运行中”只代表 Nginx，不代表 Python 项目。Python 项目状态
仍然通过 `systemctl is-active tg-bot` 查看。

## 十二、设置反向代理

打开站点的 **设置 -> 反向代理 -> 添加反向代理**：

```text
代理名称：tg-bot
目标 URL：http://127.0.0.1:5002
发送域名：$host
```

启用并保存。然后执行：

```bash
/www/server/nginx/sbin/nginx -t
/www/server/nginx/sbin/nginx -s reload
```

测试 HTTP：

```bash
curl -sS -o /dev/null -w 'Public HTTP %{http_code}\n' \
    http://你的域名或公网IP/login
```

返回 `200` 表示代理成功。返回 `502` 时执行：

```bash
systemctl status tg-bot
curl http://127.0.0.1:5002/login
```

因为 `502` 通常表示 Nginx 正常，但 Python 服务没运行。

## 十三、先选择访问方式

### 方案 A：有域名，正式使用（推荐）

1. 在域名服务商处添加 `A` 记录，指向服务器公网 IP。
2. 确认 `http://你的域名/login` 能返回页面。
3. 宝塔打开站点 **SSL -> Let's Encrypt/免费证书**。
4. 申请并部署证书。
5. 开启强制 HTTPS。
6. 访问 `https://你的域名/login`。

保持 `.env` 中：

```text
SESSION_COOKIE_SECURE=1
```

### 方案 B：只有 IP，临时 HTTP 测试

如果没有域名和 SSL，先执行：

```bash
cd /www/wwwroot/tg-bot
sed -i 's/^SESSION_COOKIE_SECURE=.*/SESSION_COOKIE_SECURE=0/' .env
systemctl restart tg-bot
```

然后使用：

```text
http://服务器公网IP/login
```

不要输入 `https://服务器公网IP/login`，除非确实给这个 IP 配置了有效证书。

切换 HTTPS 后恢复安全设置：

```bash
sed -i 's/^SESSION_COOKIE_SECURE=.*/SESSION_COOKIE_SECURE=1/' \
    /www/wwwroot/tg-bot/.env
systemctl restart tg-bot
```

修改 Cookie 设置后，清理浏览器对该 IP/域名的 Cookie，或使用无痕窗口重新登录。

## 十四、配置 Telegram Bot

1. 使用第九步的管理员账号登录后台。
2. 通过 BotFather 新申请 Telegram Bot Token。
3. 在后台添加并启用机器人配置。
4. 不要使用旧源码、旧日志或旧数据库里的 Token。
5. 保存后执行：

```bash
systemctl restart tg-bot
journalctl -u tg-bot -n 100 --no-pager
```

6. 在 Telegram 中向机器人发送 `/start`，验证是否回复。

本发布包已显式订阅 Telegram 的 `callback_query` 更新，商品、购买、支付等内联按钮
不再依赖 Bot Token 以前保存的更新过滤设置。

### 服务器无法连接 Telegram 时

先执行：

```bash
curl -I --max-time 15 https://api.telegram.org
```

海外服务器通常不需要代理。必须通过代理访问时，编辑 `.env`，取消以下两行注释并
填写服务器实际可用的代理地址：

```text
HTTPS_PROXY=http://127.0.0.1:7890
HTTP_PROXY=http://127.0.0.1:7890
```

然后重启并检查日志：

```bash
systemctl restart tg-bot
journalctl -u tg-bot -n 100 --no-pager
```

## 十五、SQLite 数据库和备份

项目使用 SQLite，不需要单独安装数据库服务。

```text
数据库：/www/wwwroot/tg-bot/instance/mall.db
备份目录：/www/wwwroot/tg-bot/instance/backups/
```

手动创建迁移备份：

```bash
tar -czf /root/tg-bot-backup-$(date +%F-%H%M%S).tar.gz \
    -C /www/wwwroot/tg-bot .env instance
```

通过宝塔文件管理器下载 `/root/` 下生成的压缩包，并保存到服务器以外的位置。

## 十六、支付功能警告

原项目部分支付回调没有完整验证支付服务商签名。完成对应平台的签名验证前，不要正式
收款。暂时不用支付时执行：

```bash
sed -i 's/^ENABLE_PAYMENT_CALLBACKS=.*/ENABLE_PAYMENT_CALLBACKS=0/' \
    /www/wwwroot/tg-bot/.env
systemctl restart tg-bot
```

## 十七、以后只用这些维护命令

```bash
# 查看状态
systemctl status tg-bot

# 重启
systemctl restart tg-bot

# 停止
systemctl stop tg-bot

# 启动
systemctl start tg-bot

# 查看最近日志
journalctl -u tg-bot -n 100 --no-pager

# 实时日志，按 Ctrl+C 只退出日志查看
journalctl -u tg-bot -f

# 一键排查
cd /www/wwwroot/tg-bot && bash deploy/doctor.sh
```

不要再用宝塔 Python 项目管理器启动，也不要手工后台运行 `deploy/start.sh`。

## 十八、故障对照表

### 浏览器完全打不开

先确认输入的是 `http://` 还是 `https://`。没有证书时只测试 HTTP，然后执行：

```bash
cd /www/wwwroot/tg-bot
bash deploy/doctor.sh
```

### `Unit tg-bot.service not found`

说明还没有完成第七步，执行：

```bash
cd /www/wwwroot/tg-bot
bash deploy/setup.sh
```

### `Connection refused` 或状态码 `000`

说明 `5002` 没有监听：

```bash
systemctl status tg-bot
journalctl -u tg-bot -n 100 --no-pager
```

### `502 Bad Gateway`

说明 Nginx 能访问，但后端不可用，检查：

```bash
systemctl is-active tg-bot
curl http://127.0.0.1:5002/login
```

### `Address already in use`

说明存在重复启动：

```bash
ss -lntp | grep ':5002'
```

删除宝塔 Python 项目管理器中的同名启动项，只保留 systemd，然后执行：

```bash
systemctl restart tg-bot
```

### 能看到登录页，但登录后又回到登录页

- HTTP/IP：设置 `SESSION_COOKIE_SECURE=0`，重启并清理 Cookie。
- HTTPS/域名：设置 `SESSION_COOKIE_SECURE=1`，确认 SSL 和反向代理正常。

### `Permission denied`

必须使用最新包重新创建 `.venv`：

```bash
systemctl stop tg-bot 2>/dev/null || true
cd /www/wwwroot/tg-bot
mv .venv ".venv.failed-$(date +%s)"
bash deploy/setup.sh
```

### `chown ... .user.ini: Operation not permitted`

`.user.ini` 是宝塔锁定文件，不要强制删除。站点根目录应改成：

```text
/www/wwwroot/tg-bot-site
```

### 登录页健康检查返回 500

使用最新包已修复 `HEAD /login`。正常检查推荐使用真正的 GET：

```bash
curl -sS -o /dev/null -w '%{http_code}\n' \
    http://127.0.0.1:5002/login
```

## 十九、部署完成检查清单

- [ ] 使用的是 `tg-bot-baota-ubuntu-20260917.zip`。
- [ ] 没有在宝塔 Python 项目管理器添加 `tg-bot`。
- [ ] `systemctl is-enabled tg-bot` 输出 `enabled`。
- [ ] `systemctl is-active tg-bot` 输出 `active`。
- [ ] `127.0.0.1:5002/login` 返回 `200`。
- [ ] 网站根目录是 `/www/wwwroot/tg-bot-site`。
- [ ] Nginx 反向代理目标是 `http://127.0.0.1:5002`。
- [ ] 公网 `/login` 返回 `200`，不是 `502`。
- [ ] 防火墙和安全组未开放 `5002`。
- [ ] 正式部署使用域名和 HTTPS。
- [ ] 已保存后台密码并使用新 Telegram Bot Token。
- [ ] 已把 `.env` 和 `instance/` 备份到服务器外。
- [ ] 正式收款前已完成支付回调验签。
