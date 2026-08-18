# Harbor 操作手册（隧道内 HTTP）

## 一、环境要求

| 项 | 要求 |
|---|---|
| 版本 | Harbor v2.15.2（离线包 730 MB） |
| 机器 | 4 核 / 8 GB / 120 GB，另挂 4 GB swap |
| 软件 | Docker Engine > 20.10、Docker Compose > 2.3 |
| **前置** | **两台机器间的 WireGuard 隧道已建立**，见 [../README.md](../README.md) 第四、五节 |
| 主机名 | `harbor.internal` → 机器 A 隧道地址 `10.8.0.1` |
| 端口 | 8081 只允许 `wg0` 进入，公网拒绝 |
| 客户端 | 两台机器 daemon 都要配 `insecure-registries` |

两台机器只能走公网，Harbor 因此跑在隧道内：流量由 WireGuard 加密，HTTP 明文只存在于隧道内部，`insecure-registries` 的豁免也仅限隧道地址。**不要跳过隧道直接把 8081 开到公网** —— `docker login` 凭据是 base64 明文，截获即可推镜像投毒。

磁盘分配：系统与 Harbor 本体 15 GB、镜像存储 `/srv/harbor` 70 GB、Dokploy 数据与构建缓存 30 GB。

内存占用：Harbor 常驻约 2 GB，Trivy 扫描峰值 +1 GB，Dokploy 控制平面 0.3~1 GB，`docker build` 峰值 +2~4 GB。GC 与 Trivy 定时任务安排在凌晨 3 点，与构建时段错开。

镜像由 Dokploy 在本机构建后推入 Harbor，机器 B 经隧道拉取。Harbor 凭据在两台机器各 `docker login` 一次，Dokploy 里另配一份 Registry 用于构建后推送。

## 二、安装步骤

### 步骤 1：配主机名指向隧道地址（两台机器都做）

```bash
echo '10.8.0.1 harbor.internal' | sudo tee -a /etc/hosts
```

### 步骤 2：配 insecure-registries（两台机器都做）

写 `/etc/docker/daemon.json`：

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

### 步骤 3：下载离线包

```bash
curl -fsSLO https://github.com/goharbor/harbor/releases/download/v2.15.2/harbor-offline-installer-v2.15.2.tgz
```

### 步骤 4：解压

```bash
sudo tar xzf harbor-offline-installer-v2.15.2.tgz -C /opt
```

### 步骤 5：生成配置

```bash
cd /opt/harbor && sudo cp harbor.yml.tmpl harbor.yml
```

按 [harbor.yml.example](harbor.yml.example) 修改，必改四处：

- `hostname: harbor.internal`
- `http.port: 8081`
- `harbor_admin_password`、`database.password` 换成强密码
- `https` 段与 `external_url` 保持注释

`_version` 保持安装包自带值不动。

### 步骤 6：安装启动

```bash
sudo ./install.sh --with-trivy
```

首次约 3~5 分钟，脚本自动执行 `prepare`、加载镜像、`docker compose up -d`。

### 步骤 7：防火墙

8081 只允许隧道网卡进入，公网一律拒绝：

```bash
sudo ufw allow in on wg0 to any port 8081 proto tcp && sudo ufw deny 8081/tcp
```

从任意外部机器验证公网不可达，应超时或被拒绝：

```bash
curl -m 5 http://<机器A公网IP>:8081/
```

### 步骤 8：验证

```bash
docker login harbor.internal:8081
```

管理员访问 Web UI 走 SSH 隧道，浏览器开 `http://localhost:8081`：

```bash
ssh -L 8081:harbor.internal:8081 user@<机器A公网IP>
```

## 三、初始化步骤

1. 界面右上角改 admin 密码（harbor.yml 的初始密码只在首次生效）
2. 管理 → 配置管理 → 认证 → 取消「允许自注册」
3. 项目 → 新建项目 `apps`，访问级别选**私有**，存储配额 80 GB
4. 进入 `apps` → 机器人账号 → 新建两个，密码只显示一次，当场存好：

| 名称 | 权限 | 登录位置 |
|---|---|---|
| `ci` | pull + push | 机器 A |
| `deploy` | pull | 机器 B |

