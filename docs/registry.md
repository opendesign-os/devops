# 镜像仓库参考（Zot，隧道内 HTTP）

Zot 是 CNCF 的 OCI 原生镜像仓库，单进程单容器，常驻内存 100~200 MB。
它在本方案里同时承担两件事：存自研镜像（`apps/`）、代理缓存公共镜像（`docker/`）。

**安装步骤不在本文。** 检出仓库是 [deploy.md](deploy.md) 第二节第 1 步；建目录、生成账号、启动、验证、登录在它的第五节，按顺序做下来即可。本文只讲配置项含义、授权规则、保留策略调参、回源上游与日常运维。

## 一、环境与设计取舍

| 项 | 要求 |
|---|---|
| 版本 | Zot v2.1.20，镜像 `ghcr.io/project-zot/zot-linux-amd64` |
| 机器 | 机器 A，与 Dokploy 同机 |
| 内存 | 常驻 100~200 MB；开启 CVE 扫描后另计 |
| 磁盘 | `/srv/registry/data` 预留 60 GB |
| 主机名 | `registry.internal` → 机器 A 隧道地址 `10.8.0.1` |
| 端口 | 5000，只绑定在隧道地址上 |
| 配置与启动目录 | 机器 A 的检出 `/srv/devops/registry`，容器就地读这里的 `config.json` |
| 运行时状态 | `/srv/registry/` 下的 `htpasswd` 与 `data/`，不进版本控制 |
| 回源上游 | 两条规则：`makeplane/**` 走 `docker.1panel.live`，其余走 `docker.m.daocloud.io`。**每条只配一个 URL**，理由见第七节 |
| 首次回源 | 同步阻塞且同步全部架构，单个镜像 2~5 分钟，客户端必然先报 `EOF`，见第七节 |

必须用 **full 版镜像**（`zot-linux-amd64`）。名字带 `minimal` 的那个不含任何扩展，代理缓存、UI、保留策略全都没有。

Zot 由人手工 `docker compose up -d` 启动，不交给 Dokploy 托管：Dokploy 部署任何服务都要先能拉镜像，镜像仓库反过来由它部署会形成循环依赖，机器重装时无法自举。

两台机器只能走公网，仓库因此跑在隧道内：流量由 WireGuard 加密，HTTP 明文只存在于隧道内部，`insecure-registries` 的豁免也仅限隧道地址。**不要跳过隧道直接把 5000 开到公网** —— `docker login` 凭据是 base64 明文，截获即可推镜像投毒。

本方案不给端口加防火墙规则，而是让容器**只绑定 `10.8.0.1:5000`**。端口从一开始就没监听在公网网卡上，比"先开放再用防火墙堵"更难出事。代价是 wg0 必须先于 Docker 起来，否则容器绑不到地址会反复重启。

确实无法建隧道时的替代路径（Traefik 反代 + 公网 HTTPS）见 [deploy.md](deploy.md) 第九节，与隧道方案互斥。

## 二、账号与授权

授权规则写在 `config.json` 的 `http.accessControl` 里。权威副本是本仓库的 [config.json](../registry/config.json)，改动按这条链走：改仓库 → push → 机器 A `sudo git -C /srv/devops pull` → 重启容器生效，命令见第六节。

| 账号 | `apps/**` | `docker/**` | 登录位置 |
|---|---|---|---|
| `ci` | read + create + update + delete | read | 机器 A，Dokploy 构建后推送，跑在 A 的服务也用它拉取 |
| `deploy` | read | read | 机器 B，拉取运行 |
| `admin` | 全部（走 `adminPolicy`） | 全部 | 仅登录 UI 用 |

`defaultPolicy` 与 `anonymousPolicy` 都是空数组，含义是**未匹配到策略的已认证用户、以及所有匿名请求，一律无权限**。新增账号必须同时在 htpasswd 和 accessControl 里各加一处，只加一处不会生效。

Zot 的策略里 `read` 是其他动作的前提，写 `["create"]` 而不带 `read` 属于无效配置。

## 三、Web UI

UI 由 `extensions.ui` 提供，和 API 同端口，不额外占端口。管理员经 SSH 隧道访问，命令见 [deploy.md](deploy.md) 第八节。

UI 的漏洞面板依赖 `extensions.search` 的 CVE 能力，开启后会定期下载并更新漏洞库，额外占磁盘与内存。机器 A 资源紧张时，把 `search` 与 `ui` 两段一起从 config 里删掉即可，镜像存取与代理缓存完全不受影响。

## 四、保留策略与 GC

Zot 把保留和 GC 做在一起，都配在 `storage` 段，由内置调度器按周期自动执行，**不需要分两处配、也不需要手工触发**。`gc` 必须为 `true`，保留策略才会执行。

默认给了两条策略：

