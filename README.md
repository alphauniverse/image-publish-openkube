# Docker 镜像构建仓库

集中管理多个自定义 Docker 镜像的构建配置，每个镜像一个子目录，共享 GitHub Secrets 推送到 Docker Hub。

## 目录结构

```
.
├── .github/workflows/
│   ├── docker-publish.yml    # 可复用共享工作流（workflow_call）
│   └── mirror-hub.yml        # mirror-hub 镜像调用工作流
├── mirror-hub/               # 镜像 1：自定义 registry
│   ├── Dockerfile
│   ├── config.yml
│   └── README.md
└── <其他镜像>/               # 未来新增镜像，结构同上
    ├── Dockerfile
    └── README.md
```

## 共享 Secrets / Variables

在 GitHub 仓库 **Settings → Secrets and variables → Actions** 中配置一次，所有镜像工作流共享。

### Secrets（敏感信息，加密存储）

| 名称 | 说明 |
|------|------|
| `DOCKERHUB_USERNAME` | Docker Hub 用户名 |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token（在 Docker Hub → Account Settings → Security 创建，不要用密码） |

### Variables（非敏感，明文）

每个镜像对应一个 Variable，命名规则 `IMAGE_NAME_<大写目录名>`：

| 名称 | 说明 | 示例 |
|------|------|------|
| `IMAGE_NAME_MIRROR_HUB` | mirror-hub 目录的目标镜像全名 | `longshu/mirror-hub` |
| `IMAGE_NAME_<其他>` | 其他镜像的目标镜像全名 | `longshu/nginx` |

> 调用工作流中通过 `vars.IMAGE_NAME_XXX || '默认回退值'` 读取，未配置则使用回退值（需手动修改）。

## 新增镜像步骤

1. 在仓库根创建子目录（如 `my-nginx/`），放入 `Dockerfile` 和所需文件
2. 复制 `.github/workflows/mirror-hub.yml` 为 `.github/workflows/my-nginx.yml`，修改：
   - `name: my-nginx`
   - `paths` 监听改为 `my-nginx/**` 和 `.github/workflows/my-nginx.yml`
   - `tags` 改为 `my-nginx/v*`
   - `image_name` 改为 `${{ vars.IMAGE_NAME_MY_NGINX || 'your-dockerhub-username/my-nginx' }}`
   - `context` / `dockerfile` / `base_image` / `tag_prefix` 按需调整
3. 在 GitHub Variables 中添加 `IMAGE_NAME_MY_NGINX`
4. 推送即可触发构建

## 工作流设计

- **`docker-publish.yml`**：可复用工作流（`workflow_call`），封装构建、多架构、tag 生成、推送、缓存逻辑
- **`<镜像名>.yml`**：每个镜像一个轻量调用文件，只负责监听路径和传参，通过 `secrets: inherit` 共享 Secrets

## 通用 Tag 规则

| 触发场景 | 生成的 Tag |
|----------|-----------|
| push 到默认分支 | `latest` + 分支名 |
| 推送 tag `<镜像>/v1.2.3` | `1.2.3` + `1.2` |
| 任意 commit | `<short-sha>` |

## 本地验证

每个镜像目录的 README 中有对应的本地构建命令。通用模式：

```bash
docker build -t <image-name>:test -f <dir>/Dockerfile <dir>/
```