5. 两台机器各登录一次

```bash
docker login harbor.internal:8081 -u 'robot$apps+ci'
```

```bash
docker login harbor.internal:8081 -u 'robot$apps+deploy'
```

## 四、保留策略与 GC（必做）

两步都要配，只配第一步不会释放磁盘。

**第一步 · Tag 保留策略** — 项目 `apps` → 策略 → Tag 保留 → 新建规则：

- 规则：保留最近推送的 10 个 artifact
- 应用范围：`**`
- 定时：每天 `02:00`

**第二步 · 垃圾回收** — 管理 → 清理 → 垃圾回收：

- 勾选「删除未被任何 manifest 引用的层」
- 定时：每周日 `03:00`（GC 期间仓库只读，不要与构建时间重叠）

执行结果在管理 → 清理 → 历史里查看。

## 五、可选：公共镜像代理缓存

1. 管理 → 仓库服务 → 新建，类型选 `Docker Hub`
2. 新建项目 `dockerhub`，勾选「代理缓存」，选上一步的 endpoint
3. 应用侧镜像地址改写：`nginx:1.27-alpine` → `harbor.internal:8081/dockerhub/library/nginx:1.27-alpine`

代理缓存项目同样占磁盘，也要配保留策略。

## 六、备份

数据库每日备份（不可再生，优先级最高）：

```bash
docker exec -t harbor-db pg_dumpall -U postgres | gzip > /srv/backup/harbor-db-$(date +%F).sql.gz
```

镜像层增量同步：

```bash
rsync -a --delete /srv/harbor/registry/ /srv/backup/harbor-registry/
```

配置文件 `/opt/harbor/harbor.yml` 含密码，单独加密留存。

完整冷备份（季度一次）：

```bash
cd /opt/harbor && sudo docker compose down && sudo tar -czf /srv/backup/harbor-full-$(date +%F).tar.gz -C /srv harbor && sudo docker compose up -d
```

## 七、运维命令

均在 `/opt/harbor` 下执行。

| 操作 | 命令 |
|---|---|
| 查看状态 | `sudo docker compose ps` |
| 停止 | `sudo docker compose down` |
| 启动 | `sudo docker compose up -d` |
| 查看日志 | `sudo docker compose logs -f core` |
| 改完 harbor.yml 后生效 | `sudo docker compose down && sudo ./prepare && sudo docker compose up -d` |
| Harbor 磁盘占用 | `sudo du -sh /srv/harbor/*` |
| 构建缓存占用 | `docker system df` |

改配置必须重跑 `./prepare`，直接 restart 不生效。

升级：备份数据库与 `/opt/harbor` → `docker compose down` → 解压新版包 → 用官方 `harbor-migrator` 迁移 harbor.yml → `./install.sh --with-trivy`。跨多个大版本时逐版升级。

## 八、替代方案：不建隧道，改用公网 HTTPS

只有在无法建立隧道时才用这条路，运维成本更高，且 Harbor 会暴露在公网扫描下。

机器 A 上 80/443 已被 Dokploy 的 Traefik 占用，因此**不要给 Harbor 自己配证书**，而是让 Traefik 反代它：

1. Harbor 保持现状（`http.port: 8081`，`https` 段仍注释），但 `external_url` 改成 `https://harbor.example.com`，改完重跑 `./prepare` 并重启
2. `harbor.example.com` 解析到机器 A 公网 IP
3. 在 Dokploy 里为 Harbor 建一个 Compose 服务或用 Traefik 的 file provider，把该域名反代到宿主机 `8081`，勾选 Let's Encrypt。反代需放开请求体限制并延长超时，否则推大镜像会失败
4. 两台机器的 `daemon.json` **删掉** `insecure-registries`
5. Dokploy 的项目共享变量里 `REGISTRY`、`REGISTRY_CACHE` 换成 `harbor.example.com`（去掉 `:8081`），两台机器重新 `docker login harbor.example.com`
6. 确认所有 Harbor 项目为私有、已关闭自注册，并及时跟进 Harbor 安全更新