| 前缀 | 规则 | 效果 |
|---|---|---|
| `apps/**` | `mostRecentlyPushedCount: 10` | 每个自研镜像保留最近 10 次推送，决定回滚能回多远 |
| `docker/**` | `pulledWithin: "720h"` | 30 天内被拉过的公共镜像保留，长期不用的自动清 |

两条都开了 `deleteUntagged`，清理没有 tag 指向的悬空 manifest。

调整时注意：

- `delay: "24h"` 是删除前的宽限期，防止刚推上去还没被引用的镜像被清掉，不要调到小时级
- `gcInterval: "24h"` 是扫描周期。保留与 GC 按仓库粒度执行，没有全局只读窗口，不必特意和构建时段错开
- 改大 `mostRecentlyPushedCount` 会线性增加磁盘占用
- 拿不准就先把 `retention.dryRun` 设成 `true`，跑一天看日志里会删什么，确认无误再关掉

看 GC 实际删了什么：

```bash
docker logs zot 2>&1 | grep -iE 'retention|gc|delete'
```

## 五、备份

Zot 的全部状态就是文件系统，没有数据库，备份即目录同步。

镜像层每周增量同步：

```bash
rsync -a --delete /srv/registry/data/ /srv/backup/registry/
```

账号口令（bcrypt，单独加密留存）：

```bash
sudo tar czf /srv/backup/registry-conf-$(date +%F).tar.gz -C /srv/registry htpasswd
```

只备 `htpasswd` 就够：`config.json` 与 `compose.yaml` 在本仓库里，`git pull` 随时取回；`htpasswd` 是现场生成的，丢了两台机器都得重新 `docker login`。

恢复：先停容器，再把镜像层与口令放回原处，最后起容器。

```bash
cd /srv/devops/registry && sudo docker compose down
```

```bash
sudo rsync -a /srv/backup/registry/ /srv/registry/data/ && sudo tar xzf /srv/backup/registry-conf-<日期>.tar.gz -C /srv/registry
```

```bash
cd /srv/devops/registry && sudo docker compose up -d
```

`data/docker/` 是代理缓存，丢了会自动回源，磁盘紧张时可以从备份里排除：

```bash
rsync -a --delete --exclude 'docker/' /srv/registry/data/ /srv/backup/registry/
```

镜像的备份优先级低于 Dokploy 数据库 —— 自研镜像可以从 Gitee 的源码重新构建，只是历史版本没了会失去回滚能力。数据分级见 [layout.md](layout.md) 第一节。

## 六、运维命令与故障

`docker compose` 相关命令都在检出目录 `/srv/devops/registry` 下执行 —— compose 认的是当前目录里的 `compose.yaml`。curl 与 du 在哪个目录都行。

配置改动一律走仓库：改本地 → push → 机器 A `git pull` → 重启容器。不要在服务器上就地改 —— `config.json` 是单文件 bind mount，`git pull` 与 `sed -i` 都会换 inode，容器里挂的仍是旧 inode 的内容，不重启不生效；就地改还会让下次 pull 冲突。

| 操作 | 命令 |
|---|---|
| 查看状态 | `sudo docker compose ps` |
| 启动 / 停止 | `sudo docker compose up -d` / `sudo docker compose down` |
| 查看日志 | `sudo docker logs -f zot` |
| 改完 `config.json` 后生效 | 改动 push 回仓库，机器 A `sudo git -C /srv/devops pull`，再 `sudo docker compose restart` |
| 改完 `compose.yaml` 后生效 | 同上，但最后一步必须 `sudo docker compose up -d` —— `restart` 不重建容器，新的 `ports`、`image`、`volumes` 不会生效 |
| 服务器上被就地改过、pull 被拒 | `sudo git -C /srv/devops checkout -- registry/<文件>` 丢掉本地改动后再 pull |
| 存活检查 | `curl -s -o /dev/null -w '%{http_code}\n' http://registry.internal:5000/v2/`，期望 `401` |
| 磁盘占用 | `sudo du -sh /srv/registry/data/*` |
| 列出仓库 | `curl -s -u ci http://registry.internal:5000/v2/_catalog` |
| 列出某镜像的 tag | `curl -s -u ci http://registry.internal:5000/v2/apps/<app>/tags/list` |
| 检查隧道 | `sudo wg show` |

容器镜像基于 scratch，**里面没有 shell**，`docker exec` 进不去，也写不了 shell 形式的 healthcheck。所有检查都在宿主机用 curl 做。

升级：改仓库里 `registry/compose.yaml` 的 `ZOT_VERSION` 或镜像 tag，push 后在机器 A `git pull`，再 `docker compose up -d`。Zot 的存储布局是 OCI 标准，小版本升级不需要迁移工具；跨大版本前先备份 `data/` 与 config。

### 常见故障

