# Linux ARM64 / 树莓派

这个 fork 保留上游主程序，只增加 ARM64 浏览器支持。Go 主程序和浏览器必须与镜像的目标架构一致。Dockerfile 用 BuildKit 的 `TARGETARCH` 编译 Go 程序，并按同一架构选择浏览器压缩包。

ARM64 使用 CloakBrowser 免费版 Chromium 146。版本号和 SHA256 分别固定在 [`browser/browser_linux_arm64_version.txt`](../browser/browser_linux_arm64_version.txt) 和 [`browser/browser_linux_arm64.sha256`](../browser/browser_linux_arm64.sha256)，Go 运行时下载器与 Dockerfile 共用这两个文件。构建时从 CloakHQ 官方 Release 下载浏览器；仓库不包含浏览器二进制。

## 构建与部署

在仓库根目录使用 Docker Buildx 构建 ARM64 镜像：

```bash
docker buildx build --platform linux/arm64 -t xiaohongshu-mcp:arm64 --load .
```

如果在 x86 主机交叉构建，Buildx 的 `RUN` 步骤需要 ARM64 模拟环境；直接在树莓派构建则不需要。如果下载依赖或浏览器需要代理，可通过 `--build-arg HTTP_PROXY=...` 和 `--build-arg HTTPS_PROXY=...` 传入。

Compose 覆盖文件会选用本地 ARM64 镜像。在树莓派构建镜像或把镜像导入树莓派之后运行：

```bash
cd docker
docker compose -f docker-compose.yml -f docker-compose.arm64.yml up -d --no-build
docker compose -f docker-compose.yml -f docker-compose.arm64.yml logs --tail=100
curl -fsS http://127.0.0.1:18060/health
```

也可以在树莓派上把 `up -d --no-build` 换成 `up -d --build`，直接从源码构建。每次启动都要同时指定两个 Compose 文件；单独使用上游的基础文件会选用上游镜像。现有的 `docker/data` 挂载会在更新镜像后保留 Cookie。

`AUTH_TOKEN` 为空时，MCP 和 API 路由不校验访问令牌。基础 Compose 文件默认把 18060 端口发布到所有网卡。可设置 `AUTH_TOKEN`；如果只有树莓派本机的 Tunnel 客户端需要访问，也可把端口映射改为 `127.0.0.1:18060:18060`。

## 后续维护

让 fork 的 `main` 分支持续跟进上游，定期合并上游更新：

```bash
git fetch upstream
git merge upstream/main
git push origin main
```

升级 ARM64 浏览器时，先确认目标 CloakHQ 版本仍可免费使用，再修改版本文件；从官方 Release 下载对应的 `cloakbrowser-linux-arm64.tar.gz`，计算 SHA256 并更新校验文件。随后运行 `go test ./...` 和 ARM64 Docker 构建。上游现有测试工作流已包含 Linux ARM64 的 Go 交叉编译检查；Docker 镜像构建在本地执行。

[CloakBrowser 二进制许可](https://github.com/CloakHQ/CloakBrowser/blob/main/BINARY-LICENSE.md)允许内部 Docker 使用和在构建脚本中引用依赖，但限制再分发浏览器二进制。未另获 CloakHQ 授权前，不要公开发布预装该浏览器的镜像。上游发布工作流仍按原样保留，不负责本 fork 的 ARM64 镜像发布。
