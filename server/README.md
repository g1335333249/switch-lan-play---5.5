# switch-lan-play — 中继服务器

本服务通过 UDP 在各台 Switch 之间转发局域网联机数据包。它是一个资源需求较低的 Node.js/TypeScript 应用（原文估计一台 512 MB 内存的 VPS 可支持数百名玩家同时在线）。

---

## 方案 A：使用云平台托管

### Oracle Cloud Always Free ⭐

[Oracle 官方说明](https://www.oracle.com/cloud/free/faq/)将 Ampere A1 列为 Always Free 资源。原文使用 1 OCPU、1 GB 内存的实例作为示例；创建时请确认实例和总资源用量都在免费额度内。

1. 在 <https://cloud.oracle.com/> 注册账号（需要信用卡验证；使用 Always Free 资源时不收费）。
2. 创建一台 **Ampere A1** 虚拟机（ARM、1 OCPU、1 GB 内存、Ubuntu 22.04）。
3. 通过 SSH 登录虚拟机并执行：

```sh
# 安装 Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 退出 SSH 并重新登录，然后执行：

# 克隆仓库
git clone https://github.com/shaklinedj/switch-lan-play---5.5
cd switch-lan-play---5.5/server

# 启动服务器
docker compose up -d

# 在 Oracle 防火墙中开放 UDP 端口 11451
# 同时还需在 Oracle 网页控制台的 VCN 安全列表中开放此端口
sudo iptables -I INPUT -p udp --dport 11451 -j ACCEPT
```

4. 在 Oracle 控制台查看虚拟机的**公网 IP 地址**。
5. 将 `你的公网IP:11451` 分享给其他玩家。

---

### fly.io（短期免费试用，后续按用量计费）

原文称其提供“免费爱好者套餐”，但 [Fly.io 当前的试用说明](https://fly.io/docs/about/free-trial/)限定了试用时长和用量；正式使用前请查看[价格说明](https://fly.io/docs/about/pricing/)。

```sh
# 安装 flyctl
curl -L https://fly.io/install.sh | sh

# 在 server/ 目录下执行：
fly launch --no-deploy
# 编辑 fly.toml：将 [[services]] 的 internal_port 设为 11451，protocol 设为 "udp"
fly deploy
```

---

### Railway.app / Render.com

原文建议使用本目录的 Dockerfile，并开放 `11451/udp`。但 [Railway 公网网络文档](https://docs.railway.com/networking)仅列出 HTTP/HTTPS 和 TCP 代理，[Render Web Service 文档](https://render.com/docs/web-services)则只提供公网 HTTP 端口。因此，不能直接按原文步骤在这两个平台托管此 UDP 中继服务。

---

## 方案 B：使用自己的 VPS（Hostinger、DigitalOcean、Hetzner 等）

任何具有公网 IP 的 Ubuntu VPS 均可使用。最低配置：512 MB 内存、1 个 vCPU。

```sh
# 安装 Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# 克隆仓库并启动服务
git clone https://github.com/shaklinedj/switch-lan-play---5.5
cd switch-lan-play---5.5/server
docker compose up -d

# 开放防火墙端口
sudo ufw allow 11451/udp
sudo ufw allow 11451/tcp   # 状态 API 使用（可选）
```

---

## 方案 C：在本机运行（测试或局域网聚会）

```sh
cd server
npm install
npm run build
npm run server
```

默认监听地址为 `0.0.0.0:11451`。

### 自定义端口

```sh
node dist/main.js --port 12345
```

---

## 验证服务器是否正常运行

```sh
# 在安装了 netcat 的任意设备上执行：
printf '\002\000\000\000' | nc -u YOUR_SERVER_IP 11451
# 正常情况下会收到相同的 4 字节响应。
```

也可以在浏览器中打开 `http://YOUR_SERVER_IP:11451/info`，查看包含在线客户端数量的 JSON 状态信息。

---

## 可选：密码保护

编辑 `server/users.json`（格式见 `users_schema.json`），然后运行：

```sh
node dist/main.js --jsonAuth ./users.json
```

玩家还需在各自的 `config.ini` 中设置 `username` 和 `password`。

---

## 端口开放汇总

| 端口 | 协议 | 用途 |
|------|------|------|
| 11451 | **UDP** | 转发游戏数据包（**必需**） |
| 11451 | TCP | 状态 API（可选，用于查看统计信息） |

---

## 更新

```sh
cd switch-lan-play---5.5/server
git pull
docker compose up -d --build
```
