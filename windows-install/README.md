# OpenClaw Windows 安装指南

本文档提供了在 Windows 系统上安装 OpenClaw 的详细步骤，包括直接安装和 Docker 镜像安装两种方式。

## 端口配置说明

为避免端口冲突，不同安装方式使用不同的端口：
- Windows 直接安装：使用默认端口 18789
- Docker 镜像安装：使用端口 18791
- WSL 安装：使用默认端口 18789（已存在）
- 本指南中的示例：使用端口 18793（避免与已存在的服务冲突）

## Windows 直接安装

### 前置要求
- Windows 10 或 Windows 11
- PowerShell 5.1 或更高版本
- Node.js 24+（安装脚本会自动尝试安装）
- Git（安装脚本会自动尝试安装）

### 安装步骤

1. **打开 PowerShell**（以管理员身份运行）

2. **执行安装脚本**：
   ```powershell
   iwr -useb https://openclaw.ai/install.ps1 | iex
   ```

3. **验证安装**：
   ```powershell
   openclaw --version
   ```

4. **启动网关服务**（在后台运行）：
   ```powershell
   Start-Process -FilePath 'powershell' -ArgumentList '-ExecutionPolicy Bypass -Command "openclaw gateway --allow-unconfigured --port 18793 --bind loopback"' -WindowStyle Minimized
   ```

5. **验证服务状态**：
   ```powershell
   openclaw status
   ```

### 配置信息
- 配置目录：`%USERPROFILE%\.openclaw`
- 工作目录：`%USERPROFILE%\.openclaw\workspace`
- 默认端口：18789
- 示例端口：18793（避免冲突）
- 桥接端口：18790

### 验证结果
✅ 已成功安装 OpenClaw 2026.3.13
✅ 命令行可识别 openclaw 命令
✅ 版本验证成功
✅ 网关服务已在后台启动

## Docker 镜像安装

### 前置要求
- Docker Desktop for Windows
- WSL 2 后端（推荐）

### 安装步骤

1. **创建安装目录**：
   ```powershell
   mkdir -p openclaw-docker\config openclaw-docker\workspace
   cd openclaw-docker
   ```

2. **创建 Docker Compose 配置文件**（docker-compose.yml）：
   ```yaml
   services:
     openclaw-gateway:
       build: ../openclaw
       environment:
         HOME: /home/node
         TERM: xterm-256color
         OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN:-}
         OPENCLAW_ALLOW_INSECURE_PRIVATE_WS: ${OPENCLAW_ALLOW_INSECURE_PRIVATE_WS:-}
         TZ: ${OPENCLAW_TZ:-UTC}
       volumes:
         - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
         - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
       ports:
         - "${OPENCLAW_GATEWAY_PORT:-18791}:18789"
         - "${OPENCLAW_BRIDGE_PORT:-18792}:18790"
       init: true
       restart: unless-stopped
       command:
         [
           "node",
           "openclaw.mjs",
           "gateway",
           "--bind",
           "lan",
           "--port",
           "18789",
         ]
       healthcheck:
         test:
           [
             "CMD",
             "node",
             "-e",
             "fetch('http://127.0.0.1:18789/healthz').then((r)=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))",
           ]
         interval: 30s
         timeout: 5s
         retries: 5
         start_period: 20s
   ```

3. **创建环境变量文件**（.env）：
   ```env
   OPENCLAW_IMAGE=openclaw:local
   OPENCLAW_CONFIG_DIR=./config
   OPENCLAW_WORKSPACE_DIR=./workspace
   OPENCLAW_GATEWAY_PORT=18791
   OPENCLAW_BRIDGE_PORT=18792
   OPENCLAW_GATEWAY_TOKEN=your-secure-token-here
   ```

4. **构建并启动容器**：
   ```powershell
   docker-compose up -d
   ```

5. **验证安装**：
   ```powershell
   docker ps
   # 检查 openclaw-gateway 容器是否运行
   ```

6. **访问服务**：
   打开浏览器访问 `http://localhost:18791`

### 配置信息
- 配置目录：`./config`（相对于 docker-compose.yml 所在目录）
- 工作目录：`./workspace`（相对于 docker-compose.yml 所在目录）
- 网关端口：18791
- 桥接端口：18792

