# mcp-local-search-integration

> 在 Windows 10 LTSC + WSL2 (Ubuntu 24.04) 上，把一个只支持 stdio 的免费搜索 MCP，做成**开机自启、断线自愈、零成本**的常驻 HTTP 服务，同时供 Dify 和 DeepSeek Harness 调用。

## 这是什么

这是我在自己那台 **2 核 4 线程、16GB 内存、没有独显**的老笔记本上，把本地免费搜索服务 `ai-search-mcp` 通过 `supergateway` 桥接成 HTTP/StreamableHTTP 端点，并让它**开机自动运行、崩溃自动恢复**的完整操作记录。

内容涵盖：从最朴素的 `nohup npx` 起步，一路踩到 WSL 发行版空闲被回收，最后收敛到 **systemd 系统服务 + 分离式容器 + 独立看门狗 + Windows 侧常驻会话** 这套能真正无人值守的方案。

如果你也遇到"MCP 服务隔一会儿就断""重启电脑后要手动开一遍""客户端报 unknown tool"这类问题，这份文档里的**坑 5、坑 7、坑 8、坑 12** 大概率就是你要找的答案。

## 解决了什么问题

- **搜索服务按次计费**：用免费的 `ai-search-mcp` 替代 Tavily 等按次计费的搜索 API，实现零成本联网搜索。
- **MCP 协议不兼容**：`ai-search-mcp` 只支持 stdio 模式，通过 `supergateway` 桥接为 StreamableHTTP，Dify 和 DeepSeek Harness 都能调用。
- **一次配置、多端复用**：同一套服务同时供 Dify 和 DSH 使用，无需重复部署。
- **国内网络直连**：通过 `SEARCH_REGION=cn-zh` 与国内引擎链，把搜索耗时从 45 秒压到 3～5 秒，无需代理。
- **开机自启 + 无人值守**：Windows 登录即自动拉起，服务崩溃后 **8 秒内自愈**，夜里也能自己修好。
- **老硬件可用**：全程 CPU-only，16GB 内存足够跑，不需要显卡。

## 问题总清单

我把整个过程中遇到的全部 13 个问题先列在这里。**建议先扫这张表**——如果你的症状能对上，直接跳到对应的「踩坑详解」小节。

| # | 问题 | 现象 | 根因 | 解决 |
|---|---|---|---|---|
| 1 | ai-search-mcp 无法直接监听 HTTP | 进程在跑，但 `ss` 查不到端口，curl 拒绝连接 | 它是 stdio 模式，不支持 HTTP | 用 supergateway 桥接为 HTTP |
| 2 | 容器每次启动都要装依赖 | 容器启动慢，网络一抖就以 exit 126 死亡，报 `supergateway: Permission denied` | 每次启动都执行 `npm install -g` + `npx` 下载 | 构建预装依赖的本地镜像 |
| 3 | WSL 里 crun 不可用 | 容器起不来，报 `unknown version specified` | 该 WSL 内核与 crun 不兼容 | 全局切到 runc |
| 4 | rootless podman + 用户级 systemd 不稳定 | 容器被反复杀死，PID1 自己变来变去 | 用户级 systemd 在 WSL 下生命周期不可靠 | 改用**系统级** systemd + rootful podman |
| 5 | 裸 TCP 探活把服务探死 | 做一次端口探测，服务就崩一次 | supergateway 4.1.0 对中断的原始连接会抛异常 | 改用规范 HTTP `initialize` 探活 |
| 6 | supergateway 随客户端断开而退出 | DSH 发完请求就报 Connection error | 无状态模式下请求结束即退出 | 容器内加守护循环，退出后 1 秒内重启 |
| 7 | 服务启动约 19 秒后被 systemd 停掉 | `Started` 后紧接着 `Stopping` | 前台 `podman run` 当主进程，一退出整个单元就停 | 改成**分离式**（`podman run -d`）+ 独立看门狗 |
| 8 | **MCP 每隔一会儿就断**（最坑） | 稳定运行几十秒后突然不通，反复如此 | **WSL 空闲会自动回收发行版**，连带杀死 systemd 和容器；Windows 侧发来的 HTTP 流量**不算 WSL 活动** | 登录时保留一个常驻 WSL 会话 |
| 9 | 保活会话启动即退出 | 会话进程刚起就没了 | PowerShell 把带空格的参数拆散了 | 参数去掉空格，用 `sleep 999999999` |
| 10 | `.bashrc` 自动启动导致 WSL 登录报错 | 一进 WSL 就报 `unexpected EOF while looking for matching ')'` | 写进 `.bashrc` 的片段有语法错误 | 删掉该片段，改用 Windows 侧引导 |
| 11 | 中文搜索 45 秒超时 | 搜索经常卡死或空结果 | 默认引擎走了 DuckDuckGo / Google，国内不可达 | 指定国内引擎链 + 缩短超时 |
| 12 | DSH 显示 unknown tool | 工具调用报 `unknown tool`，重启也没用 | `failOnStartupError: true`，启动瞬间失败就永久摘掉工具 | 改为 `false` + 开启 reconnect |
| 13 | WSL 重启后客户端配置失效 | 昨天还好好的，今天连不上 | WSL IP 是临时的，重启会变 | 引导脚本检测并告警，手工更新 URL |

