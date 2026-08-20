# 部署手册（Dokploy + Zot + Gitee）

从零到跑起来的全部操作步骤，唯一来源。架构总览、文档索引与目录说明见 [../README.md](../README.md)。

机器 A 的 Zot 从本仓库的检出 `/srv/devops` 启动，克隆是第五节第 1 步；机器 B 不需要检出，它的编排与 `.env` 由 Dokploy 经 SSH 下发。

## 一、前提

**系统 Alibaba Cloud Linux 3**（RHEL 系，内核 5.10）。命令按它写：包管理用 `dnf`，入站靠阿里云安全组。换 Ubuntu/Debian 需把 `dnf` 换成 `apt`、`httpd-tools` 换成 `apache2-utils`。

| | 机器 A | 机器 B |
|---|---|---|
| 配置 | 4 核 / 8 GB / 120 GB | 4 核 / 8 GB / 200 GB |
| 隧道地址 | `10.8.0.1`（`registry.internal`） | `10.8.0.2` |
| 域名 | `ci.example.com` + 跑在 A 的应用域名（Plane） | 跑在 B 的应用域名 |

- Gitee 只用 WebHooks 与部署公钥，**不使用 Gitee Go**
- 全程 **不勾选 Let's Encrypt**；Dokploy 面板与 Zot UI 都只经 SSH 隧道访问

### 需要替换的值

命令与配置里凡是 `<尖括号>` 的都要换成你的实际值，开始前先备齐：

| 占位符 | 从哪来 |
|---|---|
| `<机器A公网IP>` | 阿里云控制台，机器 A 的公网 IP |
| `<机器B公网IP>` | 阿里云控制台，机器 B 的公网 IP |
| `<SSH用户>` | 登录服务器的账号，如 `root` |
| `<机器A私钥>` `<机器A公钥>` | 第二节第 3 步在机器 A 上生成 |
| `<机器B私钥>` `<机器B公钥>` | 第二节第 3 步在机器 B 上生成 |
| `ci.example.com` | 换成你给机器 A 准备的真实域名 |
| `plane.example.com` | 换成各应用的真实域名 |

**下面这些是固定值，不要改成公网 IP：**

| 固定值 | 含义 |
|---|---|
| `10.8.0.1` | **机器 A** 的隧道地址，`registry.internal` 指向它 |
| `10.8.0.2` | **机器 B** 的隧道地址 |
| `registry.internal` | 隧道内的主机名，解析到 `10.8.0.1` |
| `51820` / `5000` / `3000` | WireGuard / Zot / Dokploy 的端口 |
| `/srv/devops` | 本仓库在机器 A 上的检出路径 |

### 安全组

入站由**安全组**决定，控制台里没放行的端口，系统内开了也没用。阿里云官方镜像的 firewalld 默认不启用，若你启用了要同步放行 80 与 51820。

| 机器 | 端口 | 协议 | 授权对象 | 描述（填进阿里云「描述」字段） |
|---|---|---|---|---|
| A | 22 | TCP | 限管理 IP | `SSH 管理入口，限办公网 IP` |
| A | 80 | TCP | 全部 | `Gitee webhook 与跑在 A 的应用域名入口（Traefik）` |
| A | 51820 | **UDP** | `<机器B公网IP>/32` | `WireGuard 隧道入口，仅机器B接入，用于拉取 Zot 私有镜像` |
| B | 22 | TCP | 限管理 IP | `SSH 管理入口，限办公网 IP` |
| B | 80 | TCP | 全部 | `应用域名入口（Traefik）` |

表里没有的一律**不放行**：两台的 3000、5000，以及机器 B 的 51820 —— 机器 B 是主动发起隧道的一方，回包走已建立的会话，不需要入方向规则。

添加 51820 那条时两处容易错：协议类型要选**自定义 UDP**（默认是 TCP，选错则握手永远不成功）；授权对象必须带 `/32` 掩码，只填 IP 不被接受。

3000 是 Dokploy 面板、5000 是 Zot，都只经 SSH 隧道访问。Zot 容器还额外只绑定隧道地址 `10.8.0.1`，公网网卡上没有监听。

### 章节顺序

从上到下按序执行，本文是唯一的操作步骤来源。

