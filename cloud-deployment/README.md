# OpenClaw 云端部署指南

本文档提供了在云端部署 OpenClaw 的详细步骤，主要基于 Fly.io 平台的部署方式。

## 前置要求
- [Fly.io 账号](https://fly.io)（免费套餐可用）
- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) 工具
- 模型提供商的 API 密钥（如 Anthropic、OpenAI 等）
- 渠道凭证（如 Discord 机器人令牌、Telegram 令牌等）

## 部署步骤

### 1. 安装 Fly.io CLI

**Windows (PowerShell)：**
```powershell
iwr https://fly.io/install.ps1 -useb | iex
```

**macOS / Linux：**
```bash
curl -L https://fly.io/install.sh | sh
```

### 2. 登录 Fly.io

```bash
fly auth login
```

### 3. 克隆 OpenClaw 仓库

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

### 4. 创建 Fly 应用

```bash
# 创建新的 Fly 应用（选择自己的应用名称）
fly apps create my-openclaw

# 创建持久卷（1GB 通常足够）
fly volumes create openclaw_data --size 1 --region iad
```

**提示：** 选择靠近您的区域。常见选项：`lhr`（伦敦）、`iad`（弗吉尼亚）、`sjc`（圣何塞）。

### 5. 配置 fly.toml

编辑 `fly.toml` 文件，匹配您的应用名称和需求：

```toml
app = "my-openclaw"  # 您的应用名称
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  NODE_ENV = "production"
  OPENCLAW_PREFER_PNPM = "1"
  OPENCLAW_STATE_DIR = "/data"
  NODE_OPTIONS = "--max-old-space-size=1536"

[processes]
  app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 1
  processes = ["app"]

[[vm]]
  size = "shared-cpu-2x"
  memory = "2048mb"

[mounts]
  source = "openclaw_data"
  destination = "/data"
```

**关键设置：**

| 设置 | 原因 |
| --- | --- |
| `--bind lan` | 绑定到 `0.0.0.0`，以便 Fly 的代理能够访问网关 |
| `--allow-unconfigured` | 在没有配置文件的情况下启动（之后会创建） |
| `internal_port = 3000` | 必须与 `--port 3000` 匹配，以确保 Fly 健康检查正常 |
| `memory = "2048mb"` | 512MB 太小；推荐 2GB |
| `OPENCLAW_STATE_DIR = "/data"` | 在卷上持久化状态 |

### 6. 设置密钥

```bash
# 必需：网关令牌（用于非环回绑定）
fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

# 模型提供商 API 密钥
fly secrets set ANTHROPIC_API_KEY=sk-ant-...

# 可选：其他提供商
fly secrets set OPENAI_API_KEY=sk-...
fly secrets set GOOGLE_API_KEY=...

# 渠道令牌
fly secrets set DISCORD_BOT_TOKEN=MTQ...
```

**注意：**
- 非环回绑定（`--bind lan`）需要 `OPENCLAW_GATEWAY_TOKEN` 以确保安全。
- 将这些令牌视为密码。
- **首选环境变量而非配置文件**来存储所有 API 密钥和令牌。这样可以防止密钥在 `openclaw.json` 中意外暴露或被记录。

### 7. 部署

```bash
fly deploy
```

首次部署会构建 Docker 镜像（约 2-3 分钟）。后续部署会更快。

部署后，验证：

```bash
fly status
fly logs
```

您应该看到：

```
[gateway] listening on ws://0.0.0.0:3000 (PID xxx)
[discord] logged in to discord as xxx
```

### 8. 创建配置文件

SSH 进入机器创建适当的配置：

```bash
fly ssh console
```

创建配置目录和文件：