**归纳一下这 13 个坑的层次**，其实只有三件事：

1. **协议层**（坑 1、6）：stdio 要桥接、桥接进程要活得久。
2. **运行时层**（坑 2、3、4、7、9、10）：容器怎么起、谁来管、用什么运行时。
3. **宿主层**（坑 5、8、11、12、13）：WSL 的生命周期、探活方式、网络可达性、客户端容错——**这一层最难，因为前 12 个坑里有一半是它伪装出来的**。

## 最终架构

```
┌────────────────────────────────────────────────────────────────┐
│ Windows 10 LTSC                                                │
│                                                                │
│  Startup 文件夹 ──▶ mcp_autostart.ps1                          │
│    (VBS 静默调用)      ├─ 唤醒 WSL                             │
│                        ├─ 确保 systemd 单元在跑                │
│                        ├─ 拉起常驻会话（保 WSL 不被回收）★     │
│                        ├─ 检测 WSL IP 是否变化                 │
│                        └─ HTTP initialize 健康检查             │
│                                                                │
│  DSH ──▶ http://<WSL_IP>:11000/mcp                        │
└────────────────────────────┬───────────────────────────────────┘
                             │ WSL2 (Ubuntu 24.04)
┌────────────────────────────▼───────────────────────────────────┐
│ systemd (系统级)                                               │
│   mcp-search.service          —— 分离式容器（podman run -d）   │
│   mcp-search-watchdog.timer   —— 每 2 分钟 HTTP 探活 ★         │
│                                                                │
│   ┌──────────────────────────────────────────┐                 │
│   │ 容器 dify-mcp-search (localhost/…:local) │                 │
│   │   sh -c "while true; do                   │                 │
│   │      supergateway --stdio ai-search-mcp   │                 │
│   │        --outputTransport streamableHttp   │                 │
│   │        --port 11000; sleep 1; done"       │  ← 守护循环 ★   │
│   └──────────────────┬───────────────────────┘                 │
│                      │ stdio                                   │
│              ┌───────▼────────┐                                │
│              │ ai-search-mcp  │  免费引擎链                     │
│              └────────────────┘  so360→bing→baidu→sogou        │
│                                                                │
│   Dify (容器) ──▶ host.containers.internal:11000/mcp           │
│                   （注意是 /mcp，不是 /sse）                    │
└────────────────────────────────────────────────────────────────┘
```

★ 的四处，是这个方案能"无人值守"的关键，也是我踩坑最多的地方。

## 数据来源说明

为了不误导后来者，我标明本文数据的来源：

- **实测数据**（我在自己机器上跑出来的）：版本号、镜像大小、`/mcp` 与 `/sse` 的端点行为、连续性 12/12、看门狗自愈日志、服务状态、主机配置。这些都经过当次实际执行验证。
- **过程记录**（来自我当时的问题追踪记录）：各坑的报错原文与排查思路。报错信息是当时抄录的，个别机器专属细节（如绝对路径）请按自己的环境替换。
- **Dify 接入步骤**：写本文时我的 Dify 栈处于停止状态（我按需手动启动），所以该节的界面操作来自我的配置记录，**未经当次实测**；但端点路径 `/mcp` 是我当场实测确认的。

## 环境准备

我实测所用的环境：

- **操作系统**：Windows 10 IoT 企业版 LTSC 21H2
- **WSL2**：Ubuntu 24.04 LTS
- **Node.js**：v20.20.2（WSL2 内）
- **容器运行时**：Podman 6.1.2 + runc 1.4.3（**不用 crun**，原因见坑 3）
- **硬件**：2 核 4 线程 CPU、16GB 内存、无独显
- **客户端**：Dify（容器方式运行）、DeepSeek Harness（Windows 侧）

> 说明：这是一台有些年头的老笔记本。**它能跑，说明这套方案对硬件没有苛刻要求。**

## 部署步骤

下面是我最终收敛出来的步骤，可以照抄。每一步都说明「为什么这么做」。

