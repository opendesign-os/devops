# 镜像仓库参考（Zot，隧道内 HTTP）

Zot 是 CNCF 的 OCI 原生镜像仓库，单进程单容器，常驻内存 100~200 MB。
它在本方案里同时承担两件事：存自研镜像（`apps/`）、代理缓存公共镜像（`docker/`）。

**安装步骤不在本文。** 克隆仓库、建目录、生成账号、启动、验证、登录，全部在 [deploy.md](deploy.md) 第五节，按那里一路做下来即可。本文只讲配置项含义、授权规则、保留策略调参与日常运维。

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
| 回源上游 | `docker.m.daocloud.io` 优先、`index.docker.io` 兜底 —— 大陆机器直连 Docker Hub 基本不通 |

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

`data/docker/` 是代理缓存，丢了会自动回源，磁盘紧张时可以从备份里排除：

```bash
rsync -a --delete --exclude 'docker/' /srv/registry/data/ /srv/backup/registry/
```

镜像的备份优先级低于 Dokploy 数据库 —— 自研镜像可以从 Gitee 的源码重新构建，只是历史版本没了会失去回滚能力。数据分级见 [layout.md](layout.md) 第一节。

## 六、运维命令与故障

`docker compose` 相关命令都在检出目录 `/srv/devops/registry` 下执行 —— compose 认的是当前目录里的 `compose.yaml`。curl 与 du 在哪个目录都行。

| 操作 | 命令 |
|---|---|
| 查看状态 | `sudo docker compose ps` |
| 启动 / 停止 | `sudo docker compose up -d` / `sudo docker compose down` |
| 查看日志 | `sudo docker logs -f zot` |
| 改完 config.json 后生效 | 先把改动 push 回仓库，再 `sudo git -C /srv/devops pull && sudo docker compose restart` |
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
| `docker pull` 公共镜像超时、`data/` 下不生成 `docker/` 目录 | 机器 A 到上游不通。典型现象：`getent hosts registry-1.docker.io` 返回 `2a03:2880::` 段的 Meta 地址（DNS 污染），IPv4 的 443 超时，而访问别的境外站点正常。换上游即可：`config.json` 的 `extensions.sync.registries[0].urls` 按顺序尝试，默认已把国内镜像 `https://docker.m.daocloud.io` 放在 `https://index.docker.io` 之前；有阿里云镜像加速器地址（控制台 → 容器镜像服务 → 镜像加速器）就换成它，同厂商内网更稳 |
| `docker pull` 公共镜像被限速 | Docker Hub 匿名拉取有速率限制，短时间大量回源会被限；等待或给 sync 配上游账号 |
| 推送成功但 UI 看不到 | `search` 扩展被关掉了，UI 依赖它建索引 |
| 客户端报 manifest 格式不支持 | `http.compat` 里的 `docker2s2` 被删了，老 Docker 客户端需要它 |
