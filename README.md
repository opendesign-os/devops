# 部署手册（Dokploy + Zot + Gitee）

机器 A 是控制平面（Dokploy 构建 + Zot 镜像仓库），机器 B 跑应用。全站 HTTP，管理面板只经 SSH 隧道访问。

| 文档 | 内容 |
|---|---|
| 本文 | 两台机器的部署操作步骤 |
| [layout.md](layout.md) | 代码、部署文件、变量、镜像、CI/CD 五类资产的存放规范 |
| [registry/README.md](registry/README.md) | Zot 镜像仓库的安装、授权与运维 |
| [apps/plane.md](apps/plane.md) | Plane 的变量配置与初始化 |

| 目录 | 对应 |
|---|---|
| `deploy/` | 部署仓库骨架，由 Dokploy 从 Gitee 拉取，不需克隆到服务器 |
| `template/` | 应用仓库骨架，复制到每个新项目 |
| `registry/` | 复制到机器 A 的 `/srv/registry/` |

## 一、前提

| | 机器 A | 机器 B |
|---|---|---|
| 配置 | 4 核 / 8 GB / 120 GB | 4 核 / 8 GB / 200 GB |
| 隧道地址 | `10.8.0.1`（`registry.internal`） | `10.8.0.2` |
| 对公网 | 80、22、UDP 51820 | 80 |
| 域名 | `ci.example.com` | 各应用域名 |

- Docker Engine ≥ 20.10、Compose v2
- Zot v2.1.20，**必须用 full 版镜像**（`zot-linux-amd64`），minimal 版不含扩展
- Gitee 只用 WebHooks 与部署公钥，**不使用 Gitee Go**
- 全程 **不勾选 Let's Encrypt**；Dokploy 面板与 Zot UI 都只经 SSH 隧道访问

## 二、执行顺序

两台机器不能各自独立走完，中间有三处必须交叉：

| 阶段 | 在哪操作 | 做什么 |
|---|---|---|
| 1 | A 和 B **同时** | A1~A3、B1~B3：装 Docker、装 WireGuard 并生成密钥 |
| 2 | 人工 | 交换两台机器的 `/etc/wireguard/publickey` 内容 |
| 3 | A 和 B **同时** | A4~A7、B4~B7：写 wg0.conf、起隧道、配 hosts 与 daemon.json |
| 4 | A | A8~A17：装 Dokploy、部署 Zot、配 Registry 与 Git |
| 5 | B | B8~B9：登录仓库、建目录、建 SSH 部署用户 |
| 6 | **A** | B10：在 Dokploy 里 Setup Server，远程给机器 B 装 Docker 与 Traefik |
| 7 | B | B11：DNS 解析与放行 80 |

第 2 步不做，第 3 步的 wg0.conf 填不出对端公钥。第 6 步是在机器 A 的面板里操作，但效果作用在机器 B。

## 三、机器 A

**A1** 挂 4 GB swap 并持久化

```bash
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**A2** 装 Docker 与 Compose v2

```bash
curl -fsSL https://get.docker.com | sh
```

**A3** 装 WireGuard 并生成密钥对

```bash
sudo apt install -y wireguard && umask 077 && wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

**A4** 写 `wg0.conf`，先把两处尖括号替换成实际值

```bash
sudo tee /etc/wireguard/wg0.conf > /dev/null <<'EOF'
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <机器A私钥>

[Peer]
PublicKey = <机器B公钥>
AllowedIPs = 10.8.0.2/32
EOF
```

**A5** 起隧道并放行隧道端口

```bash
sudo systemctl enable --now wg-quick@wg0 && sudo ufw allow from <机器B公网IP> to any port 51820 proto udp
```

**A6** 配主机名

```bash
echo '10.8.0.1 registry.internal' | sudo tee -a /etc/hosts
```