### 1. WSL2 安装 Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # 应显示 v20.x
```

### 2. 解决 crun 不可用（很重要，先做）

我这台 WSL 上 crun 直接罢工，容器根本起不来。全局切到 runc：

```bash
sudo tee /etc/containers/containers.conf > /dev/null << 'EOF'
[engine]
runtime = "/usr/bin/runc"
EOF
```

确认：

```bash
runc --version   # runc version 1.4.3
```

> 如果跳过这步，后面每个容器操作都会报 `unknown version specified`。

### 3. 构建预装依赖的本地镜像

**不要**在容器启动时跑 `npm install`（原因见坑 2）。先构建一个装好依赖的镜像：

`mcp-search.Containerfile`：

```dockerfile
FROM docker.io/library/node:20-slim
RUN npm install -g --no-fund --no-audit supergateway ai-search-mcp
```

构建：

```bash
sudo podman build -t localhost/dify-mcp-search:local \
  -f /home/<用户名>/dify/docker/mcp-search.Containerfile /home/<用户名>/dify/docker/
```

我的镜像最终 **318 MB**，启动是离线可用的。

### 4. 写 systemd 系统级单元（分离式容器）

**关键点**：用 `podman run -d` 让容器**脱离** systemd 的主进程，而不是让 `podman run` 当前台进程（原因见坑 7）。

`/etc/systemd/system/mcp-search.service`：

```ini
[Unit]
Description=dify-mcp-search MCP server (supergateway + ai-search-mcp, streamableHttp, stateless, detached)
StartLimitIntervalSec=0

[Service]
Type=oneshot
RemainAfterExit=yes
Delegate=yes
ExecStartPre=-/usr/local/bin/podman rm -f dify-mcp-search
ExecStart=/usr/local/bin/podman run -d --runtime /usr/bin/runc --network host --name dify-mcp-search \
  -e NODE_ENV=production \
  -e SEARCH_REGION=cn-zh \
  -e SEARCH_LANG_ROUTING=1 \
  -e SEARCH_ENGINE=so360,bing,baidu,sogou \
  -e SEARCH_TIMEOUT_MS=8000 \
  -e SEARCH_CACHE_TTL=600 \
  localhost/dify-mcp-search:local \
  sh -c "while true; do supergateway --stdio 'ai-search-mcp' --port 11000 --outputTransport streamableHttp --host 0.0.0.0; echo '[supervisor] restart in 1s'; sleep 1; done"
ExecStop=/usr/local/bin/podman stop -t 10 dify-mcp-search

[Install]
WantedBy=multi-user.target
```

三个要点：

- `--network host`：容器直接用 WSL 的网络，Windows 侧才能从 WSL IP 访问到 11000。
- `while true; ... sleep 1`：**容器内的守护循环**。supergateway 退出后 1 秒内自动重启（坑 6）。
- `podman run -d` + `Type=oneshot` + `RemainAfterExit=yes`：容器与 systemd 主进程解耦（坑 7）。

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mcp-search.service
sudo systemctl status mcp-search.service
```

### 5. 写独立看门狗（每 2 分钟 HTTP 探活）

容器会活，但**发行版重启后不会自动回来**，所以加一层外部检查。

`/home/<用户名>/scripts/mcp_watchdog.sh`：

```bash
#!/bin/bash
LOG=/home/<用户名>/mcp_watchdog.log
ts() { date "+%Y-%m-%d %H:%M:%S"; }
probe() {
  curl -s -m 10 -o /dev/null -w '%{http_code}' -X POST http://127.0.0.1:11000/mcp \
    -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
    -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"watchdog","version":"1"}}}' 2>/dev/null
}
code=$(probe)
[ "$code" = "200" ] && exit 0
echo "[$(ts)] MCP unhealthy (http=$code) -> systemctl restart mcp-search.service" >> "$LOG"
systemctl restart mcp-search.service >> "$LOG" 2>&1
sleep 6
echo "[$(ts)] after restart http=$(probe)" >> "$LOG"
```

> ⚠️ **探活必须用 HTTP POST**，绝对不能用"连一下 TCP 就断开"的方式。这一点我付出了代价，详见坑 5。

配套的 service + timer：

`/etc/systemd/system/mcp-search-watchdog.service`：

```ini
[Unit]
Description=MCP search watchdog (HTTP health probe, restart if unhealthy)

[Service]
Type=oneshot
ExecStart=/home/<用户名>/scripts/mcp_watchdog.sh
```

`/etc/systemd/system/mcp-search-watchdog.timer`：

```ini
[Unit]
Description=MCP search watchdog timer

[Timer]
OnActiveSec=1min
OnUnitActiveSec=2min
OnCalendar=*:0/2
Persistent=true

[Install]
WantedBy=timers.target
```

启用：

```bash
chmod +x /home/<用户名>/scripts/mcp_watchdog.sh
sudo systemctl daemon-reload
sudo systemctl enable --now mcp-search-watchdog.timer
```

### 6. Windows 侧引导脚本（含保活会话）