| 顺序 | 章节 | 在哪操作 |
|---|---|---|
| 1 | [二、准备](#二准备) | 两台机器都做，可并行 |
| 2 | [三、建隧道](#三建隧道) | 先 A 后 B |
| 3 | [四、机器 A：Dokploy](#四机器-adokploy) | 只在 A |
| 4 | [五、机器 A：Zot 镜像仓库](#五机器-azot-镜像仓库) | 只在 A |
| 5 | [六、机器 B：接入](#六机器-b接入) | 只在 B，末尾要回 A 点一次 |
| 6 | [七、新项目接入](#七新项目接入) | Dokploy 面板 + Gitee |

### 命令的读法

每个代码块是**一次完整粘贴的单位**，不要按行拆开。带 `<<'EOF'` 的块从首行一直到单独的 `EOF` 才算一条命令 —— 只粘中间的 JSON 或配置正文，shell 会把 `{` 当成命令组、把 `"key":` 当成命令名，报一串 `command not found`，而文件根本没写成。

命令里的 `<尖括号>` 要先换成上表的实际值再执行。

## 二、准备

**两台机器都要做，内容完全一样，可以并行。**

### 1 · 挂 4 GB swap

```bash
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

报 `Text file busy` 说明已经挂过了，`swapon --show` 确认后跳过。此时 `&&` 已中断，`/etc/fstab` 不会重复追加。

### 2 · 装 Docker

`get.docker.com` 不认 `alinux`，改用 CentOS 源。`$releasever` 那步 sed 不能省 —— alinux 的值是 3，docker-ce 源没有 3 的目录。

```bash
sudo dnf install -y dnf-utils && sudo dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && sudo sed -i 's/$releasever/8/g' /etc/yum.repos.d/docker-ce.repo && sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin && sudo systemctl enable --now docker
```

报 `containerd.io` 与自带 `runc` 冲突时，末尾加 `--allowerasing`。

### 3 · 装 WireGuard 并生成密钥

alinux 官方源里没有任何 wireguard 包，必须挂 EPEL。`epel-release` 是 CentOS 的包名，这里装不了，直接写源文件；版本写死 8，同样因为 `$releasever` 是 3。内核 5.10 已内置 wg 模块，只装 tools。

```bash
sudo tee /etc/yum.repos.d/epel.repo > /dev/null <<'EOF'
[epel]
name=EPEL for Alibaba Cloud Linux 3
baseurl=https://mirrors.aliyun.com/epel/8/Everything/$basearch
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/epel/RPM-GPG-KEY-EPEL-8
EOF
sudo dnf install -y wireguard-tools && umask 077 && wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

末尾打印的是**公钥**；私钥已写入 `/etc/wireguard/privatekey`，它在管道里被 `wg pubkey` 消费掉了所以不显示，属正常。

### 4 · 配主机名

```bash
grep -q registry.internal /etc/hosts || echo '10.8.0.1 registry.internal' | sudo tee -a /etc/hosts
```

`grep -q` 打头是为了可重复执行 —— 直接 `tee -a` 跑第二次会追加出重复行。

验证，输出必须是 `1`：

```bash
grep -c registry.internal /etc/hosts
```

已经重复了的话，先清干净再重跑上面那条：

```bash
sudo sed -i '/registry.internal/d' /etc/hosts
```

### 5 · 配 daemon.json 并重启 Docker

这个文件配三件事：`insecure-registries` 允许对 `registry.internal:5000` 走 HTTP 且不校验证书，豁免仅限这一个地址；`log-driver` 与 `log-opts` 给容器日志加轮转，单文件上限 50 MB、保留 3 个，即每个容器的日志最多占 150 MB —— 不配的话 json-file 默认不轮转，长跑的容器迟早把磁盘写满。

先看文件是不是空的。**有内容就别照下面覆盖** —— 要把已有配置与 `insecure-registries` 合并后再写，直接覆盖会丢掉别的组件（如 Dokploy）写进去的设置：

```bash
cat /etc/docker/daemon.json 2>/dev/null || echo '文件不存在，可直接执行下一条'
```

下面**整段一次粘贴**，从 `sudo tee` 到 `EOF` 是一条命令，只粘中间的 JSON 会报 `command not found`：

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "insecure-registries": ["registry.internal:5000"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
EOF
```

确认写成了再重启：

```bash
cat /etc/docker/daemon.json && sudo systemctl restart docker
```

**若已经装过 Dokploy**（顺序颠倒了），这次重启会把 Dokploy 的容器一并重启，等半分钟面板恢复，数据不受影响。

### 6 · 取出密钥

两台机器都执行，把值记下来。

```bash
sudo cat /etc/wireguard/privatekey; sudo cat /etc/wireguard/publickey
```

| 密钥 | 填到哪 |
|---|---|
| 私钥 | **本机** wg0.conf 的 `[Interface] PrivateKey`，不出本机 |
| 公钥 | **对端** wg0.conf 的 `[Peer] PublicKey` |

**密钥一旦重新生成，对端的配置也必须同步更新** —— 重复执行第 3 步会覆盖旧密钥，对端还填着旧公钥的话，包会被静默丢弃，表现为 ping 不通但没有任何报错。

验证本机这对是否配套，下面这条的输出必须与上面的公钥一致：

```bash
sudo cat /etc/wireguard/privatekey | wg pubkey
```

> **两台机器都做完这 6 步再往下。** 建隧道要用对方的公钥，缺一台就没法填。

## 三、建隧道

### 机器 A

写 `wg0.conf`，`PublicKey` 填**机器 B** 的公钥。

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
sudo systemctl enable --now wg-quick@wg0
```

机器 A 必须先起，机器 B 才有对端可连。安全组放行 UDP 51820 也要提前做好，见第一节。

### 机器 B

写 `wg0.conf`，`PublicKey` 填**机器 A** 的公钥，`Endpoint` 填 `<机器A公网IP>`。

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
sudo systemctl enable --now wg-quick@wg0 && ping -c 3 10.8.0.1
```

> **ping 不通就别往下走**，后面每一步都依赖这条隧道。

两台机器都执行 `sudo wg show`，按下表对号入座：

| 现象 | 原因 |
|---|---|
| 一台的 `peer:` 值 ≠ 对端的 `public key:` 值 | 公钥填错或对端重新生成过密钥 |
| 有 `sent` 但 `0 B received` | 包发出去没回应 —— 安全组没放行 UDP 51820，或对端公钥不匹配被丢弃 |
| 完全没有 `transfer` 行 | 从未收到过任何包 |
| `latest handshake` 空白 | 握手从未成功 |

安全组的协议必须选 **UDP**，选 TCP 无效。

## 四、机器 A：Dokploy

以下全部在机器 A 操作。

### 1 · 装 Dokploy

要求 80、443、3000 三个端口空闲；本机若已有 Docker Swarm **不要**执行，脚本会强制重新初始化。

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

脚本只在检测不到 Docker 时才去调 `get.docker.com`，准备阶段已经装好，所以不会踩 `alinux` 不被识别的坑 —— 顺序反过来就会失败。

### 2 · 建管理员账号

从本地开 SSH 隧道，浏览器打开 `http://localhost:3000` 创建管理员账号并开 2FA。

```bash
ssh -L 3000:localhost:3000 <SSH用户>@<机器A公网IP>
```

### 3 · 绑定域名

Settings → **Web Server** → Server Domain：

1. Domain 填 `ci.example.com`
2. Let's Encrypt Email 留空
3. HTTPS 开关保持关闭
4. Save

### 4 · 面板配 Git

Settings → Git → Custom Git，生成 SSH 密钥，把公钥加到 Gitee 各仓库的「管理 → 部署公钥」。

### 5 · 建备份目录

不先建好，下一步的 cron 重定向会静默失败。

```bash
sudo mkdir -p /srv/backup/dokploy /srv/backup/registry
```

### 6 · 装 crontab

整段用单引号包住，`$(...)` 与 `\%` 才会原样写进去。

```bash
(crontab -l 2>/dev/null; echo '0 4 * * 0 docker builder prune -f --filter until=168h && docker image prune -a -f --filter until=336h'; echo '0 2 * * * docker exec $(docker ps -qf name=dokploy-postgres) pg_dumpall -U dokploy | gzip > /srv/backup/dokploy/dokploy-$(date +\%F).sql.gz') | crontab -
```

重复执行会追加重复条目。装完核对一遍，应当只有这两行：

```bash
crontab -l
```

## 五、机器 A：Zot 镜像仓库

仍在机器 A 操作。主机名与 `insecure-registries` 已在第二节配好，这里不重复。

容器**只绑定 `10.8.0.1:5000`**，公网网卡上不监听；代价是 wg0 必须先于容器起来，否则绑不到地址会反复重启。

配置与状态分两处：检出里的 [compose.yaml](../registry/compose.yaml) 与 [config.json](../registry/config.json) 由容器就地读；`/srv/registry/` 只放运行时状态，即 `htpasswd` 与镜像层 `data/`。目录划分见 [layout.md](layout.md) 第三节，配置项含义、权限矩阵、保留策略调参与运维命令见 [registry.md](registry.md)。

### 1 · 克隆本仓库

Zot 从检出目录启动，所以先把仓库克隆到 `/srv/devops`：

```bash
sudo dnf install -y git && sudo git clone https://github.com/opendesign-os/devops.git /srv/devops
```

已经克隆过的话，改成更新：

```bash
sudo git -C /srv/devops pull
```

`sudo` 与 `-C` 都不能省：检出属 root，普通用户执行 `git pull` 会报 `detected dubious ownership`；不带 `-C` 就得先 `cd` 进去，否则报 `not a git repository`。

拉不动 GitHub（卡在 `Cloning into` 或报 `Failed to connect`）时两条路：按 [proxy.md](proxy.md) 借本机 Clash 给服务器临时开代理后重试，注意 `sudo git` 读的是 root 的配置，代理要给 root 也配一次；或在 Gitee 建一个镜像仓库，把上面的地址换成镜像地址。

手册也随之到了机器上，`less /srv/devops/docs/deploy.md` 直接读。

### 2 · 建目录

```bash
sudo mkdir -p /srv/registry/data && sudo chmod 755 /srv/registry
```

容器以 root 运行，数据目录不需要额外 chown。

### 3 · 生成三个账号

Zot 只认 **bcrypt**，`-B` 不能省。

```bash
sudo dnf install -y httpd-tools
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

三个账号都生成了的话，文件正好三行：

```bash
sudo wc -l < /srv/registry/htpasswd
```

`ci` 在机器 A 推送、`deploy` 在机器 B 拉取、`admin` 只用于登录 UI，权限矩阵见 [registry.md](registry.md) 第二节。

**备选**（宿主机装不了 `httpd-tools` 时才用）：借容器里的 `htpasswd` 生成。`-b` 会把密码明文留在 shell 历史里，事后要 `history -c`；且此时 Zot 未启动，拉这个镜像只能直连 Docker Hub：

```bash
docker run --rm httpd:2.4-alpine htpasswd -bBn ci '<密码>' | sudo tee -a /srv/registry/htpasswd
```

### 4 · 启动

```bash
cd /srv/devops/registry && sudo docker compose up -d
```

`config.json` 的默认值可直接用，三个账号的授权、两条保留策略与 Docker Hub 代理缓存都已写好。

首次拉取 zot 镜像约 100 MB，秒级启动。起不来先 `sudo wg show` 确认隧道。

### 5 · 验证

未认证访问应返回 `401`，说明服务活着且鉴权生效：

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://registry.internal:5000/v2/
```

从任意外部机器验证公网不可达，应超时或被拒绝：

```bash
curl -m 5 http://<机器A公网IP>:5000/v2/
```

### 6 · 登录仓库（推送用）

```bash
docker login registry.internal:5000 -u ci
```

### 7 · 面板配 Registry

左侧导航 **Registry** → Add，按下表填：

| 字段 | 填什么 |
|---|---|
| Registry Name | 面板内的显示名，随意，建议 `zot-internal` |
| Username | `ci` |
| Password | 第 3 步给 `ci` 设的口令 |
| Image Prefix | `apps` |
| Registry URL | `registry.internal:5000`，只要主机名与端口，不带 `http://` |
| Server (Optional) | 留空 |

`Image Prefix` 是唯一容易错的一项。Dokploy 按 `<URL>/<prefix>/<应用名>:<tag>` 打 tag，而 `ci` 只在 `apps/**` 上有写权限、保留策略也只认这个前缀，填空或填别的名字都会以 401 失败。它就是共享变量里的 `REGISTRY_NS`，权限矩阵见 [registry.md](registry.md) 第二节。

`Server` 留空表示在 Dokploy 所在机器（机器 A）认证，构建与推送都在那儿。机器 B 的拉取凭据是另一回事，用 `deploy` 账号手工登录，见第六节第 1 步。

填完先点 **Test Registry**，通过再 Create。报连不上先 `sudo wg show` 确认隧道。

## 六、机器 B：接入

> **先确认第五节已完成** —— Zot 没起来，下面第 1 步会连不上。

### 1 · 登录仓库（拉取用）

走隧道，不经公网。

```bash
docker login registry.internal:5000 -u deploy
```

### 2 · 验证代理缓存

拉一个公共镜像，走 `docker/` 前缀。第一次会回源 Docker Hub（慢），第二次直接命中本地。

```bash
docker pull registry.internal:5000/docker/library/alpine:3.20
```

### 3 · 建业务数据与备份目录

```bash
sudo mkdir -p /srv/appdata /srv/backup
```

### 4 · 放 SSH 部署密钥

面板（机器 A）：Settings → **SSH Keys** → Create SSH Key，复制 **Public Key**。

机器 B，用 root：

```bash
sudo install -d -m 700 /root/.ssh
```

```bash
KEY='<粘贴 Public Key>'; sudo grep -qF "$KEY" /root/.ssh/authorized_keys 2>/dev/null || echo "$KEY" | sudo tee -a /root/.ssh/authorized_keys > /dev/null; sudo chmod 600 /root/.ssh/authorized_keys
```

```bash
sudo sshd -T | grep -E 'permitrootlogin|pubkeyauthentication'
```

期望 `pubkeyauthentication yes`、`permitrootlogin` 为 `yes` 或 `prohibit-password`；不满足改 `/etc/ssh/sshd_config` 后 `sudo systemctl restart sshd`。

改用非 root 用户（如 `deployer`）换成这三条：

```bash
sudo useradd -m -s /bin/bash deployer && sudo install -d -m 700 -o deployer -g deployer /home/deployer/.ssh
```

```bash
echo '<粘贴 Public Key>' | sudo tee /home/deployer/.ssh/authorized_keys > /dev/null && sudo chown deployer:deployer /home/deployer/.ssh/authorized_keys && sudo chmod 600 /home/deployer/.ssh/authorized_keys
```

```bash
echo 'deployer ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/deployer > /dev/null && sudo chmod 440 /etc/sudoers.d/deployer && sudo visudo -c
```

`visudo -c` 输出 `parsed OK` 才算成功。

### 5 · 回机器 A 点 Setup Server

> **这一步在机器 A 的面板里操作，装的是机器 B。**

1. Settings → **Remote Servers** → Create Server：Name `machine-b`、IP Address `<机器B公网IP>`、Port `22`、Username `<SSH用户>`、SSH Key 选上一步那把 → Create
2. 该服务器的操作菜单 → **Enter Terminal**，出现 shell 提示符即连通
3. 操作菜单 → **Setup Server** → 切到 **Deployments** Tab → 点里面的 **Setup Server**，等日志跑完，只需一次

机器 B 上验证：

```bash
docker info --format '{{.Swarm.LocalNodeState}}'
```

```bash
docker network ls --filter name=dokploy-network --format '{{.Name}} {{.Driver}} {{.Scope}}'
```

```bash
docker ps --format '{{.Names}}\t{{.Status}}' | grep -i traefik
```

```bash
ls /etc/dokploy
```

依次期望 `active`、`dokploy-network overlay swarm`、traefik 容器 Up、目录存在。

脚本会跳过已装的 Docker（判断条件 `command -v docker`），swarm 由这一步初始化，机器 B 不要事先自己 `docker swarm init`。

### 6 · DNS 解析

应用域名解析到它实际运行的那台机器：跑在机器 B 的解析到 `<机器B公网IP>`，跑在机器 A 的（Plane）解析到 `<机器A公网IP>`。两台的 80 端口都要在安全组里放行，见第一节。

## 七、新项目接入

每个服务用 **Server** 字段选运行机器，同一项目里的服务可以分散在两台机器上。当前分布：自研应用跑机器 B，Plane 跑机器 A。镜像一律在机器 A 构建后推 Zot，与服务跑在哪台无关。

**应用仓库**：在本机克隆本仓库，把 `template/` 内容复制到应用仓库根，按语言改 `docker/Dockerfile`，推到 Gitee，在「管理 → 部署公钥」勾选 Dokploy 的公钥。服务器上的 `/srv/devops/template/` 只作对照，代码不在服务器上改。

**单容器应用**用 Application 类型：

1. Create Project，在 Shared Environment Variables 填公共变量，对照 [deploy/env/common.env](../deploy/env/common.env)
2. Create Service → Application，Server 选运行机器（自研应用选**机器 B**），Provider 选 Custom Git，Build Type 选 Dockerfile 指向 `docker/Dockerfile`
3. Environment 填服务变量，公共项用 `${{project.VAR}}` 引用
4. Domains 加域名与容器端口，不勾 Let's Encrypt
5. General 打开 Auto Deploy，复制 Webhook URL
6. Gitee 仓库 → 管理 → WebHooks → 添加，粘贴上一步的 URL，事件选 Push

**多容器或第三方栈**用 Compose 类型：

1. 在部署仓库加 `projects/<project>/compose.yaml`，参照本仓库 [deploy/projects/example/compose.yaml](../deploy/projects/example/compose.yaml)（服务器上是 `/srv/devops/deploy/projects/example/compose.yaml`），网络加 `dokploy-network` external
2. 自研应用的卷写成 `${APPDATA_ROOT}/<app>/<子目录>`；第三方栈保持命名卷，原因见 [layout.md](layout.md) 第四节
3. Create Service → Compose，Provider 选 Git 指向部署仓库与该文件路径，Server 选运行机器
4. Environment 填变量，Dokploy 会写成 `.env` 放在 compose 同目录
5. Domains 选要暴露的 service 名与端口

首次点 Deploy，之后 push 即自动触发。

Plane 走 Compose 类型，Server 选**机器 A**，变量必改项与初始化步骤见 [plane.md](plane.md)。

## 八、日常操作

| 操作 | 位置 |
|---|---|
| 部署 / 回滚 / 看日志 / 改变量 | Dokploy 服务页 |
| 改编排 | 提交到部署仓库，重新部署时自动拉取 |
| 更新机器 A 的检出 | `sudo git -C /srv/devops pull` |
| Zot 重启 / 看日志 / 同步配置 / 查磁盘 | 见 [registry.md](registry.md) 第六节 |
| 开 Dokploy 面板 | `ssh -L 3000:localhost:3000 <SSH用户>@<机器A公网IP>` |
| 开 Zot UI | `ssh -L 5000:10.8.0.1:5000 <SSH用户>@<机器A公网IP>`，此处不能写 localhost，Zot 只绑隧道地址 |
| 查隧道 | `sudo wg show` |
| 重连隧道 | `sudo systemctl restart wg-quick@wg0` |

Zot UI 用 `admin` 账号登录，浏览器开 `http://localhost:5000`。

回滚深度由保留策略决定，默认每个镜像保留最近 10 次推送，见 [registry.md](registry.md) 第四节。

业务数据备份在应用实际运行的那台机器上做，Plane 的见 [plane.md](plane.md) 第四节。

**已知问题**：Dokploy 用私有 registry 时回滚不执行 `docker login`（[issue #3861](https://github.com/Dokploy/dokploy/issues/3861)）。机器 B 重装或凭据失效后，先手工 `docker login` 再回滚。

## 九、附录：不建隧道，改用公网 HTTPS

只有在无法建立隧道时才走这条路，仓库会暴露在公网扫描下，与第三、五节的隧道方案**互斥**。

机器 A 上 80/443 已被 Dokploy 的 Traefik 占用，因此**不要让 Zot 自己签证书**，而是让 Traefik 反代它：

1. 改仓库里的 `registry/compose.yaml`，端口改成 `"127.0.0.1:5000:5000"` 并接入 `dokploy-network`，push 后在机器 A `sudo git -C /srv/devops pull` 并重新 `docker compose up -d`
2. `registry.example.com` 解析到机器 A 公网 IP
3. 在 Dokploy 里把该域名反代到 Zot 容器的 5000，勾选 Let's Encrypt。反代需放开请求体限制并延长超时，否则推大镜像会失败
4. 两台机器的 `daemon.json` **删掉** `insecure-registries`
5. 项目共享变量里 `REGISTRY` 改成 `registry.example.com`、`REGISTRY_CACHE` 改成 `registry.example.com/docker`（都去掉 `:5000`），两台机器重新 `docker login`
6. 确认 `anonymousPolicy` 仍为空数组 —— 公网暴露下，匿名可读等于把镜像公开

Zot 也支持在 `http.tls` 里直接配证书，但那样要自己解决续期，不如复用已有的 Traefik。