**A7** 配 `daemon.json` 并重启 Docker

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "insecure-registries": ["registry.internal:5000"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
EOF
sudo systemctl restart docker
```

**A8** 装 Dokploy。本机若已有 Docker Swarm **不要**执行，脚本会强制重新初始化

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

**A9** 从本地开 SSH 隧道，浏览器打开 `http://localhost:3000` 建管理员账号并开 2FA

```bash
ssh -L 3000:localhost:3000 user@<机器A公网IP>
```

**A10** 面板 Settings → Server → Domain 绑定 `ci.example.com`，不勾 Let's Encrypt

**A11** 关闭 3000 的公网访问

```bash
sudo ufw deny 3000/tcp && sudo ufw allow 80/tcp
```

**A12** 部署 Zot，建 `admin` / `ci` / `deploy` 三个账号，步骤见 [registry/README.md](registry/README.md)

**A13** 登录仓库（推送用）

```bash
docker login registry.internal:5000 -u ci
```

**A14** 面板 Settings → Registry → Add，URL 填 `registry.internal:5000`，用户名 `ci`，密码填 A12 设的口令

**A15** 面板 Settings → Git → Custom Git，生成 SSH 密钥，把公钥加到 Gitee 各仓库的「管理 → 部署公钥」

**A16** 建备份目录。不先建好，下一步的 cron 重定向会静默失败

```bash
sudo mkdir -p /srv/backup/dokploy /srv/backup/registry
```

**A17** 装 crontab。整段用单引号包住，`$(...)` 与 `\%` 才会原样写进去

```bash
(crontab -l 2>/dev/null; echo '0 4 * * 0 docker builder prune -f --filter until=168h && docker image prune -a -f --filter until=336h'; echo '0 2 * * * docker exec $(docker ps -qf name=dokploy-postgres) pg_dumpall -U dokploy | gzip > /srv/backup/dokploy/dokploy-$(date +\%F).sql.gz') | crontab -
```

## 四、机器 B

**B1** 挂 4 GB swap 并持久化

```bash
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**B2** 装 Docker 与 Compose v2

```bash
curl -fsSL https://get.docker.com | sh
```

**B3** 装 WireGuard 并生成密钥对

```bash
sudo apt install -y wireguard && umask 077 && wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

**B4** 写 `wg0.conf`，先把三处尖括号替换成实际值

```bash
sudo tee /etc/wireguard/wg0.conf > /dev/null <<'EOF'
[Interface]
Address = 10.8.0.2/24
PrivateKey = <机器B私钥>

[Peer]
PublicKey = <机器A公钥>
Endpoint = <机器A公网IP>:51820
AllowedIPs = 10.8.0.1/32
PersistentKeepalive = 25
EOF
```

**B5** 起隧道并验证连通

```bash
sudo systemctl enable --now wg-quick@wg0 && ping -c 3 10.8.0.1
```

**B6** 配主机名

```bash
echo '10.8.0.1 registry.internal' | sudo tee -a /etc/hosts
```

**B7** 配 `daemon.json` 并重启 Docker

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "insecure-registries": ["registry.internal:5000"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
EOF
sudo systemctl restart docker
```

**B8** 登录仓库（拉取用，经隧道）

```bash
docker login registry.internal:5000 -u deploy
```

**B9** 建业务数据与备份目录

```bash
sudo mkdir -p /srv/appdata /srv/backup
```

**B10** 建 SSH 部署用户，放好机器 A 的公钥，确认 sudo 免密可用。然后**回到机器 A 的面板** Settings → Servers → Create Server，填机器 B 公网 IP、SSH 端口、用户名与私钥，保存后点 **Setup Server**

**B11** 各应用域名 DNS 解析到机器 B 公网 IP，放行 80

```bash
sudo ufw allow 80/tcp
```

## 五、新项目接入

**应用仓库**：把 `template/` 内容复制到仓库根，按语言改 `docker/Dockerfile`，推到 Gitee，在「管理 → 部署公钥」勾选 Dokploy 的公钥。

**单容器应用**用 Application 类型：

1. Create Project，在 Shared Environment Variables 填公共变量，对照 [deploy/env/common.env](deploy/env/common.env)
2. Create Service → Application，Server 选**机器 B**，Provider 选 Custom Git，Build Type 选 Dockerfile 指向 `docker/Dockerfile`
3. Environment 填服务变量，公共项用 `${{project.VAR}}` 引用
4. Domains 加域名与容器端口，不勾 Let's Encrypt
5. General 打开 Auto Deploy，复制 Webhook URL
6. Gitee 仓库 → 管理 → WebHooks → 添加，粘贴上一步的 URL，事件选 Push

**多容器或第三方栈**用 Compose 类型：

1. 在部署仓库加 `projects/<project>/compose.yaml`，参照 `example`，网络加 `dokploy-network` external
2. 自研应用的卷写成 `${APPDATA_ROOT}/<app>/<子目录>`；第三方栈保持命名卷，原因见 [layout.md](layout.md) 第四节
3. Create Service → Compose，Provider 选 Git 指向部署仓库与该文件路径，Server 选机器 B
4. Environment 填变量，Dokploy 会写成 `.env` 放在 compose 同目录
5. Domains 选要暴露的 service 名与端口

首次点 Deploy，之后 push 即自动触发。

## 六、日常操作

| 操作 | 位置 |
|---|---|
| 部署 / 回滚 / 看日志 / 改变量 | Dokploy 服务页 |
| 改编排 | 提交到部署仓库，重新部署时自动拉取 |
| 开 Dokploy 面板 | `ssh -L 3000:localhost:3000 user@<机器A公网IP>` |
| 开 Zot UI | `ssh -L 5000:10.8.0.1:5000 user@<机器A公网IP>` |
| 查隧道 | `sudo wg show` |
| 重连隧道 | `sudo systemctl restart wg-quick@wg0` |

回滚深度由保留策略决定，默认每个镜像保留最近 10 次推送，见 [registry/README.md](registry/README.md) 第五节。

机器 B 的业务数据备份随应用而定，Plane 的见 [apps/plane.md](apps/plane.md) 第三节。

**已知问题**：Dokploy 用私有 registry 时回滚不执行 `docker login`（[issue #3861](https://github.com/Dokploy/dokploy/issues/3861)）。机器 B 重装或凭据失效后，先手工 `docker login` 再回滚。