这是**整个方案最关键的一环**（坑 8）。Windows 侧脚本要做五件事：唤醒 WSL → 确保单元在跑 → **保留一个常驻会话** → 检测 IP → 健康检查。

`C:\Users\<用户名>\AppData\Local\mcp_autostart.ps1` 核心部分：

```powershell
# 请按自己的环境替换这三个值：
#   $wslUser    —— 你的 WSL 用户名（用 wsl -d Ubuntu -e whoami 查）
#   $expectedIp —— 首次运行后从日志里抄下实际 IP
#   $logFile    —— 换成你的 Windows 用户目录
$wslDistro  = "Ubuntu"
$wslUser    = "你的WSL用户名"
$expectedIp = "<WSL_IP>"
$logFile    = "C:\Users\<用户名>\AppData\Local\mcp_autostart.log"

function Write-Log($m) {
  Add-Content -Path $logFile -Value ("[" + (Get-Date -Format "yyyy-MM-dd HH:mm:ss") + "] " + $m)
}

function Test-McpHttp($ip) {
  # 必须用规范的 MCP initialize over HTTP，返回 200 即视为健康
  try {
    $req = [System.Net.HttpWebRequest]::Create("http://" + $ip + ":11000/mcp")
    $req.Method = "POST"
    $req.ContentType = "application/json"
    $req.Accept = "application/json, text/event-stream"
    $req.Timeout = 15000
    $body = '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"win-bootstrap","version":"1"}}}'
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
    $req.GetRequestStream().Write($bytes, 0, $bytes.Length)
    $resp = $req.GetResponse()
    $sr = New-Object System.IO.StreamReader($resp.GetResponseStream())
    $null = $sr.ReadToEnd(); $sr.Close()
    $code = [int]$resp.StatusCode; $resp.Close()
    return ($code -eq 200)
  } catch { return $false }
}

# 1) 唤醒 WSL（最多 12 x 5s = 60s）
$ok = $false
for ($i = 1; $i -le 12; $i++) {
  $null = wsl -d $wslDistro -- echo ok 2>$null
  if ($LASTEXITCODE -eq 0) { $ok = $true; break }
  Start-Sleep -Seconds 5
}
if (-not $ok) { Write-Log "WSL $wslDistro not reachable; bootstrap skipped"; exit 0 }

# 2) 确保系统级单元在跑
$null = wsl -d $wslDistro -u $wslUser -e bash -c "sudo -n systemctl start mcp-search.service" 2>&1
Write-Log "WSL up; mcp-search.service start requested"

# 3) ★ 保活会话：WSL 空闲会回收发行版，连带杀死 systemd 和容器
#    Windows 侧发往 WSL IP 的 HTTP 流量不算 WSL 活动，所以必须挂一个常驻会话。
#    参数里刻意不含空格，否则 PowerShell 会把参数拆散（坑 9）。
$keeperMarker = "999999999"
$keeper = $null
try {
  $keeper = Get-CimInstance Win32_Process -Filter "Name='wsl.exe'" -ErrorAction SilentlyContinue |
            Where-Object { $_.CommandLine -like ("*" + $keeperMarker + "*") } |
            Select-Object -First 1
} catch { }
if ($keeper) {
  Write-Log ("keeper session already running (pid " + $keeper.ProcessId + ")")
} else {
  Start-Process -FilePath "wsl.exe" `
    -ArgumentList @("-d", $wslDistro, "-u", $wslUser, "--", "sleep", $keeperMarker) `
    -WindowStyle Hidden | Out-Null
  Start-Sleep -Seconds 2
  $check = Get-CimInstance Win32_Process -Filter "Name='wsl.exe'" -ErrorAction SilentlyContinue |
           Where-Object { $_.CommandLine -like ("*" + $keeperMarker + "*") } |
           Select-Object -First 1
  if ($check) {
    Write-Log ("keeper session started (pid " + $check.ProcessId + ")")
  } else {
    Write-Log "WARNING: keeper session failed to start; WSL may idle out and stop the MCP"
  }
}

# 4) 解析当前 WSL IP，变了就告警（客户端配置需要同步更新）
$ip = $null
try {
  $raw  = (wsl -d $wslDistro -e bash -c "hostname -I" 2>$null | Out-String)
  $cand = ($raw -split '\s+') | Where-Object { $_ -match '^\d{1,3}(\.\d{1,3}){3}$' } | Select-Object -First 1
  if ($cand) { $ip = $cand }
} catch { }
if (-not $ip) { $ip = $expectedIp }

if ($ip -ne $expectedIp) {
  Write-Log "WARNING: WSL IP changed - expected $expectedIp but got $ip; update DSH + Cherry Studio MCP url"
} else {
  Write-Log "WSL IP unchanged ($ip)"
}

# 5) 等 MCP 真正应答（最多 24 x 5s = 120s）
$up = $false
for ($i = 1; $i -le 24; $i++) {
  if (Test-McpHttp $ip) { $up = $true; break }
  Start-Sleep -Seconds 5
}
Write-Log ("bootstrap done, MCP HTTP initialize OK on " + $ip + ":11000 = " + $up)
```

