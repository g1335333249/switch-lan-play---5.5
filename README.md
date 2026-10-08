# switch-lan-play

[![Discord 聊天](https://img.shields.io/badge/chat-en%20discord-7289da.svg)](https://discord.gg/zEMCu5n)

无需电脑，直接在 Nintendo Switch 上通过互联网与朋友游玩本地多人游戏。

---

## 目录

1. [工作原理](#工作原理)
2. [快速安装](#快速安装)
3. [项目组件](#项目组件)
4. [中继服务器](#中继服务器)
5. [开发工具](#开发工具)
6. [从源码编译](#从源码编译)
7. [高级配置](#高级配置)
8. [协议](#协议)
9. [故障排查](#故障排查)

---

## 工作原理

本项目在 Switch 上配合使用**两个 sysmodule（系统模块）**：

```text
Switch（游戏的“本地无线联机”模式）
    |
    v
ldn_mitm（Title ID：4200000000000010）
    |  拦截 Nintendo LDN 服务，将通信转换为
    |  端口 11452 上的 LAN UDP/TCP 数据包
    |
    v
lan-play sysmodule（Title ID：42000000000000B1）
    |  LDN Bridge：捕获端口 11452 的流量，
    |  重写 IP（ALG）并发送到中继服务器
    |
    v  WiFi 上的 UDP
中继服务器（端口 11451）
    |
    v
其他玩家（Switch / PC / 模拟器）
```

### 数据流

1. **ldn_mitm** 拦截游戏的 LDN 调用，将其转换为本地 UDP 广播（端口 11452）。
2. **lan-play sysmodule** 在桥接端口 11453 捕获这些数据包，将实际 WiFi IP 重写为 `10.13.x.x` 虚拟 IP，再发送到中继服务器。
3. 来自中继服务器的数据包通过 `127.0.0.1:11452` 注入回 ldn_mitm（使用 loopback，兼容 Horizon OS）。
4. 端口 11453 上的 **TCP 代理**通过中继服务器处理 Station → Access Point 连接。

### 支持哪些游戏？

- **官方提供 LAN 模式的游戏**：例如《马力欧卡丁车 8 豪华版》（按 L + R + 左摇杆进入 LAN 模式），可直接使用。
- **仅提供本地无线联机的游戏**：例如《任天堂明星大乱斗》《宝可梦》和《集合啦！动物森友会》，由 ldn_mitm 将本地无线通信转换为 LAN 通信。

每台 Switch 都会根据设备序列号，自动获得 `10.13.0.0/16` 范围内的一个**唯一虚拟 IP**。

---

## 快速安装

> 需要安装 **Atmosphere CFW ≥ 1.11.0** 的 Nintendo Switch，并连接 WiFi。

### 第 1 步：下载

从 [Releases](https://github.com/shaklinedj/switch-lan-play---5.5/releases) 下载 `switch-lan-play-all-in-one-v1.15.zip`。

### 第 2 步：复制到 SD 卡

将 ZIP 文件解压到 **SD 卡根目录**。解压后的目录结构如下：

```text
sdmc:/
+-- atmosphere/
|   +-- contents/
|   |   +-- 4200000000000010/          <- ldn_mitm（拦截 LDN → LAN）
|   |   |   +-- exefs.nsp
|   |   |   +-- flags/boot2.flag
|   |   |   +-- toolbox.json
|   |   +-- 42000000000000B1/          <- lan-play sysmodule（中继桥接）
|   |       +-- exefs.nsp
|   |       +-- flags/boot2.flag
|   |       +-- toolbox.json
|   +-- hosts/
|       +-- default.txt                <- DNS 覆盖配置（可选）
+-- switch/
    +-- .overlays/
    |   +-- ldnmitm_config.ovl         <- ldn_mitm 的 Tesla 悬浮菜单
    +-- lan-play/
    |   +-- lanplay-setup.nro           <- 配置应用
    |   +-- lanplay-debug.nro           <- 自制程序调试工具
    |   +-- lanplay-sys-debug.nro       <- sysmodule 调试工具
    +-- ldnmitm_config/
        +-- ldnmitm_config.nro          <- ldn_mitm 配置应用
```

### 第 3 步：配置中继服务器

1. 重启 Switch（两个 sysmodule 会通过 `boot2.flag` 自动启动）。
2. 打开 **Homebrew Menu**，运行 **LanPlay Setup**。
3. 按 **A**，输入中继服务器地址（例如 `192.168.1.100:11451`），再按 **+**。
4. 重启 Switch。

### 第 4 步：开始游戏

1. 打开支持的游戏。
2. 选择**本地无线联机**（如果游戏提供 LAN 模式，也可以选择 LAN 模式）。
3. 连接到同一中继服务器的玩家会自动发现彼此。

> **注意：**目前可在 sysmodule 的中继配置中使用 IP 地址。主机名（例如 `tekn0.net:11451`）在 Switch 上的支持情况仍待验证。

---

## 项目组件

| 目录 | 说明 |
|------|------|
| `sysmodule/` | 主 sysmodule（Title ID `42000000000000B1`）：LDN 桥接、中继客户端和虚拟 IP。 |
| `ldn_mitm-1.25.1/` | 修改后的 ldn_mitm v1.25.1 分支，将 LDN 转换为 LAN。**本项目的修改：**访问虚拟 IP 时，通过中继代理 `127.0.0.1:11453` 建立 TCP 连接。 |
| `hbapp/` | 在 Switch 上配置中继服务器的 LanPlay Setup 自制程序。 |
| `server/` | 基于 Node.js/TypeScript 的 UDP 中继服务器。 |
| `all_in_one/` | 包含已编译二进制文件、可直接复制到 SD 卡的整合包。 |
| `tools/` | 开发工具：`pc-peer.ts`（测试节点）、`decode_scanresp.js`（LDN 载荷解码器）。 |
| `src/` | 原版 PC 客户端（通过 libpcap 捕获数据包）。 |
| `lwip/` | 轻量级 TCP/IP 协议栈，供 PC 客户端使用。 |

### 当前版本

| 组件 | 版本 | Title ID |
|------|------|----------|
| lan-play sysmodule | v1.15 | `42000000000000B1` |
| ldn_mitm（修改版） | v1.25.1 | `4200000000000010` |
| Atmosphere 最低要求 | ≥ 1.11.0 | — |

---

## 中继服务器

服务器监听 `11451/UDP`，在所有已连接的主机之间转发数据包。

### Docker（推荐）

```sh
cd server
docker compose up -d
```

### 直接使用 Node.js

```sh
cd server
npm install
npm run build
npm start
```

可用参数：

| 参数 | 说明 |
|------|------|
| `--port 11451` | UDP 端口（默认 `11451`） |
| `--simpleAuth 用户名:密码` | 基本身份验证 |
| `--jsonAuth ./users.json` | 使用 JSON 文件进行身份验证 |

### 状态监控

```text
GET http://服务器IP:11451/info
→ { "online": 5 }
```

### 所需端口

| 端口 | 协议 | 用途 |
|------|------|------|
| 11451 | **UDP** | 转发 LAN 数据包（**必需**） |
| 11451 | TCP | 状态 API（可选） |

### 免费托管

Oracle Cloud、fly.io、Railway 等平台的部署指南见 [server/README.md](server/README.md)。

---

## 开发工具

### pc-peer.ts：PC 测试节点

让 PC 作为虚拟节点连接中继服务器，无需两台 Switch 即可测试 Scan/ScanResp。

```sh
cd server && npm install && cd ..
npx --prefix ./server ts-node --project ./server/tsconfig.json ./tools/pc-peer.ts <中继服务器> <端口> <虚拟IP>
```

交互式控制台命令：

- `scan`：发送 LDN Scan 广播。
- `autoscan [ms]`：每隔指定毫秒数自动扫描。
- `stopscan`：停止自动扫描。
- `ping <ip>`：向虚拟节点发送 ping。
- `stats`：显示统计信息。
- `quit`：退出。

### decode_scanresp.js：载荷解码器

```sh
node tools/decode_scanresp.js
```

输出 ScanResp 载荷的十六进制和 ASCII 对照内容，便于查看。

---

## 从源码编译

### Sysmodule（Switch）

需要支持 Switch 开发的 **devkitPro**：

```sh
dkp-pacman -S switch-dev switch-atmo-tools

cd sysmodule
make
# 生成的 atmosphere/ 目录复制到 SD 卡根目录
```

### ldn_mitm（Switch）

```sh
cd ldn_mitm-1.25.1
git submodule update --init --recursive
make
# 也可以使用 Docker：
docker-compose up --build
```

### Homebrew 配置应用

```sh
cd hbapp
make
# 生成的 lanplay-setup.nro 复制到 sdmc:/switch/lan-play/
```

### PC 客户端（可选）

```sh
mkdir build && cd build
cmake ..
make
```

需要 `libpcap-dev`（Linux）、`npcap`（Windows）或 `libpcap`（macOS）。

#### Windows（MSYS2/MinGW）

1. 安装 **MSYS2**，打开 `MSYS2 MinGW 64-bit`。
2. 安装工具链及相关工具：

   ```sh
   pacman -S --needed mingw-w64-x86_64-gcc mingw-w64-x86_64-cmake mingw-w64-x86_64-make git
   ```

3. 确保 `C:\msys64\mingw64\bin` 位于 `PATH` 中。
4. 在仓库根目录的 PowerShell 中运行：

   ```powershell
   powershell -ExecutionPolicy Bypass -File .\tools\build_pc_windows_mingw.ps1 -BuildDir build -Config Release
   ```

如果仓库来自不含 `.git` 目录的 ZIP 文件，CMake 会在配置时自动下载 `libuv` 和 `uvw`。

---

## 高级配置

### 配置文件

位置：`sdmc:/config/lan-play/config.ini`

```ini
[server]
relay_addr = 192.168.1.100:11451

; 可选：指定虚拟 IP（默认根据设备序列号生成）
; ip = 10.13.5.10

; 可选：身份验证
; username = myuser
; password = mypassword
```

### 自动分配虚拟 IP

每台 Switch 都会根据序列号，在 `10.13.1.1-10.13.254.254` 范围内获得固定的虚拟 IP，无需 DHCP 或手动配置。

### Tesla 悬浮菜单（ldn_mitm）

如果安装了 Tesla Menu，可以通过 `ldnmitm_config.ovl` 悬浮菜单实时启用或禁用 ldn_mitm，无需重启。

---

## 协议

### 中继协议（端口 11451）

```c
struct packet {
    uint8_t type;       // 0=KEEPALIVE, 1=IPV4, 2=PING, 3=IPV4_FRAG
    uint8_t payload[];
};
```

### LDN（端口 11452）

```c
struct ldn_header {
    uint32_t magic;             // 0x11451400
    uint8_t  type;              // 0=Scan, 1=ScanResp, 2=Connect, 3=SyncNetwork
    uint8_t  compressed;
    uint16_t length;            // 数据体长度
    uint16_t decompress_length;
    uint8_t  reserved[2];
};
// 后续为 NetworkInfo（ScanResp/SyncNetwork）或空数据（Scan）
```

---

## 技术特性

- **LDN Bridge**：捕获 ldn_mitm 的 LDN 流量，重写 IP（ALG）后发送到中继服务器。
- **Loopback 注入**：使用 `127.0.0.1:11452` 代替广播，兼容不会将广播回送到本机的 Horizon OS。
- **TCP 代理**：通过端口 11453 和中继服务器转发 Station → AP TCP 连接。
- **DNS 绕过（inet_pton）**：使用 IP 地址时跳过 Nintendo DNS 解析，避免出现“System Busy”错误。
- **线程权限模拟**：动态复制线程权限，避免 Atmosphere 的 CPU 限制导致崩溃。
- **固定虚拟 IP**：根据设备序列号生成，无需 DHCP。

---

## 故障排查

| 现象 | 排查方法 |
|------|----------|
| Sysmodule 无法启动 | 确认 Atmosphere ≥ 1.11.0，并检查两个 Title ID 对应目录中是否都有 `boot2.flag`。 |
| 看不到其他玩家的房间 | 确认双方连接到同一中继服务器，且游戏已进入“本地无线联机”模式。 |
| Homebrew Menu 中找不到 LanPlay Setup | 检查 `sdmc:/switch/lan-play/lanplay-setup.nro` 是否存在。 |
| 无法连接中继服务器 | 检查服务器防火墙是否放行 `11451/UDP`。 |
| ldn_mitm 没有拦截游戏 | 检查 `4200000000000010` 目录中是否有 `boot2.flag`，然后重启 Switch。 |
| 延迟较高 | 选择地理位置靠近所有玩家的中继服务器。 |

---

## 许可证

[MIT](LICENSE.txt)

---

## 致谢

- [spacemeowx2/switch-lan-play](https://github.com/spacemeowx2/switch-lan-play)：原项目。
- [spacemeowx2/ldn_mitm](https://github.com/spacemeowx2/ldn_mitm)：原版 ldn_mitm。
- [Atmosphere-NX](https://github.com/Atmosphere-NX/Atmosphere)：Nintendo Switch 自定义固件。