| 现象 | 原因 |
|---|---|
| 容器反复重启，日志报绑定地址失败 | wg0 没起来，`sudo systemctl restart wg-quick@wg0` 后再起容器 |
| `docker push` 报 401 | 账号在 htpasswd 里但没在 `accessControl` 里，或策略漏了 `read` |
| `docker pull` 公共镜像超时、`data/` 下不生成 `docker/` 目录 | 机器 A 到上游不通。典型现象：`getent hosts registry-1.docker.io` 返回 `2a03:2880::` 段的 Meta 地址（DNS 污染），IPv4 的 443 超时，而访问别的境外站点正常。换掉 `extensions.sync.registries` 里对应那条的 `urls` 主地址，挑法见第七节 |
| `docker pull` 报 `EOF`、`data/docker/` 里却在长大 | 首次回源同步阻塞，客户端等不及先断了。**不是失败**，Zot 日志有 `successfully synced image` 就重拉一次，见第七节 |
| 某个仓库回源报 403 / 404，同批其他镜像正常 | 上游没代理到这个仓库。上游对匿名 `curl` 返回 401 属正常（要 token），403 是 Zot 换到 token 之后回源才暴露的，只能看 `docker logs zot` 里的 `failed to get upstream image manifest details`。修法是给这个仓库单开一条 `registries` 规则指向别的上游，见第七节 |
| `docker pull` 公共镜像被限速 | Docker Hub 匿名拉取有速率限制，短时间大量回源会被限；等待或给 sync 配上游账号 |
| 推送成功但 UI 看不到 | `search` 扩展被关掉了，UI 依赖它建索引 |
| 客户端报 manifest 格式不支持 | `http.compat` 里的 `docker2s2` 被删了，老 Docker 客户端需要它 |

## 七、回源上游

`extensions.sync.registries` 是一个**有序数组**，每条规则由「上游地址」和「管哪些仓库」两部分组成。当前是两条：

| 顺序 | 上游 | `content.prefix` | 管什么 |
|---|---|---|---|
| 1 | `docker.1panel.live` | `makeplane/**` | Plane 的 6 个镜像 |
| 2 | `docker.m.daocloud.io` | `**` | 其余全部公共镜像 |

拆成两条是因为 daocloud 回源不到 `makeplane/*`（`library/*`、`valkey/*` 正常），而它已经缓存的那些没必要推倒重来。

匹配是按顺序走的：`**` 能匹配一切，所以 makeplane 两条都命中，先试第 1 条，失败再落到第 2 条 —— 等于自带降级。新增规则要**插在 `**` 那条之前**，放后面永远轮不到。

`content` 三个字段的配合决定本地仓库名怎么映射回上游：

| 字段 | 含义 |
|---|---|
| `prefix` | 匹配**上游**仓库名，`**` 跨层级通配 |
| `destination` | 存到本地哪个前缀下，两条都是 `/docker` |
| `stripPrefix` | 映射时是否砍掉 `prefix` 里的字面量部分 |

`prefix: "**"` 没有字面量部分，`stripPrefix` 填 `true` 或 `false` 都一样；`prefix: "makeplane/**"` 的字面量是 `makeplane/`，要保留它才能拼出 `docker/makeplane/plane-proxy`，所以必须 `false`。填反了本地会变成 `docker/plane-proxy`，Plane 的 compose 就对不上了。

### 不要配多个 URL 兜底

一条规则里写多个 `urls` **不是逐镜像回退**，而且有实打实的代价，两件事都反直觉：

第一，它不解决问题。Zot 靠 `/v2/` 探活挑一个能连的用，上游整体活着、只是某个仓库回源失败时不会切换 —— daocloud 正是这种情况，`/v2/` 好好的，就是不给 `makeplane/*`。

第二，它会拖慢每一次拉取。**镜像已在缓存里时，Zot 仍会回上游校验一次是否最新**（日志：`sync: syncing image` → `skipping image because it's already synced`）。曾给第 2 条加过 `https://index.docker.io` 兜底，而机器 A 根本连不上 Docker Hub，结果每个非 makeplane 镜像的拉取都要多等一个 `retryDelay`：

| 第 2 条的 urls | 缓存命中后的拉取耗时 |
|---|---|
| `[daocloud, index.docker.io]` | **30.1 秒**（正好 `retryDelay: "30s"`） |
| `[daocloud]` | **1.0~1.6 秒** |

10 个镜像的栈就是白等 2 分钟，而且这份额外延迟叠加在并发部署上，足以把本来能过的拉取重新推回 EOF。

**结论：一条规则只配一个 URL。** 要换上游就改这个地址，或者按仓库前缀新开一条规则。

### 挑上游

候选站挨个测，`makeplane/plane-proxy` 是已知 daocloud 出不了的那个，`library/alpine` 作对照：

