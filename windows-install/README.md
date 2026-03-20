# OpenClaw Windows 安装指南

本文档提供了在 Windows 系统上安装 OpenClaw 的详细步骤，提供多种安装方式供您选择。

## 端口配置说明

为避免端口冲突，不同安装方式使用不同的端口：
- Windows 直接安装：使用默认端口 18789
- Docker 镜像安装：使用端口 18791
- WSL 安装：使用默认端口 18789（适用于Linux的Windows子系统,可以在windows 10或11系统直接运行Linux环境）

## 安装方式选择

OpenClaw 提供以下安装方式：

1. **方式一：使用 npm 直接安装（推荐）** - 最简单直接的方式
2. **方式二：使用本地安装脚本** - 适合需要自动安装依赖的场景
3. **方式三：Docker 镜像安装** - 适合容器化部署
4. **方式四：从源码安装** - 适合开发者

---

## 方式一：使用 npm 直接安装（推荐）

### 前置要求
- Windows 10 或 Windows 11
- Node.js 22+（[下载地址](https://nodejs.org) node -v 检查版本号是否为 22.x.x）
- npm（随 Node.js 一起安装）(npm --version 检查版本号)

### 安装步骤

1. **打开 PowerShell**（以管理员身份运行）

2. **安装 OpenClaw**：
   ```powershell
   npm install -g openclaw@latest
   # 若出现a git connection error occurred 或是 Permission denied(publickey)
   git config --global url."https://".insteadOf "ssh://"
   git config --global http.sslverify "false"
   ```

3. **验证安装**：
   ```powershell
   openclaw --version
   ```

4. **完成初始化设置**：
   ```powershell
   openclaw onboard --install-daemon
   ```

5. **启动网关服务**（如果需要手动启动）：
   ```powershell
   openclaw gateway --allow-unconfigured --port 18789
   ```

6. **验证服务状态**：
   ```powershell
   openclaw status
   ```

### 配置信息
- 配置目录：`%USERPROFILE%\.openclaw`
- 工作目录：`%USERPROFILE%\.openclaw\workspace`
- 默认端口：18789
- 桥接端口：18790

---

## 方式二：使用本地安装脚本

### 前置要求
- Windows 10 或 Windows 11
- PowerShell 5.1 或更高版本
- Node.js 22+（安装脚本会自动尝试安装）
- Git（安装脚本会自动尝试安装）

### 安装步骤

1. **打开 PowerShell**（以管理员身份运行）

2. **下载安装脚本**：
   ```powershell
   # 方法A：从 GitHub 下载
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/install.ps1" -OutFile "install.ps1"
   
   # 方法B：如果已克隆仓库，直接使用本地脚本
   # cd M:\Tianqing_Space\openclaw\openclaw\scripts
   ```

3. **执行安装脚本**：
   ```powershell
   # 使用默认参数安装（npm 方式）
   .\install.ps1
   
   # 或者指定版本
   .\install.ps1 -Tag "latest"
   
   # 或者使用 Git 方式安装
   .\install.ps1 -InstallMethod "git" -GitDir "$env:USERPROFILE\openclaw"
   ```

4. **完成初始化设置**：
   ```powershell
   openclaw onboard
   ```

5. **验证安装**：
   ```powershell
   openclaw --version
   ```

6. **启动网关服务**（在后台运行）：
   ```powershell
   Start-Process -FilePath 'powershell' -ArgumentList '-ExecutionPolicy Bypass -Command "openclaw gateway --allow-unconfigured --port 18789 --bind loopback"' -WindowStyle Minimized
   ```

7. **验证服务状态**：
   ```powershell
   openclaw status
   ```

### 配置信息
- 配置目录：`%USERPROFILE%\.openclaw`
- 工作目录：`%USERPROFILE%\.openclaw\workspace`
- 默认端口：18789
- 桥接端口：18790

### 安装脚本参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-InstallMethod` | 安装方式：npm 或 git | npm |
| `-Tag` | npm 安装版本标签 | latest |
| `-GitDir` | Git 安装目录 | `$env:USERPROFILE\openclaw` |
| `-NoOnboard` | 跳过初始化设置提示 | false |
| `-NoGitUpdate` | Git 安装时不更新仓库 | false |
| `-DryRun` | 仅显示将要执行的操作 | false |

### 常见问题

**Q: 执行策略限制错误**

如果遇到执行策略限制，运行以下命令：
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
```

**Q: Node.js 版本过低**

安装脚本会自动尝试安装 Node.js 22+，如果失败，请手动安装：
- 访问 [Node.js 官网](https://nodejs.org) 下载 LTS 版本
- 或使用 winget：`winget install OpenJS.NodeJS.LTS`

**Q: Git 未安装**

安装脚本会自动尝试安装 Git，如果失败，请手动安装：
- 访问 [Git 官网](https://git-scm.com) 下载
- 或使用 winget：`winget install Git.Git`

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
Invoke-WebRequest -UseBasicParsing http://localhost:18791/healthz
```

### 从源码安装验证
```powershell
# 检查版本
pnpm --version

# 运行测试
pnpm test
```

## 配置信息汇总

| 安装方式 | 配置目录 | 工作目录 | 网关端口 | 桥接端口 | 访问地址 |
|---------|---------|---------|---------|---------|----------|
| Windows 直接安装 | `%USERPROFILE%\.openclaw` | `%USERPROFILE%\.openclaw\workspace` | 18789/18793 | 18790 | http://localhost:18789 或 http://localhost:18793 |
| Docker 镜像安装 | `./config` | `./workspace` | 18791 | 18792 | http://localhost:18791 |
| WSL 安装 | `~/.openclaw` | `~/.openclaw/workspace` | 18789 | 18790 | http://localhost:18789 |
| 从源码安装 | `%USERPROFILE%\.openclaw` | `%USERPROFILE%\.openclaw\workspace` | 18789 | 18790 | http://localhost:18789 |

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