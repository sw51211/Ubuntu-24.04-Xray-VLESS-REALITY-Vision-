# Ubuntu 24.04 部署 Xray VLESS + REALITY + Vision：从零配置与故障排查

> 一份可复用的自建代理部署实践文档，适用于 Ubuntu 24.04 服务端与 Windows v2rayN 客户端。请遵守当地法律法规、服务商条款和所在网络的使用政策。
>
> **安全说明：** 本文中的 IP、UUID、公私钥和 Short ID 均为占位符或示例值，不是可直接使用的生产凭据。发布 GitHub 前请勿粘贴真实配置、订阅链接、终端截图或访问日志。

## 1. 架构和配置思路

```text
Windows / 浏览器
    │
    ├─ v2rayN（本地 HTTP/SOCKS 代理）
    │
    └─ VLESS + REALITY + Vision（TCP）
                │
                ▼
    Ubuntu VPS（Xray 入站）
    ├─ REALITY 握手与认证
    ├─ VLESS UUID 用户识别
    └─ freedom 出站 ──► 目标网站
```

- **VLESS**：标识客户端用户；建议每台设备使用独立 UUID。
- **REALITY**：提供连接的安全握手与目标伪装；服务端保管 PrivateKey，客户端使用对应 PublicKey。
- **Vision**：使用 `xtls-rprx-vision` Flow，服务端与客户端必须一致。
- **目标域名**：REALITY 的 `target`/`serverNames` 不等于用户访问的最终网站。本文以 `www.microsoft.com` 作为示例，部署前应自行验证其可达性和适用性。
- **出站**：默认通过 `freedom` 由 VPS 访问实际目标网站。

## 2. 环境要求

| 组件 | 示例 / 要求 |
| --- | --- |
| 服务端 | Ubuntu 24.04 LTS，具备公网 IPv4 |
| 服务端地址 | `203.0.113.10`（RFC 5737 文档示例地址，不能实际连接） |
| 入站端口 | TCP `51120`（示例，可自行调整） |
| Xray-core | 使用官方发布的稳定版本，记录两端实际版本 |
| Windows 客户端 | v2rayN，选择 Xray-core |
| REALITY 目标 | `www.microsoft.com:443` |
| REALITY SNI | `www.microsoft.com` |
| 传输 | `raw`（部分旧界面可能显示 `tcp`） |

**注意：** 端口号只是部署示例，并非 REALITY 的固定要求。服务端和客户端必须使用同一端口。

## 3. Ubuntu 服务端安装

### 3.1 系统准备

```bash
sudo apt update
sudo apt install -y curl unzip ca-certificates openssl jq
```

确认系统和架构：

```bash
cat /etc/os-release
uname -m
```

### 3.2 安装 Xray-core

参考官方项目 [XTLS/Xray-core](https://github.com/XTLS/Xray-core) 和 [XTLS/Xray-install](https://github.com/XTLS/Xray-install)。安装前应核对官方脚本来源和发布说明，不要盲目执行来源不明的脚本。

官方安装脚本示例：

```bash
curl -fsSLo /tmp/install-release.sh \
  https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh
less /tmp/install-release.sh
sudo bash /tmp/install-release.sh install
```

安装后检查：

```bash
/usr/local/bin/xray version
systemctl status xray --no-pager
systemctl show xray -p ExecStart
```

不同安装方式可能使用不同路径。下文采用常见的：

- 程序：`/usr/local/bin/xray`
- 配置：`/usr/local/etc/xray/config.json`
- 服务：`xray.service`

### 3.3 生成 VLESS UUID 和 REALITY 密钥

分别生成两个客户端 UUID：

```bash
/usr/local/bin/xray uuid
/usr/local/bin/xray uuid
```

生成 REALITY X25519 密钥对：

```bash
/usr/local/bin/xray x25519
```

输出字段名称可能随版本变化，例如 `PrivateKey` / `Password` 和 `PublicKey`。**必须以当前版本命令的实际输出和官方说明为准**，不要把私钥填入客户端。

生成 Short ID：

```bash
openssl rand -hex 8
```

将真实的 UUID、PrivateKey、PublicKey 和 Short ID 保存到安全的密码管理器，不要提交到 Git 仓库。

### 3.4 编写服务端配置

下面是结构示例。**必须替换所有尖括号占位符**，尤其是 UUID、PrivateKey 和 Short ID。本文使用两个用户，保留独立身份。

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "vless-reality-in",
      "listen": "0.0.0.0",
      "port": 51120,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<CLIENT_A_UUID>",
            "flow": "xtls-rprx-vision"
          },
          {
            "id": "<CLIENT_B_UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "raw",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "target": "www.microsoft.com:443",
          "xver": 0,
          "serverNames": ["www.microsoft.com"],
          "privateKey": "<REALITY_PRIVATE_KEY>",
          "shortIds": ["<REALITY_SHORT_ID>"]
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom"
    },
    {
      "tag": "block",
      "protocol": "blackhole"
    }
  ]
}
```

> 兼容性提示：不同 Xray-core 版本对 `target` / `dest` 和 `raw` / `tcp` 的支持与显示可能不同。以目标版本的官方文档及 `xray run -test` 为准，不要直接照搬不兼容版本的字段。

备份已有配置后再写入：

```bash
sudo cp -a /usr/local/etc/xray/config.json \
  /root/xray-config.backup.json
