# 镜像仓库操作手册（Zot，隧道内 HTTP）

Zot 是 CNCF 的 OCI 原生镜像仓库，单进程单容器，常驻内存 100~200 MB。
它在本方案里同时承担两件事：存自研镜像（`apps/`）、代理缓存公共镜像（`docker/`）。

## 一、环境要求

| 项 | 要求 |
|---|---|
| 版本 | Zot v2.1.20，镜像 `ghcr.io/project-zot/zot-linux-amd64` |
| 机器 | 机器 A，与 Dokploy 同机 |
| 内存 | 常驻 100~200 MB；开启 CVE 扫描后另计 |
| 磁盘 | `/srv/registry/data` 预留 60 GB |
| **前置** | **两台机器间的 WireGuard 隧道已建立**，见 [../README.md](../README.md) 第四、五节 |
| 主机名 | `registry.internal` → 机器 A 隧道地址 `10.8.0.1` |
| 端口 | 5000，只绑定在隧道地址上 |
| 客户端 | 两台机器 daemon 都要配 `insecure-registries` |

必须用 **full 版镜像**（`zot-linux-amd64`）。名字带 `minimal` 的那个不含任何扩展，代理缓存、UI、保留策略全都没有。

两台机器只能走公网，仓库因此跑在隧道内：流量由 WireGuard 加密，HTTP 明文只存在于隧道内部，`insecure-registries` 的豁免也仅限隧道地址。**不要跳过隧道直接把 5000 开到公网** —— `docker login` 凭据是 base64 明文，截获即可推镜像投毒。

本方案不给端口加防火墙规则，而是让容器**只绑定 `10.8.0.1:5000`**。端口从一开始就没监听在公网网卡上，比"开了再用 ufw 堵"更难出事。代价是 wg0 必须先于 Docker 起来，否则容器绑不到地址会反复重启。

## 二、安装步骤

### 步骤 1：配主机名指向隧道地址（两台机器都做）

```bash
echo '10.8.0.1 registry.internal' | sudo tee -a /etc/hosts
```

### 步骤 2：配 insecure-registries（两台机器都做）

写 `/etc/docker/daemon.json`：

```json
{
  "insecure-registries": ["registry.internal:5000"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

```bash
sudo systemctl restart docker
```

### 步骤 3：建目录（机器 A）

```bash
sudo mkdir -p /srv/registry/data && sudo chmod 755 /srv/registry
```

容器以 root 运行，数据目录不需要额外 chown。

### 步骤 4：生成账号

Zot 只认 **bcrypt**，`-B` 不能省。三个账号一次生成：

```bash
sudo apt install -y apache2-utils
```

```bash
htpasswd -Bn admin | sudo tee /srv/registry/htpasswd
```

```bash
htpasswd -Bn ci | sudo tee -a /srv/registry/htpasswd
```

```bash
htpasswd -Bn deploy | sudo tee -a /srv/registry/htpasswd
```

`-Bn` 会交互式提示输入密码，不留 shell 历史。第一条用 `tee` 覆盖建文件，后两条用 `tee -a` 追加，别写反。

机器上没有 `htpasswd` 命令时用容器生成，注意这种写法密码会进 shell 历史，事后 `history -c`：

```bash
docker run --rm httpd:2.4-alpine htpasswd -bBn ci '<密码>' | sudo tee -a /srv/registry/htpasswd
```

### 步骤 5：放配置与编排

把本目录的 [config.json](config.json) 与 [compose.yaml](compose.yaml) 一起复制到机器 A：

```bash
sudo cp config.json compose.yaml /srv/registry/
```

配置里已经写好了三个账号的授权、两条保留策略和 Docker Hub 代理缓存，**默认值可直接用**。要改的话看第五节。

### 步骤 6：启动

```bash
cd /srv/registry && sudo docker compose up -d
```

首次拉取 zot 镜像约 100 MB，秒级启动。

### 步骤 7：验证

未认证访问应返回 `401`，说明服务活着且鉴权生效：

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://registry.internal:5000/v2/
```

从任意外部机器验证公网不可达，应超时或被拒绝：

```bash
curl -m 5 http://<机器A公网IP>:5000/v2/
```

两台机器各登录一次：

```bash
docker login registry.internal:5000 -u ci
```

```bash
docker login registry.internal:5000 -u deploy
```

机器 A 登 `ci`（推送用），机器 B 登 `deploy`（拉取用）。

### 步骤 8：验证代理缓存

在机器 B 上拉一个公共镜像，走的是 `docker/` 前缀：

```bash
docker pull registry.internal:5000/docker/library/alpine:3.20
```

第一次会回源 Docker Hub（慢），第二次直接命中本地。

## 三、账号与授权

授权规则写在 `config.json` 的 `http.accessControl` 里，改完重启容器生效。

| 账号 | `apps/**` | `docker/**` | 登录位置 |
|---|---|---|---|
| `ci` | read + create + update + delete | read | 机器 A，Dokploy 构建后推送 |
| `deploy` | read | read | 机器 B，拉取运行 |
| `admin` | 全部（走 `adminPolicy`） | 全部 | 仅登录 UI 用 |

