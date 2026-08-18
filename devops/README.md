# Docker 部署操作手册（Dokploy + Harbor）

## 一、环境要求

### 机器

| | 机器 A | 机器 B |
|---|---|---|
| 职责 | 自动化部署：Dokploy 控制平面 + 镜像构建 + Harbor | 应用服务运行 |
| 配置 | 4 核 / 8 GB / 120 GB，另挂 4 GB swap | 4 核 / 8 GB / 200 GB，另挂 4 GB swap |
| 装什么 | Dokploy（UI、builder、PostgreSQL、Redis、Traefik）、Harbor | Docker + Traefik（由 Dokploy 自动装） |
| 对公网 | 80（仅 webhook 端点）、22、UDP 51820 | 80（各应用域名） |
| 隧道地址 | `10.8.0.1`（`harbor.internal`） | `10.8.0.2` |
| 域名 | `ci.example.com` → 机器 A，HTTP | 各应用域名 → 机器 B，HTTP |

机器 B 不装 nginx、不装 certbot、不放源码、不放编排文件 —— Dokploy 通过 SSH 下发 compose 与 `.env`，Traefik 自动签发证书。

### 机器间通信

两台机器只能走公网，必须先建 WireGuard 隧道：机器 B 从 Harbor 拉镜像走隧道，Harbor 的 8081 不对公网开放。Dokploy 控制机器 B 走 SSH（本身已加密），不依赖隧道。

| 项 | 值 |
|---|---|
| 隧道网段 | `10.8.0.0/24` |
| 隧道端口 | 机器 A 放行 UDP `51820`，来源限机器 B 公网 IP |

### 全站 HTTP 的暴露面控制

本方案不启用 HTTPS。为避免管理密码明文过公网，**Dokploy 面板不从域名访问，而是走 SSH 隧道**；公网上的 `http://ci.example.com` 只用于接收 Gitee 的 webhook 推送。

| 用途 | 访问方式 |
|---|---|
| Dokploy 面板 UI | `ssh -L 3000:localhost:3000 user@<机器A公网IP>` → `http://localhost:3000` |
| Gitee webhook | `http://ci.example.com/api/deploy/webhook/<APPLICATION_ID>` |
| Harbor UI | `ssh -L 8081:harbor.internal:8081 user@<机器A公网IP>` → `http://localhost:8081` |

这样公网只暴露一个触发部署的端点，且它只能部署仓库里已有的代码，注入不了任意镜像。仍需做的加固：面板开 2FA、Harbor 与 Dokploy 都用强密码。

残留风险需知晓：应用侧（含 Plane）的用户密码在 HTTP 下明文传输；浏览器会在密码框标记「不安全」，并禁用剪贴板、Service Worker 等仅限安全上下文的 API。后续切 HTTPS 的成本很低 —— Dokploy 里给域名勾上 Let's Encrypt，再把 Plane 的 `WEB_URL`、`CORS_ALLOWED_ORIGINS` 改成 `https://`、`MINIO_ENDPOINT_SSL` 改为 `1` 即可。

### 软件

- Docker Engine ≥ 20.10，Compose v2
- Dokploy：需 ≥ 2 GB 内存、30 GB 磁盘，占用 80 / 443 / 3000 端口
- Harbor v2.15.2
- Gitee：只需仓库的 WebHooks 与部署公钥功能，**不需要 Gitee Go**

### 内存分配

| 机器 A | 常驻 | 机器 B | 常驻 |
|---|---|---|---|
| 系统 + Docker | 0.7 GB | 系统 + Docker | 0.7 GB |
| Dokploy 控制平面 | 0.3~1 GB | Traefik | 0.1 GB |
| Harbor 11 容器 | 2 GB | Plane 13 容器 | 3 GB |
| `docker build` 峰值 | +2~4 GB | 每个自研应用 | 0.5~1 GB |
| 余量 | ~1~2 GB（构建时） | 余量 | ~4.2 GB |

构建全部发生在机器 A，机器 B 不承担构建峰值。机器 A 在构建与 Harbor GC 同时发生时最紧，两者定时错开并挂 swap。

### 磁盘分配

| 机器 A | 额度 | 机器 B | 额度 |
|---|---|---|---|
| 系统 | 15 GB | 系统 | 15 GB |
| Harbor 数据 `/srv/harbor` | 70 GB | `/var/lib/docker` | 40 GB |
| Dokploy 数据与构建缓存 `/etc/dokploy` | 30 GB | docker 命名卷（业务数据） | 100 GB |
| 备份 | 20 GB | 备份 | 40 GB |

两台机器都加 crontab 清理旧镜像与构建缓存：

```bash
0 4 * * 0 docker builder prune -f --filter until=168h && docker image prune -a -f --filter until=336h
```

## 二、三处集中管理