### 7. 让脚本在登录时静默运行

在启动文件夹放置一个 VBS，避免弹出黑窗口：

路径：`C:\Users\<用户名>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\MCP-Autostart-DifyMcpSearch.vbs`

```vbscript
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run "powershell -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File ""C:\Users\<用户名>\AppData\Local\mcp_autostart.ps1""", 0, False
```

**撤销方法**：删掉这个 VBS 即可，不影响其他部分。

### 8. 接入 Dify

> ⚠️ **端点是 `/mcp`，不是 `/sse`。**
>
> 我实测过：`POST /mcp` 返回 **200**，`POST /sse` 返回 **404**。网上不少教程（包括我自己早期的笔记）写的是 `/sse`，那是 SSE 传输时代的旧端点。现在用的是 `--outputTransport streamableHttp`，路径是 **`/mcp`**。
>
> 另外说明：写这份文档时我的 Dify 栈处于停止状态（我平时按需手动启动），所以下面的步骤来自我的配置记录，**步骤本身未经当次实测**。端点路径则以实测为准。

- 进 Dify 工作空间 → 工具 → MCP → 添加 MCP 服务器（HTTP）
- **服务端点 URL**：`http://host.containers.internal:11000/mcp`
- **名称 / 标识符**：`free-web-search`
- 认证留空，动态客户端注册关闭
- 点「添加并授权」

成功后 Dify 会列出 4 个工具：`search`、`fetch_page`、`research`、`status`。

**如果连不上**，先按顺序查三件事：

```bash
sudo podman ps | grep dify-mcp-search    # 1. MCP 容器在跑吗
ss -lnt | grep 11000                     # 2. 端口在听吗
curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  http://127.0.0.1:11000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}'
```

### 9. 接入 DeepSeek Harness

配置文件：`C:\Users\<用户名>\.dsh\mcp-servers\free-search\cordis.patch.yml`

```json
[
  {
    "insert": [
      {
        "id": "mcp-manager-free-search",
        "name": "@deepseek-ai/dsh-mcp-client",
        "config": {
          "serverName": "free-search",
          "transport": "streamable-http",
          "url": "http://<WSL_IP>:11000/mcp",
          "failOnStartupError": false,
          "reconnect": {
            "enabled": true,
            "initialDelayMs": 500,
            "maxDelayMs": 30000,
            "maxAttempts": 10
          }
        }
      }
    ]
  }
]
```

两个**必须**注意的点：

- `failOnStartupError: false` —— 否则启动瞬间连不上就永久摘掉工具（坑 12）。
- `reconnect` 必须开启 —— 服务重启后客户端能自己接回来。

保存后**完全退出 DSH 再重开**。

## 踩坑详解

下面按"症状 → 排查 → 根因 → 修复"记录每个坑的完整过程。**坑 5、7、8 是最难查的，因为它们的表象完全一样**（都是"服务莫名其妙不通了"），但根因完全不同。

### 坑 1：ai-search-mcp 无法直接监听 HTTP

**症状**：进程明明在跑，`ss -tln | grep 11000` 什么都没有，curl 直接拒绝连接。

**排查**：看它的文档，发现是 stdio 模式的 MCP server——通过标准输入输出通信，压根没有网络监听这一说。

**根因**：MCP 协议本身有 stdio / SSE / StreamableHTTP 多种传输方式，`ai-search-mcp` 只实现了 stdio。

**修复**：用 `supergateway` 做协议桥接，把 stdio 转成 HTTP。

### 坑 2：容器每次启动都要装依赖

**症状**：容器启动很慢；网络一抖就以 `exit 126` 死亡，日志里是 `supergateway: Permission denied`。

**排查**：查看启动命令，发现每次都在执行 `npm install -g supergateway ai-search-mcp`，然后用 `npx` 拉起。某次安装不完整，`supergateway` 文件权限不对，就再也起不来了。

**根因**：把"安装"和"运行"混在了启动阶段。容器应该是不可变的、离线可启动的。

**修复**：写 `Containerfile` 预装依赖，构建成 `localhost/dify-mcp-search:local`（318 MB），启动时直接调用二进制，**不再联网、不再安装**。

### 坑 3：WSL 里 crun 不可用

**症状**：容器起不来，报 `unknown version specified`。

**排查**：这个报错很含糊，不像常规的镜像或权限问题。逐个排除后发现是 OCI 运行时本身的问题——这台 WSL 的内核与 crun 版本不兼容。

