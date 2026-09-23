可以在当前目录下：
使用`docker compose up -d`直接运行。
使用`docker compose up -d --build --force-recreate`强制重新构建所有服务的镜像并删除旧容器重新运行

`.env`用来配置环境变量，在这里配好之后，其它的配置会自动联动的。

`build.sh`用来以日期为tag构建镜像，推送到指定的容器注册表内（Windows下可以使用`Git Bash`运行）

## 镜像从哪来

`polaris-wvp`（后端）和 `polaris-nginx`（前端）两个镜像有两条获取路径，二选一。
**前置准备、端口清单、运维命令等完整说明见文档站
[《Docker 部署》](../doc/_content/introduction/deployment_docker.md)**，这里只列命令。

**一、本机构建** —— 用本地改造过的 Dockerfile 现搭：

```bash
docker compose --env-file .env.local \
  -f docker-compose.yml -f docker-compose.local.yml up -d --build
```

**二、拉取已发布的镜像** —— 在第一条命令上再叠一层 `docker-compose.ghcr.yml`：

```bash
docker compose --env-file .env.local \
  -f docker-compose.yml -f docker-compose.local.yml -f docker-compose.ghcr.yml up -d
```

这层 overlay 用 `build: !reset null` 摘掉两个服务的 `build`，改成从 GitHub Packages 拉取：

| 镜像 | 构建它的 Dockerfile |
| --- | --- |
| `ghcr.io/liuhong1happy/wvp_gb28181_pro_server` | `wvp/Dockerfile.local` |
| `ghcr.io/liuhong1happy/wvp_gb28181_pro_web` | `nginx/Dockerfile.local` |

镜像由 `.github/workflows/docker-publish.yml` 在 push 到 main、推 `v*.*.*` 标签时构建推送。
这两个包是公开的，不需要 `docker login ghcr.io`。

> 上面两条命令用到的 `.env.local` 和 `media/config.local.ini` **不在仓库里**
> （见 `.git/info/exclude`，两者含真实凭据），首次部署要先分别从 `.env` 和
> `media/config.ini` 复制一份。详见上面链接的文档。

其它的文件的作用暂不明确