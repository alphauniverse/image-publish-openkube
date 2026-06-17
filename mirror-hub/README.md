# Mirror Hub Registry 镜像

基于 `dqzboy/registry:latest`，将本目录下的 `.password` 复制到容器内 `/etc/auth/.password`，并设置 `600` 权限，确保 registry 能以认证文件方式启动。

## 目录结构

```
mirror-hub/
├── Dockerfile          # 镜像构建文件
├── .password           # htpasswd 格式的认证密码文件
├── config.yml          # registry 配置文件（参考用，未打包进镜像）
└── README.md
```

## 镜像行为

- 基础镜像：`dqzboy/registry:latest`（可通过 build-arg `BASE_IMAGE` 覆盖）
- 将 `.password` 复制到 `/etc/auth/.password`，权限 `600`
- 暴露端口 `5000`，挂载点 `/var/lib/registry`

## 触发条件

由 `.github/workflows/mirror-hub.yml` 调用共享工作流 `docker-publish.yml` 触发：

- 推送到 `main` / `master` 分支且修改了 `mirror-hub/**` 或对应工作流文件
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
docker build -t registry-test -f mirror-hub/Dockerfile mirror-hub/

# 运行验证密码文件已就位
docker run --rm registry-test ls -l /etc/auth/.password
# 应输出：-rw------- root root ... /etc/auth/.password
```

## 注意事项

- `mirror-hub/.password` 当前为空文件，推送前请填入 htpasswd 格式的实际内容
- 目标镜像名通过 GitHub Variables 中的 `IMAGE_NAME_MIRROR_HUB` 配置（详见仓库根目录 README）