`defaultPolicy` 与 `anonymousPolicy` 都是空数组，含义是**未匹配到策略的已认证用户、以及所有匿名请求，一律无权限**。新增账号必须同时在 htpasswd 和 accessControl 里各加一处，只加一处不会生效。

Zot 的策略里 `read` 是其他动作的前提，写 `["create"]` 而不带 `read` 属于无效配置。

## 四、Web UI

UI 由 `extensions.ui` 提供，和 API 同端口，不额外占端口。管理员经 SSH 隧道访问：

```bash
ssh -L 5000:10.8.0.1:5000 user@<机器A公网IP>
```

浏览器打开 `http://localhost:5000`，用 `admin` 账号登录。

UI 的漏洞面板依赖 `extensions.search` 的 CVE 能力，开启后会定期下载并更新漏洞库，额外占磁盘与内存。机器 A 资源紧张时，把 `search` 与 `ui` 两段一起从 config 里删掉即可，镜像存取与代理缓存完全不受影响。

## 五、保留策略与 GC

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

## 六、备份

Zot 的全部状态就是文件系统，没有数据库，备份即目录同步。

镜像层每周增量同步：

```bash
rsync -a --delete /srv/registry/data/ /srv/backup/registry/
```

配置与账号（含 bcrypt 口令，单独加密留存）：

```bash
sudo tar czf /srv/backup/registry-conf-$(date +%F).tar.gz -C /srv/registry config.json htpasswd compose.yaml
```

`data/docker/` 是代理缓存，丢了会自动回源，磁盘紧张时可以从备份里排除：

```bash
rsync -a --delete --exclude 'docker/' /srv/registry/data/ /srv/backup/registry/
```

镜像的备份优先级低于 Dokploy 数据库 —— 自研镜像可以从 Gitee 的源码重新构建，只是历史版本没了会失去回滚能力。数据分级见 [../layout.md](../layout.md) 第一节。

## 七、运维命令

均在 `/srv/registry` 下执行。

| 操作 | 命令 |
|---|---|
| 查看状态 | `sudo docker compose ps` |
| 启动 / 停止 | `sudo docker compose up -d` / `sudo docker compose down` |
| 查看日志 | `sudo docker logs -f zot` |
| 改完 config.json 后生效 | `sudo docker compose restart` |
| 存活检查 | `curl -s -o /dev/null -w '%{http_code}\n' http://registry.internal:5000/v2/`，期望 `401` |
| 磁盘占用 | `sudo du -sh /srv/registry/data/*` |
| 列出仓库 | `curl -s -u ci http://registry.internal:5000/v2/_catalog` |
| 列出某镜像的 tag | `curl -s -u ci http://registry.internal:5000/v2/apps/<app>/tags/list` |
| 检查隧道 | `sudo wg show` |

容器镜像基于 scratch，**里面没有 shell**，`docker exec` 进不去，也写不了 shell 形式的 healthcheck。所有检查都在宿主机用 curl 做。

升级：改 `compose.yaml` 里的 `ZOT_VERSION` 或镜像 tag，然后 `docker compose up -d`。Zot 的存储布局是 OCI 标准，小版本升级不需要迁移工具；跨大版本前先备份 `data/` 与 config。

### 常见故障

| 现象 | 原因 |
|---|---|
| 容器反复重启，日志报绑定地址失败 | wg0 没起来，`sudo systemctl restart wg-quick@wg0` 后再起容器 |
| `docker push` 报 401 | 账号在 htpasswd 里但没在 `accessControl` 里，或策略漏了 `read` |
| `docker pull` 公共镜像超时 | Docker Hub 匿名拉取有速率限制，短时间大量回源会被限；等待或给 sync 配上游账号 |
| 推送成功但 UI 看不到 | `search` 扩展被关掉了，UI 依赖它建索引 |
| 客户端报 manifest 格式不支持 | `http.compat` 里的 `docker2s2` 被删了，老 Docker 客户端需要它 |

## 八、替代方案：不建隧道，改用公网 HTTPS

只有在无法建立隧道时才用这条路，仓库会暴露在公网扫描下。

机器 A 上 80/443 已被 Dokploy 的 Traefik 占用，因此**不要让 Zot 自己签证书**，而是让 Traefik 反代它：

1. `compose.yaml` 里端口改成 `"127.0.0.1:5000:5000"`，并接入 `dokploy-network`
2. `registry.example.com` 解析到机器 A 公网 IP
3. 在 Dokploy 里把该域名反代到 Zot 容器的 5000，勾选 Let's Encrypt。反代需放开请求体限制并延长超时，否则推大镜像会失败
4. 两台机器的 `daemon.json` **删掉** `insecure-registries`
5. 项目共享变量里 `REGISTRY` 改成 `registry.example.com`、`REGISTRY_CACHE` 改成 `registry.example.com/docker`（都去掉 `:5000`），两台机器重新 `docker login`
6. 确认 `anonymousPolicy` 仍为空数组 —— 公网暴露下，匿名可读等于把镜像公开

Zot 也支持在 `http.tls` 里直接配证书，但那样要自己解决续期，不如复用已有的 Traefik。