| 对象 | 位置 | 说明 |
|---|---|---|
| **源代码** | Dokploy（机器 A） | 所有仓库由 Dokploy 统一拉取，一处 SSH 密钥、一处 webhook 配置。机器 B 不放源码 |
| **编排** | 部署仓库 `projects/<project>/docker-compose.yml` | Dokploy 的 Compose 服务用 git provider 拉取，编排仍受版本控制 |
| **docker 镜像** | Harbor（机器 A） | 自研镜像推 `apps` 项目；公共镜像走 `dockerhub` 代理缓存。Dokploy 里配一次 registry，所有服务共用 |
| **环境变量** | Dokploy 的变量层级 | 项目共享变量（等价 common.env）+ 服务变量（等价 project.env），服务内用 `${{project.VAR}}` 引用，加密存在 Dokploy 的 PostgreSQL 里 |

Harbor 无法作为透明 registry mirror（Docker 的 `registry-mirrors` 不接受路径前缀），公共镜像用显式地址：compose 里写 `${REGISTRY_CACHE:-docker.io}/library/postgres:15.7-alpine`，留空则回落直连。

环境变量从文件搬进了 Dokploy 的数据库，因此**机器 A 的 Dokploy 备份等级与 Harbor 数据库同高** —— 丢了它等于丢掉全部部署配置。备份见第七节。

## 三、目录结构

### 部署仓库（`devops/deploy-repo/` 即其骨架）

```
<deploy-repo>/
├── projects/
│   ├── app-example/docker-compose.yml   # 多容器自研应用模板
│   └── plane/docker-compose.yaml        # Plane 编排
└── env/                                 # 变量清单，供在 Dokploy UI 里填写时对照
    ├── common.env.example
    ├── app-example.env.example
    └── plane.env.example
```

这个仓库不需要克隆到任何服务器，Dokploy 直接从 Gitee 拉取。

### 应用仓库（`devops/template/` 即其骨架）

```
<project>/
├── src/
├── docker/
│   ├── Dockerfile                       # Dokploy 用它构建，测试也放构建阶段
│   └── nginx.conf                       # 前端项目用，后端删掉
├── .dockerignore
├── .editorconfig
└── .gitattributes
```

不含流水线定义 —— 触发由 Gitee webhook 直达 Dokploy。

### 机器 A

```
/opt/harbor/          # Harbor 安装包与 harbor.yml
/etc/dokploy/         # Dokploy 数据、源码检出、构建缓存
/srv/
├── harbor/           # Harbor 数据（data_volume）
└── backup/
```

### 机器 B

```
/var/lib/docker/      # 镜像与命名卷，业务数据都在这里
/srv/backup/
```

机器 B 上没有需要人工维护的目录，全部由 Dokploy 下发。

## 四、机器 A 操作步骤

1. 安装 Docker 与 Compose v2，挂 4 GB swap

```bash
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
```

2. 安装 WireGuard 并生成密钥对

```bash
sudo apt install -y wireguard && umask 077 && wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

3. 写 `/etc/wireguard/wg0.conf`（两台机器先各自生成密钥再交换公钥）

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <机器A私钥>

[Peer]
PublicKey = <机器B公钥>
AllowedIPs = 10.8.0.2/32
```

4. 启动隧道，放行隧道端口

```bash
sudo systemctl enable --now wg-quick@wg0 && sudo ufw allow from <机器B公网IP> to any port 51820 proto udp
```

5. 配主机名与 Docker 配置

```bash
echo '10.8.0.1 harbor.internal' | sudo tee -a /etc/hosts
```

`/etc/docker/daemon.json`：

```json
{
  "insecure-registries": ["harbor.internal:8081"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

```bash
sudo systemctl restart docker
```

6. 装 Dokploy（会执行 `docker swarm init` 并装 Traefik，占用 80/443/3000）

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

> 本机若已有 Docker Swarm，**不要**用此脚本，它会强制退出当前 swarm 重新初始化。Harbor 用的是 compose，不受影响。

7. 建 SSH 隧道后在浏览器打开 `http://localhost:3000` 创建管理员账号并开启 2FA

```bash
ssh -L 3000:localhost:3000 user@<机器A公网IP>
```

8. Settings → Server → Domain 绑定 `ci.example.com`，**不勾选 Let's Encrypt**（webhook 走 HTTP）

9. 关闭 3000 端口的公网访问，面板此后只经 SSH 隧道访问

```bash
sudo ufw deny 3000/tcp && sudo ufw allow 80/tcp
```

10. 按 [harbor/README.md](harbor/README.md) 部署 Harbor（`hostname: harbor.internal`、`http.port: 8081`），建私有项目 `apps`、代理缓存项目 `dockerhub`、`ci` 与 `deploy` 两个机器人账号，配保留策略与 GC

11. 防火墙把 8081 锁进隧道

```bash
sudo ufw allow in on wg0 to any port 8081 proto tcp && sudo ufw deny 8081/tcp
```

12. 登录 Harbor（构建推送用）

```bash
docker login harbor.internal:8081 -u 'robot$apps+ci'
```

13. 在 Dokploy 里配 Registry：Settings → Registry → Add，Registry URL 填 `harbor.internal:8081`，用户名 `robot$apps+ci`，密码填机器人 token

14. 在 Dokploy 里配 Git：Settings → Git → 添加 Custom Git，生成 SSH 密钥，把公钥加到 Gitee 各仓库的「管理 → 部署公钥」

15. crontab 加清理任务

