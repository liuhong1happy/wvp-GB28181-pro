<!-- Docker 部署 -->

# Docker 部署

`docker/` 目录下有一套完整的 compose 编排，一次性把 WVP 所需的五个服务都拉起来：
WVP-PRO（后端）、polaris-nginx（前端）、ZLMediaKit（流媒体）、MySQL、Redis。

镜像有两个来源，按场景选一种：

| 方式 | 适用场景 | 镜像从哪来 |
|---|---|---|
| [一、本地构建并部署](#一本地构建并部署) | 改了代码要立刻生效，或者网络拉不到 GHCR | 本机 `docker build` |
| [二、使用 GHCR 镜像部署](#二使用-ghcr-镜像部署) | 只想把服务跑起来，不想等十几分钟的编译 | GitHub Packages |

两种方式的前置准备完全一样，先做 [0 前置准备](#0-前置准备)。

## 0 前置准备

### 0.1 环境要求

| 依赖 | 版本 | 说明 |
|--------|-------|------|
| docker | 20.10+ | |
| docker compose | v2.24+ | 方式二用到了 compose 的 `!reset` 指令，版本过旧会报 unknown tag |

### 0.2 取代码

```shell
git clone git@gitee.com:liuhong1happy/wvp-GB28181-pro.git
cd wvp-GB28181-pro/docker
```

### 0.3 准备两个不在仓库里的文件

下面两个文件含真实凭据，被 `.git/info/exclude` 有意排除在版本控制之外，
clone 下来是没有的，**必须先补齐**，否则起不来。

**一、`docker/.env.local`** —— 环境变量覆盖。从仓库里那份 `.env` 复制后按实际情况改：

```shell
cp .env .env.local
```

| 键 | 作用 |
|--------|------|
| `Stream_IP` / `SDP_IP` | 流地址与 SDP 里用的 IP，填宿主机对设备可达的那个 IP |
| `SIP_ShowIP` | 设备注册时看到的平台 IP |
| `SIP_Id` / `SIP_Password` | 平台国标编码与信令密码，**生产环境务必改掉** |
| `SIP_Port` | SIP 信令端口，默认 8160；公网部署建议换成非标准端口 |
| `MediaRtmp` / `MediaRtsp` / `MediaRtp` | 媒体收流端口 |
| `WebHttp` | 前端访问端口，默认 8080 |

> 注意：下面的命令都带 `--env-file .env.local`。compose 指定了 `--env-file` 之后就**不再**
> 自动读取同目录的 `.env`，所以 `.env.local` 要是一份完整配置，而不是只写和 `.env` 的差异项。

**二、`docker/media/config.local.ini`** —— ZLM 的配置。从仓库里那份复制：

```shell
cp media/config.ini media/config.local.ini
```

仓库里那份是上游默认值，本地这份主要改两处：

- `[hls]` 段的 `enable_hls` 打开为 1 —— 上游默认是 0，不开的话 HLS 播放返回 404
- `[rtc]` 段的 `externIP` 填宿主机 IP —— 不填的话 ZLM 会把容器自己的内网 IP（如 172.18.0.x）
  当作 ICE 候选地址发给浏览器，那个地址在局域网上路由不到，WebRTC 必然连不上

> 为什么这个文件不能缺：`docker-compose.local.yml` 把 `./media/config.local.ini` 挂载到
> 容器内的 `/conf/config.ini`。若宿主机上这个文件不存在，Docker 会**建一个同名目录**再挂进去，
> ZLM 读不到配置文件直接启动失败。

## 一、本地构建并部署

```shell
docker compose --env-file .env.local \
  -f docker-compose.yml -f docker-compose.local.yml up -d --build
```

`docker-compose.local.yml` 这层 overlay 做了四件事：

1. 把 `polaris-wvp` / `polaris-nginx` 的 `build.dockerfile` 指向 `wvp/Dockerfile.local`
   和 `nginx/Dockerfile.local`
2. 给 `polaris-media` 发布 80（HTTP 播流）与 8000（WebRTC）端口
3. 给 `polaris-media` 换用上一步准备的 `config.local.ini`
4. 把 `polaris-wvp` 的 `depends_on` 从短语法改成长语法，让它真正等到 redis/mysql
   healthy 再启动，避免抢跑

镜像在容器内完成编译，**本机不需要装 JDK 和 Node**，代价是一次完整的 Maven + npm 构建，
十几分钟属于正常，看日志确认进度：

```shell
docker compose --env-file .env.local \
  -f docker-compose.yml -f docker-compose.local.yml logs -f polaris-wvp polaris-nginx
```

## 二、使用 GHCR 镜像部署

`.github/workflows/docker-publish.yml` 会在 push 到 main、推送 `v*.*.*` 标签、以及手动触发时，
构建两个镜像推送到 GitHub Packages：

| 镜像 | 由哪个 Dockerfile 构建 |
|--------|------|
| `ghcr.io/liuhong1happy/wvp_gb28181_pro_server` | `docker/wvp/Dockerfile.local` |
| `ghcr.io/liuhong1happy/wvp_gb28181_pro_web` | `docker/nginx/Dockerfile.local` |

可用的标签：

| 标签 | 含义 |
|--------|------|
| `latest` | 默认分支上最新一次成功构建 |
| `main` / `master` | 分支名 |
| `v2.7.4` | 发布标签，对应推的 `v*.*.*` |
| `sha-64dc829` | 对应 commit 的短哈希，适合固定版本或回滚 |

部署时在方式一的基础上再叠一层 `docker-compose.ghcr.yml`：

```shell
docker compose --env-file .env.local \
  -f docker-compose.yml -f docker-compose.local.yml -f docker-compose.ghcr.yml up -d
```

这层 overlay 用 `build: !reset null` 把两个服务的 `build` 摘掉、改成拉取上面的镜像，
所以省掉的正是那十几分钟编译。

> `!reset` 不能省。compose 合并多个 `-f` 文件时，`docker-compose.yml` 里的 `build` 会被继承，
> 而 `image` 和 `build` 同时存在时 `up` 走的是构建分支（构建完再打成这个名字），等于白拉。

这两个包是公开的，**不需要 `docker login ghcr.io`**。

想固定在某个版本而不跟 `latest` 走，把 `docker-compose.ghcr.yml` 里的 `:latest` 换成具体标签即可。

## 三、访问

浏览器打开 `http://<宿主机IP>:<WebHttp>`（默认 8080），默认登录账号和密码均为 `admin`。

## 四、需要开放的端口

| 服务 | 端口 | 类型 | 必选 |
|--------|--------|--------|--------|
| polaris-nginx | `${WebHttp}`，默认 8080 | tcp | 是 |
| polaris-wvp | 18978 | tcp | 是 |
| polaris-wvp | `${SIP_Port}`，默认 8160 | udp 和 tcp | 是 |
| polaris-media | `${MediaRtmp}`，默认 10001 | udp 和 tcp | 是 |
| polaris-media | `${MediaRtsp}`，默认 10002 | udp 和 tcp | 否 |
| polaris-media | `${MediaRtp}`，默认 10003 | udp 和 tcp | 是 |
| polaris-media | 80 | tcp | 是，且仅由 `docker-compose.local.yml` 发布 |
| polaris-media | 8000 | udp 和 tcp | 否，WebRTC 用，仅由 `docker-compose.local.yml` 发布 |

MySQL 和 Redis 只在 compose 的内部网络 `media-net` 里暴露，未发布到宿主机。

> 80 端口这条容易被忽略：WVP 生成的播放地址（FLV / WS-FLV / FMP4 / HLS）用的都是媒体节点的
> `http.port`，也就是 ZLM 的 80。这个端口不发布出去，播放地址指向宿主机 80 而无人监听，
> 浏览器上表现为「连接被拒绝」。

## 五、常用运维命令

以下命令都在 `docker/` 目录下执行，`-f` 的叠加方式与部署时保持一致。为简洁，
下面用变量代替：

```shell
export WVP_COMPOSE="docker compose --env-file .env.local -f docker-compose.yml -f docker-compose.local.yml"
```

**查看服务状态**

```shell
$WVP_COMPOSE ps
```

**查看日志**

```shell
$WVP_COMPOSE logs -f polaris-wvp
```

**停止服务**（保留数据）

```shell
$WVP_COMPOSE stop
```

**停止并删除容器**（数据在 `docker/volumes/` 下，不会丢）

```shell
$WVP_COMPOSE down
```

**更新到最新代码后重新部署**（方式一）

```shell
git pull
$WVP_COMPOSE up -d --build
```

**更新到最新镜像后重新部署**（方式二）

```shell
$WVP_COMPOSE -f docker-compose.ghcr.yml pull
$WVP_COMPOSE -f docker-compose.ghcr.yml up -d
```

**回滚到指定版本**（方式二）

把 `docker-compose.ghcr.yml` 里的 `:latest` 改成目标标签（如 `:sha-64dc829`），再执行一次上面的更新命令。

> MySQL 的初始化脚本来自 `数据库/2.7.4/初始化-mysql-2.7.4.sql`，只在**数据卷为空**时执行一次。
> 升级时若涉及表结构变更，需要手工执行对应的 `更新-*.sql`，或清空 `docker/volumes/mysql/data`
> 重新初始化（会丢数据）。