sudoedit /usr/local/etc/xray/config.json
```

如果是全新安装且文件不存在，可跳过备份。

校验并重启：

```bash
sudo /usr/local/bin/xray run -test \
  -config /usr/local/etc/xray/config.json
sudo systemctl restart xray
sudo systemctl status xray --no-pager
sudo ss -lntp | grep ':51120'
```

仅在配置测试成功后重启。需要在 VPS 服务商安全组和 Ubuntu 防火墙中允许 TCP 51120，防火墙规则应符合实际环境；不要随意关闭防火墙。

## 4. Windows v2rayN 客户端配置

在 [v2rayN 官方 Releases](https://github.com/2dust/v2rayN/releases) 下载适合 Windows 的版本，配置或选择 Xray-core。

新增 VLESS 节点，填写：

| 字段 | 填写内容 |
| --- | --- |
| 地址 | `<SERVER_PUBLIC_IP>` |
| 端口 | `51120` |
| 用户 ID | `<CLIENT_A_UUID>` |
| Flow | `xtls-rprx-vision` |
| 传输协议 | `raw` / 对应版本中的 TCP |
| 传输安全 | `reality` |
| SNI / ServerName | `www.microsoft.com` |
| Fingerprint | `chrome` |
| PublicKey | `<REALITY_PUBLIC_KEY>` |
| Short ID | `<REALITY_SHORT_ID>` |

第二台设备使用 `<CLIENT_B_UUID>`，其他共享参数可以相同。客户端使用的是 **PublicKey**，服务端使用的是 **PrivateKey**。

保存后选择新节点，启动 Xray-core，并确认本地 HTTP 代理端口。以下测试假设端口为 `10808`；若实际端口不同，请相应替换。

## 5. 部署验收

### 5.1 服务端自检

```bash
/usr/local/bin/xray version
sudo /usr/local/bin/xray run -test \
  -config /usr/local/etc/xray/config.json
systemctl status xray --no-pager
ss -lntp | grep ':51120'
```

服务器直连目标网站测试：

```bash
curl -4 -v --connect-timeout 10 --max-time 20 \
  https://api.ipify.org/
curl -4 -v --connect-timeout 10 --max-time 20 \
  https://www.google.com/
```

这些命令仅验证 **VPS 自身出站**，不验证客户端代理链路。

### 5.2 Windows 通过代理测试

PowerShell：

```powershell
curl.exe -4 -v --connect-timeout 10 --max-time 30 `
  -x http://127.0.0.1:10808 `
  https://api.ipify.org/
```

```powershell
curl.exe -4 -v --connect-timeout 10 --max-time 30 `
  -x http://127.0.0.1:10808 `
  https://www.google.com/