```bash
mkdir -p /data
cat > /data/openclaw.json << 'EOF'
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": ["anthropic/claude-sonnet-4-5", "openai/gpt-4o"]
      },
      "maxConcurrent": 4
    },
    "list": [
      {
        "id": "main",
        "default": true
      }
    ]
  },
  "auth": {
    "profiles": {
      "anthropic:default": { "mode": "token", "provider": "anthropic" },
      "openai:default": { "mode": "token", "provider": "openai" }
    }
  },
  "bindings": [
    {
      "agentId": "main",
      "match": { "channel": "discord" }
    }
  ],
  "channels": {
    "discord": {
      "enabled": true,
      "groupPolicy": "allowlist",
      "guilds": {
        "YOUR_GUILD_ID": {
          "channels": { "general": { "allow": true } },
          "requireMention": false
        }
      }
    }
  },
  "gateway": {
    "mode": "local",
    "bind": "auto"
  },
  "meta": {
    "lastTouchedVersion": "2026.1.29"
  }
}
EOF
```

**注意：** 使用 `OPENCLAW_STATE_DIR=/data` 时，配置路径为 `/data/openclaw.json`。

**注意：** Discord 令牌可以来自：
- 环境变量：`DISCORD_BOT_TOKEN`（推荐用于密钥）
- 配置文件：`channels.discord.token`

如果使用环境变量，无需在配置中添加令牌。网关会自动读取 `DISCORD_BOT_TOKEN`。

重启以应用：

```bash
exit
fly machine restart <machine-id>
```

### 9. 访问网关

#### 控制 UI

在浏览器中打开：

```bash
fly open
```

或访问 `https://my-openclaw.fly.dev/`

粘贴您的网关令牌（来自 `OPENCLAW_GATEWAY_TOKEN`）进行身份验证。

#### 日志

```bash
fly logs              # 实时日志
fly logs --no-tail    # 最近的日志
```

#### SSH 控制台

```bash
fly ssh console
```

## 故障排查

### "App is not listening on expected address"

网关绑定到 `127.0.0.1` 而不是 `0.0.0.0`。

**修复：** 在 `fly.toml` 的进程命令中添加 `--bind lan`。

### 健康检查失败 / 连接被拒绝

Fly 无法在配置的端口上访问网关。

**修复：** 确保 `internal_port` 与网关端口匹配（设置 `--port 3000` 或 `OPENCLAW_GATEWAY_PORT=3000`）。

### OOM / 内存问题

容器不断重启或被杀死。迹象：`SIGABRT`、`v8::internal::Runtime_AllocateInYoungGeneration` 或静默重启。

**修复：** 在 `fly.toml` 中增加内存：

```toml
[[vm]]
  memory = "2048mb"
```

或更新现有机器：

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

**注意：** 512MB 太小。1GB 可能工作但在负载或详细日志下可能 OOM。**推荐 2GB**。

### 网关锁定问题

网关拒绝启动，显示 "already running" 错误。

当容器重启但 PID 锁定文件在卷上持久存在时会发生这种情况。

**修复：** 删除锁定文件：

```bash
fly ssh console --command "rm -f /data/gateway.*.lock"
fly machine restart <machine-id>
```

锁定文件位于 `/data/gateway.*.lock`（不在子目录中）。

### 配置未被读取

如果使用 `--allow-unconfigured`，网关会创建最小配置。重启时应该读取您在 `/data/openclaw.json` 的自定义配置。

验证配置是否存在：

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### 通过 SSH 写入配置

`fly ssh console -C` 命令不支持 shell 重定向。要写入配置文件：

```bash
# 使用 echo + tee（从本地管道到远程）
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# 或使用 sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

**注意：** 如果文件已存在，`fly sftp` 可能失败。先删除：

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### 状态未持久化

如果重启后丢失凭证或会话，状态目录正在写入容器文件系统。

**修复：** 确保在 `fly.toml` 中设置了 `OPENCLAW_STATE_DIR=/data` 并重新部署。

## 更新

```bash
# 拉取最新更改
git pull

# 重新部署
fly deploy