**根因**：Podman 默认用 crun，但该 WSL 环境下 crun 无法正常工作。

**修复**：全局切到 runc。

```bash
sudo tee /etc/containers/containers.conf > /dev/null << 'EOF'
[engine]
runtime = "/usr/bin/runc"
EOF
```

此后每个容器操作都稳定了。**这个坑很值得记**——报错信息完全不指向真正的原因。

### 坑 4：rootless podman + 用户级 systemd 不稳定

**症状**：容器被反复杀死，用户级 systemd 的 PID1 自己变来变去。

**排查**：一开始我按"最佳实践"用 rootless podman + `systemctl --user`。但在 WSL 里，用户级 systemd 的生命周期很不可靠——WSL 的会话管理会干扰它。

**根因**：WSL 下用户级 systemd 的存活语义与常规 Linux 发行版不同。

**修复**：改用**系统级** systemd + rootful podman。这是让服务真正稳定的转折点。

### 坑 5：裸 TCP 探活把服务探死

**症状**：只要做一次端口探测，服务就崩一次。这个现象极具迷惑性——看起来像"服务自己不稳定"。

**排查**：我原本用"连一下 TCP 就断开"的方式做健康检查。发现**每次探测后服务都异常退出**，日志里是 supergateway 抛出的连接异常。

**根因**：supergateway 4.1.0 对"被中断的原始 TCP 连接"处理有缺陷，会直接抛异常导致进程退出。

**修复**：健康检查改用**规范的 MCP `initialize` HTTP POST**，完整读写响应后正常关闭。此后探测不再影响服务。

> **教训**：健康检查也是一种"流量"，必须走协议正路。

### 坑 6：supergateway 随客户端断开而退出

**症状**：DSH 发完请求就报 Connection error。

**排查**：观察到 supergateway 有两种模式——stateful 状态在客户端断开时会崩，stateless 则处理完一个请求就退出。

**根因**：这两个模式各有毛病，没有一个是开箱即用的常驻服务。

**修复**：**不指望它自己活着，用外层循环兜住**。在容器启动命令里套一个守护循环：

```bash
sh -c "while true; do
  supergateway --stdio 'ai-search-mcp' --port 11000 --outputTransport streamableHttp --host 0.0.0.0
  echo '[supervisor] restart in 1s'
  sleep 1
done"
```

它退出后 1 秒内自动重启。**用"容忍崩溃"代替"避免崩溃"**，这是这个方案的一个重要思路转变。

### 坑 7：服务启动约 19 秒后被 systemd 停掉

**症状**：`journalctl` 里看到 `Started` 之后紧接着 `Stopping`，间隔只有十几秒。

**排查**：日志时间线非常清楚：

```
HH:MM:SS Started mcp-search.service
...       (约 19 秒后)
        Stopping mcp-search.service
```

**根因**：我最初让 `podman run`（前台模式）作为 systemd 的主进程。主进程一退出，systemd 就认为服务结束了，于是停掉/重启整个单元。而 `podman run` 前台模式在某些情况下会提前返回。

**修复**：改成**分离式**——`podman run -d` 让容器在后台独立运行，systemd 单元用 `Type=oneshot` + `RemainAfterExit=yes`，只管"确保容器在跑"，不把容器绑在自己的主进程上。

### 坑 8：MCP 每隔一会儿就断（最终 Boss）

**症状**：服务稳定运行几十秒后突然不通，反复如此。前面所有修复都没能解决它。

**排查过程**（这一步花的时间最多）：

1. 先怀疑 supergateway 崩溃 → 加了守护循环，**没用**。
2. 再怀疑前台进程问题 → 改成分离式容器，**还是没用**。
3. 于是换个角度：**对比"发行版启动时间"和"容器启动时间"**。

```bash
echo "distro boot: $(uptime -s)"
echo "container started: $(podman inspect ... --format '{{.State.StartedAt}}')"
```

结果看到了决定性的证据：

```
distro boot:      <日期> <时刻>
container start:  <日期> 06:29:--   ← 紧跟其后
```

**发行版本身被重启了。** 不是容器崩了，是整个 WSL 被回收了。

4. 再验证：只从 Windows 侧发 HTTP 请求（不碰 WSL），100 秒内服务从第 3 次开始全部失败。而**只要我执行一条 `wsl` 命令，服务就又活了**。

**根因**：**WSL 在空闲时会自动关停整个发行版**，systemd 和容器跟着一起死。而**从 Windows 侧发往 WSL IP 的 HTTP 流量，不算作 WSL 的"活动"**——所以我的探活根本阻止不了回收。

**修复**：在 Windows 登录时起一个**常驻 WSL 会话**，把发行版钉住：

```powershell
Start-Process -FilePath "wsl.exe" `
  -ArgumentList @("-d", "Ubuntu", "-u", $wslUser, "--", "sleep", "999999999") `
  -WindowStyle Hidden | Out-Null
```

