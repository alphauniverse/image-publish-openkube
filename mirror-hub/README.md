# Mirror Hub Registry 镜像

基于 `dqzboy/mirror-hub:latest`，将本目录下的 `config.yml` 覆盖到容器内 `/etc/distribution/config.yml`，启动一个 **Docker Hub 代理 / 缓存 registry**。

## 目录结构

```
mirror-hub/
├── Dockerfile          # 镜像构建文件
├── config.yml          # registry 配置文件（打包进镜像）
└── README.md
```

## 镜像行为

- 基础镜像：`dqzboy/mirror-hub:latest`（可通过 build-arg `BASE_IMAGE` 覆盖）
- 将 `config.yml` 复制到 `/etc/distribution/config.yml`
- 暴露端口 `5000`，挂载点 `/var/lib/registry`
- 运行模式：Docker Hub 代理缓存（`proxy.remoteurl: https://registry-1.docker.io`），`auth` 段未启用

## 触发条件

由 `.github/workflows/mirror-hub.yml` 调用共享工作流 `docker-publish.yml` 触发：

- 推送到 `main` 分支且修改了 `mirror-hub/**` 或对应工作流文件
- 推送 `mirror-hub/v*` 格式的 tag（如 `mirror-hub/v1.0.0`）
- 手动触发（workflow_dispatch）

## 镜像 Tag 规则

| 触发场景 | 生成的 Tag |
|----------|-----------|
| push 到默认分支 | `latest` + `main` |
| 推送 tag `mirror-hub/v1.2.3` | `1.2.3` + `1.2` |
| 任意 commit | `<short-sha>` |

## 本地验证

```bash
# 在仓库根目录执行
docker build -t mirror-hub-test -f mirror-hub/Dockerfile mirror-hub/

# 验证 config 已就位
docker run --rm mirror-hub-test cat /etc/distribution/config.yml
# 应输出 config.yml 内容，包含 proxy.remoteurl: https://registry-1.docker.io

# 启动代理 registry（需挂载存储卷）
docker run -d -p 5000:5000 -v registry-data:/var/lib/registry mirror-hub-test
# 拉取测试：docker pull localhost:5000/library/alpine
```

## 配置说明（config.yml 关键项）

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `http.addr` | `:5000` | 监听端口 |
| `http.relativeurls` | `true` | 返回相对 location，便于反代 |
| `storage.filesystem.rootdirectory` | `/var/lib/registry` | 存储根目录 |
| `storage.delete.enabled` | `true` | 允许删除镜像 |
| `storage.maintenance.uploadpurging` | `age: 168h, interval: 24h` | 上传分块清理：保留 7 天，每 24h 执行 |
| `proxy.remoteurl` | `https://registry-1.docker.io` | 上游 Docker Hub 地址 |
| `auth` | （注释） | 未启用认证 |

## 注意事项

- 当前为 **无认证** 模式，部署到公网需自行在 `config.yml` 中启用 `auth.htpasswd` 段并补充 htpasswd 文件
- 目标镜像名由 `DOCKERHUB_USERNAME` secret 拼接为 `<USERNAME>/mirror-hub`（详见仓库根目录 README）
