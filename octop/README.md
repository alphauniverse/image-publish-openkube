# Octop 镜像

基于 [TencentCloud/Octop](https://github.com/TencentCloud/Octop) 源码构建的 **自托管 AI 助手平台**镜像。Octop 支持多用户、多 Agent，集成记忆系统、技能市场、远程浏览器/桌面、多 Provider 接入等能力。

> 本镜像**从 Octop 上游源码多阶段构建**，非基于官方预构建镜像覆盖（Octop 目前未发布官方镜像）。构建使用上游仓库 `docker/Dockerfile`，构建上下文为上游仓库根目录。

## 目录结构

```
octop/
└── README.md          # 本说明（构建配置在 .github/workflows/octop.yml）
```

> 与 `mirror-hub/` 不同，本目录**不含 Dockerfile**——构建直接使用上游 Octop 仓库的 `docker/Dockerfile`，源码在构建时由工作流从 `TencentCloud/Octop` checkout 到 `_external/` 目录。

## 镜像行为

- **构建方式**：多阶段构建（node:20-slim 前端 + python:3.12-slim 运行时 + Playwright chromium）
- **暴露端口**：`8088`（HTTP 服务，Web Dashboard + API）
- **数据卷**：`/data/.octop`（SQLite 数据库、secrets、agents 工作区、日志、venv）
- **默认账号**：`admin` / `octop`（首次启动初始化，需立即修改）
- **健康检查**：`GET /api/health`，启动宽限期 90s

## 触发条件

由 `.github/workflows/octop.yml` 调用共享工作流 `docker-publish.yml` 触发：

- 推送到 `main` / `master` 分支且修改了 `octop.yml` 或 `docker-publish.yml`
- 推送 `octop/v*` 格式的 tag（如 `octop/v1.0.0`）
- 手动触发（workflow_dispatch），可在触发时指定 `octop_ref` 选择上游版本
- 定时触发（schedule）：每 6 小时（00:30、06:30、12:30、18:30 UTC）自动检测上游最新 tag，若 Docker Hub 尚无该 tag 则构建推送；已存在则跳过

## 上游版本控制

Octop 源码 ref 按以下优先级解析：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1 | workflow_dispatch 的 `octop_ref` 输入 | 手动触发时指定 |
| 2 | GitHub Variables `OCTOP_REF` | 仓库级配置，便于无代码改动切换版本 |
| 3 | `main`（默认） | 跟踪上游最新 |

> 在 GitHub **Settings → Secrets and variables → Actions → Variables** 中添加 `OCTOP_REF`（如 `0.9.13`）即可固定版本。

## 镜像 Tag 规则

| 触发场景 | source_ref | 生成的镜像 Tag |
|----------|-----------|---------------|
| push 到默认分支 | `main` | `latest` + `main` + `<short-sha>` |
| 推送 tag `octop/v1.2.3` | `main` | `latest` + `main` + `1.2.3` + `1.2` + `<short-sha>` |
| 手动触发，输入 `v0.9.19` | `v0.9.19` | `latest` + `main` + `v0.9.19` + `<short-sha>` |
| 手动触发，留空 | `vars.OCTOP_REF` 或 `main` | `latest` + `main` + (版本号) + `<short-sha>` |
| 定时检测到新 tag `v0.9.20` | `v0.9.20` | `latest` + `main` + `v0.9.20` + `<short-sha>` |
| 定时检测，无新 tag | - | 跳过，不构建 |

## 本地验证

由于源码在外部仓库，本地验证需先 clone Octop：

```bash
# 1. 克隆 Octop 源码
git clone https://github.com/TencentCloud/Octop.git /tmp/octop
cd /tmp/octop

# 2. 用官方 Dockerfile 构建（构建上下文为仓库根）
docker build -f docker/Dockerfile -t octop-test .

# 3. 运行
docker run -d -p 8088:8088 \
  -v octop-data:/data/.octop \
  -e HOME=/data \
  -e OCTOP_DEFAULT_PASSWORD=changeme \
  octop-test

# 4. 访问 http://localhost:8088 ，默认账号 admin / changeme
```

## 运行配置（关键环境变量）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OCTOP_PORT` | `8088` | HTTP 监听端口 |
| `OCTOP_ADMIN_USERNAME` | `admin` | 首次启动管理员用户名 |
| `OCTOP_DEFAULT_PASSWORD` | `octop` | 首次启动管理员密码 |
| `OCTOP_ADMIN_DISPLAY_NAME` | `Admin` | 管理员显示名 |
| `HOME` | `/data` | 容器 HOME 目录，使 `~/.octop` 映射到数据卷 |
| `OCTOP_LOG_LEVEL` | `info` | 日志级别 |

## 数据持久化

容器内数据目录 `/data/.octop`（即 `~/.octop/`）：

```
~/.octop/
├── octop.db              # SQLite — 用户、Agent、频道、定时任务
├── secrets/              # JWT secret、频道 token
├── agents/<agent_id>/    # 各 Agent 工作区（SOUL.md、skills 等）
├── logs/                 # 运行日志
└── venv/                 # uv 管理的 Python 环境
```

部署时通过 `-v octop-data:/data/.octop` 持久化，容器重建后数据不丢失。

## 注意事项

- 构建 `linux/amd64` + `linux/arm64` 双架构：Octop 含 Playwright chromium，多架构构建耗时显著（首次约 20-30 分钟，GHA 缓存命中后约 8-12 分钟）
- arm64 上 Playwright chromium 由官方支持，但构建较 amd64 更慢；如遇 arm64 兼容问题可临时在 `octop.yml` 中将 `platforms` 改回 `linux/amd64`
- 上游 Dockerfile 支持国内加速 build-arg（`PIP_INDEX_URL` / `NPM_REGISTRY` / `APT_MIRROR`），当前工作流未传入（GitHub Actions runner 在海外，无需加速）；如需国内构建可扩展 `docker-publish.yml` 的 build-args
- 目标镜像名由 `DOCKERHUB_USERNAME` secret 拼接为 `<USERNAME>/octop`（详见仓库根目录 README）
- 定时检测依赖 Docker Hub API 查重：通过 `https://hub.docker.com/v2/repositories/<USERNAME>/octop/tags/<tag>/` 返回 HTTP 404 判定 tag 不存在。API 异常时保守触发构建，不会漏发
- 上游无 tag 时（如纯 branch 仓库），定时检测跳过构建，退化为手动触发