```

**验收要点：**

1. `HTTP/1.1 200 Connection established` 只代表本地 HTTP CONNECT 隧道建立，**不等于目标 HTTPS 请求成功**。
2. 应继续看到 TLS 握手完成、目标网站返回 HTTP 状态，以及实际响应内容。
3. `api.ipify.org` 返回的出口 IP 应与预期 VPS 出口一致；注意服务器可能有不同的出口地址。
4. 浏览器实际访问测试要单独进行，不能仅凭 curl 判定所有网页都可用。

### 5.3 稳定性测试

PowerShell 连续进行 5 轮：

```powershell
1..5 | ForEach-Object {
    Write-Host "第 $_ 轮"
    curl.exe -4 -sS -o NUL `
      -w 'Google HTTP=%{http_code} time=%{time_total}s`n' `
      --connect-timeout 10 --max-time 30 `
      -x http://127.0.0.1:10808 https://www.google.com/
    curl.exe -4 -sS -o NUL `
      -w 'ipify HTTP=%{http_code} time=%{time_total}s`n' `
      --connect-timeout 10 --max-time 30 `
      -x http://127.0.0.1:10808 https://api.ipify.org/
}
```

如果出现间歇性超时，不应因为某一次 HTTP 200 就宣布部署完全稳定。

## 6. 故障排查：按链路逐层定位

### 6.1 端口无法连接

服务端检查：

```bash
systemctl status xray --no-pager
ss -lntp | grep ':51120'
sudo journalctl -u xray -n 100 --no-pager
```

Windows 检查：

```powershell
Test-NetConnection <SERVER_PUBLIC_IP> -Port 51120
```

检查 VPS 安全组、Ubuntu 防火墙、监听地址及公网路由。

### 6.2 REALITY 握手失败

典型日志：

```text
hs.c.isHandshakeComplete.Load(): false
handshake did not complete successfully
```

逐项核对：

- 客户端地址和端口是否正确；
- 客户端 SNI 是否在服务端 `serverNames` 中；
- 客户端 PublicKey 是否由当前服务端 PrivateKey 推导；
- 客户端 Short ID 是否包含在服务端 `shortIds` 中；
- 客户端与服务端 Flow 是否匹配；
- 客户端时间是否明显偏离；
- 目标站点的 TLS 行为是否与当前 Xray-core 版本兼容。

握手日志中的 `ClientShortId` 可能是十进制字节数组，需转换成十六进制后再比较。

**版本兼容性案例：** 部分旧版本对目标 TLS Certificate 记录的处理可能存在限制。如果日志持续出现大证书记录及握手失败，应查阅 Xray 官方更新记录，确认是否存在对应修复。升级前先备份程序和配置，不要把所有握手失败都归因于版本。

### 6.3 REALITY 成功，但目标网站仍超时

成功握手常见日志：

```text
hs.handshake() err: <nil>
hs.readClientFinished() err: <nil>
hs.c.isHandshakeComplete.Load(): true
accepted tcp:<TARGET_DOMAIN>:443
```

其中 `accepted tcp` 表示请求进入代理处理流程，**并不能证明最终网站已成功响应**。

同步查看服务端日志：

```bash
sudo journalctl -u xray -f -o cat
```

另开终端观察入站 TCP：

```bash
sudo tcpdump -ni any 'tcp port 51120' -tttt
```

然后 Windows **只执行一次** curl 测试，对齐时间和源端口，判断：

| 现象 | 下一步 |
| --- | --- |
| 服务端没有新 TCP 连接 | 查客户端路由、本地代理和公网链路 |
| TCP 到达但握手失败 | 查 REALITY 参数、时间和版本兼容性 |
| 握手成功但没有目标请求 | 查 VLESS 用户、路由与客户端实际请求 |
| 已有目标请求但响应超时 | 查 VPS 出站、DNS、回程链路及目标站点 |
| 多目标间歇性超时 | 对比成功与失败的抓包、重传、RST 和耗时 |

注意：TCP SYN/ACK 只代表传输层建连过程，不等于应用层成功。

### 6.4 Windows 证书吊销检查失败

Windows curl 可能返回：

```text
CRYPT_E_REVOCATION_OFFLINE (0x80092013)
```

这意味着 Schannel 无法完成证书吊销状态查询，不等于证书一定被吊销。仅用于诊断时，可以临时尝试：

```powershell
curl.exe -4 -v --ssl-no-revoke `
  -x http://127.0.0.1:10808 https://www.google.com/
```

`--ssl-no-revoke` 会降低证书状态验证的安全性，不应作为长期解决方案；应进一步检查系统时间、代理路径和 CRL/OCSP 相关网络访问。

### 6.5 ChatGPT 页面加载慢或返回 403

- `curl` 返回 `403` 可能是站点访问控制或浏览器验证挑战，不能直接证明 REALITY 故障。
- 先验证 Google、ipify 是否持续稳定，再单独排查 ChatGPT。
- 浏览器打开开发者工具 → Network，记录失败域名、状态码、耗时和错误类型。
- 不要公开 Cookie、Authorization、会话令牌或完整请求头。

## 7. 日志管理与安全维护

常规运行建议：

```json
"log": { "loglevel": "warning" }
```

REALITY `show` 为独立开关，常规运行建议 `false`；临时排障时再开启，完成后关闭。日志可能暴露连接元数据，发布截图前应脱敏。

**生产运维建议：**

- 每台客户端使用独立 UUID；注销设备时只撤销对应 UUID。
- 配置文件、密钥和备份文件限制访问权限。
- 不要把真实 `config.json`、订阅链接、二维码或访问日志提交到 GitHub。
- 如果私钥曾公开，应安排轮换，且同步更新所有受影响客户端的 PublicKey。
- 变更前备份；变更后执行 `xray run -test`，确认服务正常再做回归测试。
- 使用 `.gitignore` 忽略本地敏感文件：

```gitignore
config.json
*.key
*.pem
*.env
.env*
secrets/
backups/
logs/
```

> `.gitignore` 只能防止尚未跟踪的文件被新增；如果秘密已经进入 Git 历史，仍需轮换密钥并处理历史记录。

## 8. 验收清单

- [ ] 服务端 Xray 版本、路径与启动配置已确认
- [ ] `xray run -test` 通过
- [ ] TCP 入站端口监听正常，安全组规则正确
- [ ] 每个客户端有独立 UUID
- [ ] PublicKey、Short ID、SNI、Flow 对应正确
- [ ] Windows 经代理获得预期出口 IP
- [ ] Google 和 ipify 多轮测试成功率达到要求
- [ ] 浏览器实际访问通过
- [ ] 排障日志关闭或恢复到合理级别
- [ ] 配置、密钥和日志未提交到公开仓库

## 9. 参考项目

- [XTLS/Xray-core](https://github.com/XTLS/Xray-core)
- [XTLS/Xray-install](https://github.com/XTLS/Xray-install)
- [2dust/v2rayN](https://github.com/2dust/v2rayN)

---

**结语：** 部署成功要同时满足服务可启动、REALITY 握手成功、代理目标可访问和多轮稳定性测试。单次 `CONNECT 200` 或 `isHandshakeComplete: true` 不能替代端到端验收。