**效果**：修复后连续 12 次探活（2 分钟内）**全部 200 OK**，此前是 3/12。

> **这是整个项目最关键的一个发现。** 如果你也遇到"MCP 隔一会儿就断"，先查这一条。

### 坑 9：保活会话启动即退出

**症状**：常驻会话刚创建就没了。

**排查**：我原本这样写：

```powershell
-ArgumentList @("-d", $wslDistro, "-u", $wslUser, "-e", "bash", "-c", "exec -a my-marker sleep infinity")
```

**根因**：PowerShell 会把带空格的参数拆散，`bash -c` 的参数被切成了好几段。

**修复**：参数里**刻意不含空格**，用 `sleep 999999999` 代替 `sleep infinity`，并去掉 `-e bash -c`：

```powershell
-ArgumentList @("-d", $wslDistro, "-u", $wslUser, "--", "sleep", "999999999")
```

### 坑 10：`.bashrc` 自动启动导致 WSL 登录报错

**症状**：一进 WSL 就报 `-s): -c: line 2: unexpected EOF while looking for matching ')'`。

**排查**：我早期把自动启动脚本挂在了 `.bashrc` 里。

**根因**：写进去的片段有语法错误，导致**每次登录 WSL 都报错**。

**修复**：删掉 `.bashrc` 里那段，`bash -n` 校验通过。改用 **Windows 侧引导脚本**——这其实是更正确的架构：服务不该依赖"用户打开终端"才启动。

### 坑 11：中文搜索 45 秒超时

**症状**：搜索经常卡死或返回空结果，一次要等 45 秒。

**排查**：看日志发现请求发给了 DuckDuckGo / Google，国内根本连不上，一直等到超时。

**根因**：默认引擎选择走了海外站点。

**修复**：指定国内可直连的引擎链，并缩短超时：

```bash
-e SEARCH_REGION=cn-zh \
-e SEARCH_LANG_ROUTING=1 \
-e SEARCH_ENGINE=so360,bing,baidu,sogou \
-e SEARCH_TIMEOUT_MS=8000 \
-e SEARCH_CACHE_TTL=600
```

**效果**：搜索耗时从 45 秒降到 **3～5 秒**。

### 坑 12：DSH 显示 unknown tool

**症状**：工具调用报 `unknown tool`，**重启客户端也没用**。

**排查**：DSH 的 MCP 客户端配置里有一项 `failOnStartupError`，默认是 `true`。

**根因**：DSH 启动的瞬间如果 MCP 服务正好没就绪，它会**永久摘掉这个工具**，之后再也不会重试。

**修复**：

```json
"failOnStartupError": false,
"reconnect": {
  "enabled": true,
  "initialDelayMs": 500,
  "maxDelayMs": 30000,
  "maxAttempts": 10
}
```

改完后，服务没就绪也不会摘工具，且会自动重连。**这个配置对任何"服务比客户端启动慢"的场景都适用。**

### 坑 13：WSL 重启后客户端配置失效

**症状**：昨天好好的，今天连不上。

**根因**：WSL 的 IP（`<WSL_IP>`）是**临时的**，完整重启会变。Dify 用 `host.containers.internal` 不受影响，但 DSH 配的是写死的 IP。

**修复**：引导脚本每次登录都解析当前 IP，与预期值比对，不一致就写进日志告警：

```
WARNING: WSL IP changed - expected <WSL_IP> but got 172.x.x.x;
update DSH + Cherry Studio MCP url
```

我暂时保留了"手工更新配置"的方式——自动化改写客户端配置风险较高，**告警比乱改更稳妥**。

## 最终效果

以下都是**实测数据**，不是预期值。

### 1. 连续可用性

Windows 侧规范 HTTP 探活，2 分钟内 12 次：

```
[+0s]   HTTP=200    [+53s]  HTTP=200
[+11s]  HTTP=200    [+63s]  HTTP=200
[+21s]  HTTP=200    [+74s]  HTTP=200
[+32s]  HTTP=200    [+84s]  HTTP=200
[+42s]  HTTP=200    [+95s]  HTTP=200
                      [+105s] HTTP=200
                      [+116s] HTTP=200
结果: 12 / 12
```

修复前是 **3 / 12**（前 3 次成功，之后全部失败）。

### 2. 无人值守自愈（隔夜验证）

这是最有说服力的一次验证。第二天早上检查，看门狗日志显示：

```
distro boot: <日期> <时刻>   ← WSL 发行版当天早上被重启
[+12s] MCP unhealthy (http=000) -> systemctl restart mcp-search.service
[+20s] after restart http=200   ← 8 秒后恢复
```