```bash
for m in docker.m.daocloud.io docker.1ms.run docker.xuanyuan.me docker.1panel.live hub.rat.dev index.docker.io; do printf '%-24s ' "$m"; printf 'makeplane=%-4s ' "$(curl -s -m 15 -o /dev/null -w '%{http_code}' -H 'Accept: application/vnd.oci.image.index.v1+json,application/vnd.docker.distribution.manifest.list.v2+json' https://$m/v2/makeplane/plane-proxy/manifests/v1.4.1)"; printf 'library=%s\n' "$(curl -s -m 15 -o /dev/null -w '%{http_code}' -H 'Accept: application/vnd.oci.image.index.v1+json,application/vnd.docker.distribution.manifest.list.v2+json' https://$m/v2/library/alpine/manifests/3.20)"; done
```

| 返回 | 含义 |
|---|---|
| `200` | 匿名直出，可用 |
| `401` | 要 token，Zot 会自己换，也算可用；但**不代表回源一定成功**，daocloud 就是 401 之后才 403 的 |
| `403` | 上游明确拒绝这个仓库 |
| `429` | 在限流，不可靠 |
| `000` | 机器 A 连不上 |

只有 `200`/`401` 值得试，且改完必须实拉一个镜像验证，光看状态码不够。

### 首次拉取报 EOF 是正常的

这一条最反直觉，排障时不知道会白跑很久。

`onDemand: true` 的回源是**同步阻塞**的：本地没有该镜像时，Zot 会挂住这个 HTTP 请求，先把整个镜像从上游同步到本地，再回应客户端。而且它同步的是 manifest index 下的**全部条目** —— 以 `plane-proxy:v1.4.1` 为例，缓存里落了 `linux/amd64`、`linux/arm64` 和两个 attestation，机器 A 只用得上第一个，下载量是实际所需的两倍多。

首次拉取的等待因此远超 docker 客户端的忍耐度，客户端先断开，报成：

```
Error response from daemon: Head "http://registry.internal:5000/v2/docker/...": EOF
```

**这条 EOF 不代表回源失败。** 实测 `makeplane/plane-proxy:v1.4.1`：客户端 1m48s 断开报 EOF，而同一时刻 Zot 日志里是

```
successfully synced image  repo=docker/makeplane/plane-proxy  reference=v1.4.1
HEAD /v2/docker/makeplane/plane-proxy/manifests/v1.4.1  statusCode=200  latency=1m47s
```

镜像已经进缓存，紧接着重拉一次 5.5 秒完成。

所以**判断回源通没通不能看 docker 的报错，要看 Zot 日志**：

| Zot 日志 | 含义 | 处理 |
|---|---|---|
| `successfully synced image` | 通了，只是客户端等不及 | 重拉一次即可 |
| `failed to sync image` + `unauthorized access` | 上游不给这个仓库 | 换上游，见上文 |
| 什么都没有 | 请求根本没到 Zot | 查隧道、查 `insecure-registries` |

### 换上游之后：带重试预热

改 `config.json` → push → 机器 A `sudo git -C /srv/devops pull` → `cd /srv/devops/registry && sudo docker compose restart`，然后**串行预热，且必须带重试** —— 只跑一遍会被首次 EOF 全部绊倒：

```bash
for i in makeplane/plane-proxy:v1.4.1 makeplane/plane-backend:v1.4.1 makeplane/plane-frontend:v1.4.1 makeplane/plane-space:v1.4.1 makeplane/plane-admin:v1.4.1 makeplane/plane-live:v1.4.1 library/postgres:15.7-alpine library/rabbitmq:3.13.6-management-alpine valkey/valkey:7.2.11-alpine minio/minio:latest; do for t in $(seq 10); do sudo docker pull registry.internal:5000/docker/$i >/dev/null 2>&1 && { echo "OK   $i（第 $t 次）"; break; }; [ $t = 10 ] && echo "FAIL $i"; sleep 5; done; done
```

十个镜像全部回源约 20~40 分钟，`nohup ... &` 挂后台跑，`du -sh /srv/registry/data/docker/*` 看进度。

预热完再让 Dokploy 部署。跳过预热的话，`docker compose` 会一次并发触发十几个同步阻塞的回源，全部 EOF，一个失败就 cancel 掉同批其余的（日志里显示成 `Interrupted`），部署直接失败 —— 而真实原因被 EOF 完全盖住。

### 信任

换第三方公共镜像站等于把回源信任交出去。`preserveDigest: true` 只保证存下来的内容和 manifest 的 digest 自洽，**不校验这个 manifest 是否就是 Docker Hub 上的那个**。要紧的镜像从 Docker Hub 页面抄下官方 digest，拉完后比一下：

```bash
sudo docker inspect --format '{{index .RepoDigests 0}}' registry.internal:5000/docker/makeplane/plane-proxy:v1.4.1
```