# 检查健康
fly status
fly logs
```

### 更新机器命令

如果需要在不完整重新部署的情况下更改启动命令：

```bash
# 获取机器 ID
fly machines list

# 更新命令
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# 或增加内存
fly machine update <machine-id> --vm-memory 2048 --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

**注意：** `fly deploy` 后，机器命令可能会重置为 `fly.toml` 中的内容。如果您进行了手动更改，请在部署后重新应用。

## 私有部署（加固）

默认情况下，Fly 分配公共 IP，使您的网关可在 `https://your-app.fly.dev` 访问。这很方便，但意味着您的部署可被互联网扫描器（Shodan、Censys 等）发现。

对于 **无公共暴露** 的加固部署，使用私有模板。

### 何时使用私有部署

- 您只进行 **出站** 调用/消息（无入站 webhook）
- 您使用 **ngrok 或 Tailscale** 隧道进行任何 webhook 回调
- 您通过 **SSH、代理或 WireGuard** 而非浏览器访问网关
- 您希望部署 **对互联网扫描器隐藏**

### 设置

使用 `fly.private.toml` 代替标准配置：

```bash
# 使用私有配置部署
fly deploy -c fly.private.toml
```

或转换现有部署：

```bash
# 列出当前 IP
fly ips list -a my-openclaw

# 释放公共 IP
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# 切换到私有配置，以便将来部署不会重新分配公共 IP
#（删除 [http_service] 或使用私有模板部署）
fly deploy -c fly.private.toml

# 分配仅私有 IPv6
fly ips allocate-v6 --private -a my-openclaw
```

之后，`fly ips list` 应该只显示 `private` 类型的 IP：

```
VERSION  IP                   TYPE             REGION
v6       fdaa:x:x:x:x::x      private          global
```

### 访问私有部署

由于没有公共 URL，请使用以下方法之一：

**选项 1：本地代理（最简单）**

```bash
# 将本地端口 3000 转发到应用
fly proxy 3000:3000 -a my-openclaw

# 然后在浏览器中打开 http://localhost:3000
```

**选项 2：WireGuard VPN**

```bash
# 创建 WireGuard 配置（一次性）
fly wireguard create

# 导入到 WireGuard 客户端，然后通过内部 IPv6 访问
# 示例：http://[fdaa:x:x:x:x::x]:3000
```

**选项 3：仅 SSH**

```bash
fly ssh console -a my-openclaw
```

### 私有部署的 Webhook

如果您需要 webhook 回调（Twilio、Telnyx 等）而无需公共暴露：

1. **ngrok 隧道** - 在容器内或作为 sidecar 运行 ngrok
2. **Tailscale Funnel** - 通过 Tailscale 暴露特定路径
3. **仅出站** - 一些提供商（Twilio）在没有 webhook 的情况下也能正常进行出站调用

## 配置信息

| 配置项 | 值 |
| --- | --- |
| 应用 URL | 由 Fly.io 分配（例如：`https://your-app-name.fly.dev`） |
| 端口 | 443（HTTPS） |
| 内存 | 2048MB |
| 存储 | 1GB 持久卷 |
| 配置目录 | `/data` |
| 工作目录 | `/data/workspace` |

## 成本

使用推荐配置（`shared-cpu-2x`，2GB RAM）：
- 每月约 $10-15，具体取决于使用情况
- 免费套餐包含一些配额

有关详细信息，请参阅 [Fly.io 定价](https://fly.io/docs/about/pricing/)。

## 注意事项

- Fly.io 使用 **x86 架构**（不是 ARM）
- Dockerfile 兼容两种架构
- 对于 WhatsApp/Telegram 入职，使用 `fly ssh console`
- 持久数据存储在 `/data` 卷上
- Signal 需要 Java + signal-cli；使用自定义镜像并保持内存在 2GB+。

---

**注意**：本文档基于 OpenClaw 最新版本，如有变更请参考官方文档。