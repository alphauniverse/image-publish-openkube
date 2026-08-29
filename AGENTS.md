# AGENTS.md — 面向 AI Agent 的协作指南

> 本文件为 AI Agent 协作而生，已纳入版本管理。阅读本文前请先扫一眼仓库根目录 `README.md` 获取用户可见的项目总览。历史决策日志见 `.workbuddy/memory/`（该目录被 gitignore 忽略）。

---

## 1. 项目定位

本仓库是一个 **Docker 镜像集中构建工厂**：

- 通过 GitHub Actions 把多个自定义/上游镜像构建并推送到 Docker Hub。
- 每个镜像只需要一个轻量的 `.github/workflows/<name>.yml` 调用共享工作流。
- 构建逻辑、多架构、tag 生成、缓存都封装在 `.github/workflows/docker-publish.yml` 中。

---

## 2. 共享工作流约定

### 2.1 可复用工作流：`docker-publish.yml`

- **调用方式**：`workflow_call`，由每个镜像的 `.yml` 通过 `uses: ./.github/workflows/docker-publish.yml` 调用。
- **关键输入**：

  | input | 含义 | 示例 |
  |-------|------|------|
  | `image_repo` | 镜像仓库名（不含用户名） | `mirror-hub` / `octop` |
  | `context` | 构建上下文路径 | `.` / `_external` |
  | `dockerfile` | Dockerfile 相对**仓库根**的路径（build-push-action 的 `file` 按 GITHUB_WORKSPACE 解析，不相对 context） | `mirror-hub/Dockerfile` / `_external/docker/Dockerfile` |
  | `base_image` | 传入的 `BASE_IMAGE` build arg（仅在 Dockerfile 声明 `ARG BASE_IMAGE` 时有效） | `dqzboy/mirror-hub:latest` |
  | `tag_prefix` | 用于从 tag 提取语义版本的前缀 | `mirror-hub/` |
  | `platforms` | 目标平台 | `linux/amd64,linux/arm64` |
  | `source_repo` | 外部源码仓库（可选） | `TencentCloud/Octop` |
  | `source_ref` | 外部源码 ref（可选） | `main` / `0.9.13` |
  | `source_path` | 外部源码 checkout 后的目录名（可选） | `_external` |

- **镜像名拼接**：共享工作流内部使用 `${{ secrets.DOCKERHUB_USERNAME }}/${{ inputs.image_repo }}`。
  - **注意**：reusable workflow 的 caller 中 `with` 不能引用 `secrets`，因此拼接必须放在共享工作流内部。
- **外部源码 checkout**：当 `source_repo` 非空时，会把源码克隆到 `source_path`（默认 `_external`），构建上下文指向该目录。

### 2.2 每个镜像的调用工作流

- 命名：`.github/workflows/<image>.yml`
- 职责：只负责触发条件、路径监听、调用共享工作流并传参。
- 权限：必须声明 `permissions: contents: read` 和 `packages: write`（实际写 Docker Hub，但 Actions 需要该权限 token）。
- Secrets 通过 `secrets: inherit` 共享 `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`。

---

## 3. 两种新增镜像模式

### 模式 A：本仓库源码

参考 `mirror-hub/`：

```text
<image>/
├── Dockerfile
├── config.yml   # 可选
└── README.md
```

对应 `.github/workflows/<image>.yml`：

```yaml
name: <image>
on:
  push:
    branches: [main]
    paths: ['<image>/**', '.github/workflows/<image>.yml', '.github/workflows/docker-publish.yml']
    tags: ['<image>/v*']
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build:
    uses: ./.github/workflows/docker-publish.yml
    secrets: inherit
    with:
      image_repo: <image>
      context: <image>/
      dockerfile: <image>/Dockerfile
      base_image: '<base-image>'  # 可选
      tag_prefix: '<image>/'
      platforms: linux/amd64,linux/arm64
```

### 模式 B：外部仓库源码

参考 `octop/`：

```text
octop/
└── README.md   # 无 Dockerfile，使用上游官方 Dockerfile
```

对应 `.github/workflows/<image>.yml`：

```yaml
name: <image>
on:
  push:
    branches: [main]
    paths: ['.github/workflows/<image>.yml', '.github/workflows/docker-publish.yml']
    tags: ['<image>/v*']
  workflow_dispatch:
    inputs:
      <image>_ref:
        description: 'Upstream ref (tag/branch/commit)'
        required: false
        default: ''
  schedule:
    # 每小时检测上游 tag 变动，15 分避开整点资源拥堵
    - cron: '15 * * * *'

permissions:
  contents: read
  packages: write

jobs:
  build:
    uses: ./.github/workflows/docker-publish.yml
    secrets: inherit
    with:
      image_repo: <image>
      source_repo: owner/repo
      source_ref: ${{ github.event_name == 'schedule' && needs.check-upstream.outputs.upstream_tag || github.event.inputs.<image>_ref || vars.<IMAGE>_REF || 'main' }}
      source_path: _external
      context: _external
      dockerfile: _external/docker/Dockerfile
      platforms: linux/amd64,linux/arm64
```