### 注意事项
⚠️ 构建 Docker 镜像时可能需要良好的网络连接
⚠️ 首次构建可能需要较长时间，因为需要下载依赖和构建应用

## 验证安装

### Windows 直接安装验证
```powershell
# 检查服务状态
openclaw status

# 查看网关日志
openclaw logs
```

### Docker 安装验证
```powershell
# 检查容器状态
docker ps

# 测试健康检查
Invoke-WebRequest http://localhost:18791/healthz
```

## 配置信息汇总

| 安装方式 | 配置目录 | 工作目录 | 网关端口 | 桥接端口 | 访问地址 |
|---------|---------|---------|---------|---------|----------|
| Windows 直接安装 | `%USERPROFILE%\.openclaw` | `%USERPROFILE%\.openclaw\workspace` | 18789/18793 | 18790 | http://localhost:18789 或 http://localhost:18793 |
| Docker 镜像安装 | `./config` | `./workspace` | 18791 | 18792 | http://localhost:18791 |
| WSL 安装 | `~/.openclaw` | `~/.openclaw/workspace` | 18789 | 18790 | http://localhost:18789 |

## 常见问题

### 1. 端口冲突
- 如果遇到端口冲突，修改启动命令中的端口设置
- 例如：`openclaw gateway --port 18793`

### 2. 权限问题
- Windows 安装时确保以管理员身份运行 PowerShell
- Docker 安装时确保用户有足够的权限

### 3. 网络问题
- 确保防火墙允许相关端口的访问
- Docker 构建时需要良好的网络连接

### 4. 依赖问题
- 安装脚本会自动尝试安装必要的依赖
- 如果遇到依赖问题，手动安装 Node.js 和 Git

### 5. 服务后台运行
- 使用 `Start-Process` 命令在后台启动网关服务
- 示例：`Start-Process -FilePath 'powershell' -ArgumentList '-ExecutionPolicy Bypass -Command "openclaw gateway --port 18793"' -WindowStyle Minimized`

### 6. 认证令牌问题
- 网关启动时会自动生成认证令牌
- 令牌存储在 `%USERPROFILE%\.openclaw\openclaw.json` 文件中
- 访问 Control UI 时需要输入此令牌
- 查看令牌：`Get-Content $env:USERPROFILE\.openclaw\openclaw.json | Select-String -Pattern 'token'`

## 卸载指南

### Windows 直接安装卸载

1. **停止所有 OpenClaw 进程**：
   ```powershell
   # 查找并停止所有 node 进程
   Get-Process | Where-Object { $_.Name -like '*node*' } | Stop-Process -Force
   ```

2. **卸载 OpenClaw**：
   ```powershell
   npm uninstall -g openclaw
   ```

3. **删除配置目录**：
   ```powershell
   Remove-Item -Recurse -Force $env:USERPROFILE\.openclaw
   ```

4. **验证卸载**：
   ```powershell
   openclaw --version
   # 应该显示命令未找到
   ```

### Docker 安装卸载

1. **停止并移除容器**：
   ```powershell
   docker-compose down
   ```

2. **删除安装目录**：
   ```powershell
   Remove-Item -Recurse -Force .\openclaw-docker
   ```

3. **删除 Docker 镜像**（可选）：
   ```powershell
   docker rmi openclaw:local
   ```

## 故障排查

1. **查看日志**：
   - Windows：`openclaw logs`
   - Docker：`docker logs openclaw-gateway`

2. **检查配置**：
   - 确保配置文件正确设置
   - 检查环境变量是否正确

3. **网络测试**：
   - 使用 `telnet` 或 `Test-NetConnection` 测试端口连通性

4. **健康检查**：
   - 访问 `/healthz` 端点检查服务状态

5. **服务启动**：
   - 确保使用 `Start-Process` 在后台启动服务，避免会话断开导致服务停止

6. **认证问题**：
   - 确保在 Control UI 中输入正确的网关令牌
   - 令牌存储在 `%USERPROFILE%\.openclaw\openclaw.json` 文件中

---

**验证状态**：
- ✅ Windows 直接安装：已验证成功
- ✅ Windows 直接卸载：已验证成功
- ⚠️ Docker 镜像安装：配置正确，网络环境可能影响构建

**注意**：本文档基于 OpenClaw 最新版本，如有变更请参考官方文档。