```bash
0 4 * * 0 docker builder prune -f --filter until=168h && docker image prune -a -f --filter until=336h
```

## 五、机器 B 操作步骤

1. 安装 Docker 与 Compose v2
2. 安装 WireGuard 并生成密钥对

```bash
sudo apt install -y wireguard && umask 077 && wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

3. 写 `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = <机器B私钥>

[Peer]
PublicKey = <机器A公钥>
Endpoint = <机器A公网IP>:51820
AllowedIPs = 10.8.0.1/32
PersistentKeepalive = 25
```

4. 启动隧道并验证连通

```bash
sudo systemctl enable --now wg-quick@wg0 && ping -c 3 10.8.0.1
```

5. 配主机名与 `insecure-registries`（内容同机器 A 第 5 步），重启 Docker

```bash
echo '10.8.0.1 harbor.internal' | sudo tee -a /etc/hosts
```

6. 登录 Harbor（拉取用，经隧道）

```bash
docker login harbor.internal:8081 -u 'robot$apps+deploy'
```

7. 创建可 SSH 登录的部署用户并放好机器 A 的公钥（Dokploy 用它下发命令），确认 root 或 sudo 免密可用

8. 回到机器 A 的 Dokploy：Settings → Servers → Create Server，填机器 B 公网 IP、SSH 端口、用户名与私钥，保存后点 **Setup Server**，Dokploy 会在机器 B 装 Docker 与 Traefik

9. 各应用域名的 DNS 解析指向机器 B 公网 IP，放行 80

```bash
sudo ufw allow 80/tcp
```

## 六、新项目接入步骤

**应用仓库侧：**

1. 把 `template/` 内容复制到仓库根目录，按语言调整 `docker/Dockerfile`

**Dokploy 侧（单容器应用用 Application 类型）：**

2. Create Project（如 `prod`），在 Project → Shared Environment Variables 填公共变量（对照 `env/common.env.example`）
3. Create Service → Application，Server 选**机器 B**，Provider 选 Custom Git，填仓库 SSH 地址与分支，Build Type 选 Dockerfile 并指向 `docker/Dockerfile`
4. Environment 里填服务变量，公共项用 `${{project.VAR}}` 引用
5. Domains 里加域名与容器端口，**不勾选 Let's Encrypt**
6. General 里打开 **Auto Deploy**，复制 Webhook URL（形如 `http://ci.example.com/api/deploy/webhook/<ID>`）
7. Gitee 仓库 → 管理 → WebHooks → 添加，URL 粘贴上一步的地址，事件选 Push

**多容器应用或第三方 compose 栈用 Compose 类型：**

3'. 在部署仓库加 `projects/<project>/docker-compose.yml`（参照 `app-example`，网络加 `dokploy-network` external）
4'. Create Service → Compose，Provider 选 Git 指向部署仓库与该文件路径，Server 选机器 B
5'. Environment 里填变量，Dokploy 会写成 `.env` 放在 compose 同目录供插值
6'. Domains 里选择要暴露的 service 名与端口，同样不勾 Let's Encrypt

首次部署点 Deploy，之后 push 即自动触发。

## 七、日常操作

| 操作 | 位置 |
|---|---|
| 部署 / 重新部署 | Dokploy 服务页 → Deploy |
| 回滚 | Dokploy → Deployments，选历史镜像 tag 回滚 |
| 查看构建与运行日志 | Dokploy 服务页 → Logs |
| 改环境变量 | Dokploy → Environment，保存后重新部署 |
| 改编排 | 提交到部署仓库，Dokploy 重新部署时自动拉取新版本 |
| 访问 Dokploy 面板 | `ssh -L 3000:localhost:3000 user@<机器A公网IP>`，浏览器开 `http://localhost:3000` |
| 访问 Harbor UI | `ssh -L 8081:harbor.internal:8081 user@<机器A公网IP>`，浏览器开 `http://localhost:8081` |
| 检查隧道 | `sudo wg show`（看 latest handshake 与流量计数） |
| 隧道重连 | `sudo systemctl restart wg-quick@wg0` |

### 备份（机器 A，两项都是不可再生数据）

Dokploy 配置（含全部环境变量与服务定义）：

```bash
docker exec $(docker ps -qf name=dokploy-postgres) pg_dumpall -U dokploy | gzip > /srv/backup/dokploy-$(date +%F).sql.gz
```

Harbor 数据库与镜像层见 [harbor/README.md](harbor/README.md) 第六节。两者都加进每日 crontab。

### 已知问题

Dokploy 用私有 registry 时，**回滚不会执行 `docker login`**（[issue #3861](https://github.com/Dokploy/dokploy/issues/3861)）。若机器 B 的 Harbor 凭据失效或重装过系统，回滚会失败。定期确认机器 B 的 `~/.docker/config.json` 仍有效，重装后先手工 `docker login` 再回滚。

## 八、Plane 部署

Plane 用 Compose 类型接入，编排在部署仓库 `projects/plane/docker-compose.yaml`，13 个镜像全部经 Harbor 的 `dockerhub` 代理缓存拉取。变量清单与初始化步骤见 [../plane-docker-deploy.md](../plane-docker-deploy.md)。