> **schedule 触发依赖 `check-upstream` 两阶段 job**（见 §6.5 与 `octop.yml`）：上面的模板省略了该 job，直接照抄会让定时任务每次都用 `main` 构建推送。新增外部源码镜像时以 `octop.yml` 为底复制改名。

---

## 4. 关键 Secrets / Variables

### Secrets（必须）

| 名称 | 用途 |
|------|------|
| `DOCKERHUB_USERNAME` | 镜像名前缀（`username/repo`）及 Docker Hub 登录账号 |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token（非密码） |

### Variables（可选）

| 名称 | 用途 |
|------|------|
| `OCTOP_REF` | 固定 octop 上游源码版本，未配置则默认 `main` |

---

## 5. Tag 规则

由 `docker/metadata-action` 生成，共享工作流统一配置：

- **默认分支 push**：`latest`, `<branch-name>`
- **语义化 tag push**（如 `<image>/v1.2.3`）：`1.2.3`, `1.2`（未配置 major-only 模式，不会生成 `1`）
- **任意 commit push**：`<short-sha>`
- **外部源码模式**（`source_repo` 非空且 `source_ref` 非 `main`）：额外生成 `<source_ref>` 去 `v` 前缀的版本 tag（如 `v0.9.19` → `0.9.19`），由 `docker-publish.yml` 的 "Prepare source ref tag" 步骤实现

> Caller 通过 `tag_prefix` 告诉 metadata-action 如何剥离前缀。

---

## 6. 常见修改与注意事项

### 6.1 修改镜像名

不要改 caller 的 `image_repo` 拼接方式。只需在 GitHub Secrets 中修改 `DOCKERHUB_USERNAME`，所有镜像名会同步变化。

### 6.2 修改 actions 版本

- `actions/checkout` 当前为 `v7`（2026-07 已发布）。
- `docker/*` action 版本：setup-qemu `v4`、setup-buildx `v4`、login-action `v4`、metadata-action `v6`、build-push-action `v7`。
- 升级前建议到对应 action 的 releases 页面确认 tag 存在，避免 "unable to resolve action"。

### 6.3 多架构构建

- `mirror-hub` 默认双架构 `linux/amd64,linux/arm64`。
- `octop` 当前双架构，但因包含 Playwright chromium，arm64 兼容性需 CI 验证。
- 若某镜像 arm64 构建失败，可在该 caller 中显式指定 `platforms: linux/amd64`。

### 6.4 BASE_IMAGE build-arg 警告

`docker-publish.yml` 始终传入 `BASE_IMAGE=${{ inputs.base_image }}`。若 Dockerfile 未声明 `ARG BASE_IMAGE`，构建日志会出现：

```text
[Warning] One or more build-args were not consumed: [BASE_IMAGE]
```

这是已知低风险噪音，不影响构建成功。消除它需要条件输出 build-args，复杂度与收益不成正比，建议保持现状。

### 6.5 外部源码模式的定时触发

`schedule` 触发时采用两阶段检测：
1. **check-upstream job**：`git ls-remote --tags` 获取上游最新 tag，查询 Docker Hub API 确认该 tag 是否已存在
2. **call-docker-publish job**：仅在检测到新 tag 时调用共享工作流，`source_ref` 设为上游最新 tag

API 异常时保守触发构建（不漏发）。上游无 tag 时跳过。

---

## 7. 本地验证

### 7.1 本仓库镜像（模式 A）

```bash
cd <image>/
docker build -t <image>:test -f Dockerfile .
```

### 7.2 外部源码镜像（模式 B）

```bash
git clone --depth 1 https://github.com/owner/repo.git _external
cd _external
docker build -t <image>:test -f docker/Dockerfile .
```

---

## 8. 调试 CI

1. 到 GitHub 仓库 `Actions` 页查看对应 workflow 运行。
2. 常见失败点：
   - `actions/checkout@v7` 无法解析 → 回退到 v4。
   - Docker Hub 登录失败 → 检查 `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secret。
   - arm64 构建失败 → 在 caller 中改为 `linux/amd64`。
   - octop 外部源码拉取失败 → 检查 `source_repo` / `source_ref` 和网络。

---

## 9. Agent 操作 checklist

在协助修改本仓库前，请确认：

- [ ] 是否涉及新增镜像？若是，选择模式 A 或模式 B，并同步更新 `README.md` 目录结构。
- [ ] 是否修改了 `docker-publish.yml`？若是，检查所有 caller 的 input 名称是否仍然匹配。
- [ ] 是否修改了 action 版本？若是，先验证 tag 存在。
- [ ] 是否在 caller 的 `with` 中尝试引用 `secrets`？这是 GitHub Actions 禁止的，应把拼接放到共享工作流内部。
- [ ] 修改后是否执行了 `git status`、`git diff --stat`、提交并推送？