**发生了什么**：当天早上 06:29，WSL 发行版被重启，systemd 和容器一起消失。**看门狗在 12 秒内发现异常，8 秒内恢复服务，全程无人干预。**

### 3. 服务状态

| 检查项 | 结果 |
|---|---|
| 容器状态 | `running`，`Up 54 minutes` |
| Windows 侧 HTTP 探活 | **200** |
| systemd 单元 | `active` / `enabled` |
| 看门狗定时器 | `active` / `enabled` |
| 端口 11000 | 监听中 |
| 常驻会话 | 运行中 |

### 4. 性能

| 指标 | 结果 |
|---|---|
| 搜索耗时 | **3～5 秒**（优化前 45 秒） |
| 镜像大小 | 318 MB |
| 搜索成本 | **0 元**（免费引擎） |
| 硬件要求 | CPU-only，16GB 内存可跑 |

## 可用工具

| 工具 | 功能 | 适用场景 |
|---|---|---|
| **search** | 单次搜索，返回结构化结果 | 简单查询，需要快速结果 |
| **fetch_page** | 抓取指定 URL 内容转 Markdown | 已知网页，需要提取正文 |
| **research** | 一次调用完成"搜索 + 抓取 Top 页 + 生成证据简报" | 复杂查询，需要综合分析 |
| **status** | 诊断引擎健康状态、缓存、运行配置 | 搜索失败时排查 |

## 日常运维手册

### 排查顺序（照着走就行）

```bash
# 1. 服务活着吗
sudo systemctl status mcp-search.service
sudo podman ps | grep dify-mcp-search

# 2. 端口在听吗
ss -lnt | grep 11000

# 3. 真的能应答吗（必须用 HTTP，不能用 TCP 探测！）
curl -s -m 10 -o /dev/null -w '%{http_code}\n' -X POST http://127.0.0.1:11000/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}'

# 4. 看门狗干了什么
tail -20 /home/<用户名>/mcp_watchdog.log

# 5. 发行版是不是被回收过（关键！）
uptime -s

# 6. 容器日志
sudo podman logs --tail 50 dify-mcp-search
```

### 常用维护命令

```bash
sudo systemctl restart mcp-search.service     # 重启服务
sudo systemctl status mcp-search-watchdog.timer  # 看门狗状态
journalctl -u mcp-search.service -f           # 跟踪日志
```

### 长期使用建议

- **常驻会话别关**：它是防 WSL 空闲回收的关键。若发现会话消失，重跑一次 `mcp_autostart.ps1` 即可。
- **WSL 重启后同步 IP**：查 `hostname -I`，更新 DSH 的 MCP URL。Dify 端不用改。
- **探活只用 HTTP**：任何"连 TCP 就断"的探测都会把服务探死。
- **看日志**：搜索异常先 `sudo podman logs --tail 50 dify-mcp-search`。
- **升级 supergateway 要谨慎**：现有方案是围绕 4.1.0 的行为设计的（守护循环 + stateless），换版本需重新验证。

## 安全提醒

- **11000 端口仅对 WSL2 网段开放**，Windows 防火墙规则限定 `172.29.0.0/16`。
- **不要将 11000 端口对公网开放**——这是一个无认证的搜索端点。
- **不要将 Dify 或 Ollama 的端口对公网开放**。
- Dify 用 `host.containers.internal` 访问，DSH 用 WSL2 实际 IP。
- 云端模型处理敏感信息时，注意不要上传敏感数据。

## 已知限制

诚实说明这套方案目前没解决的问题：

1. **WSL IP 变化需要手工同步**：引导脚本会告警，但不会自动改写客户端配置。
2. **supergateway 版本敏感**：方案依赖 4.1.0 的具体行为，升级需重新验证。
3. **无认证**：11000 端点没有鉴权，安全性完全依赖网络隔离。
4. **持久化依赖 WSL 运行**：Windows 关机/休眠期间服务不可用（这是必然的）。

## 写在最后

这个项目从"`nohup npx` 起一个进程"开始，最后变成"systemd 系统服务 + 分离容器 + 看门狗 + Windows 常驻会话"这样一套架构。中间走的弯路大多源于同一个思维误区：

> **总想让它"不要崩"，而不是"崩了也能自动起来"。**

真正的转折点是接受了三件事：

1. 组件会崩 —— 那就用守护循环兜住（坑 6）。
2. 宿主会回收资源 —— 那就主动钉住它（坑 8）。
3. 客户端会启动失败 —— 那就别让它永久放弃（坑 12）。

如果你正在被"MCP 隔一会儿就断"折磨，**请直接看坑 8**。我在这一个问题上花的时间，超过其他所有坑加起来。

希望这份记录能帮你少走一段路。

## 许可

本项目采用 Apache License 2.0 发布。

Copyright (c) 2026 Author

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
