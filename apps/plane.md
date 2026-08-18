# Plane 配置与初始化（社区版 v1.4.1）

Plane 以 Dokploy 的 **Compose 服务**接入，编排文件在部署仓库，变量填在 Dokploy UI。通用的机器准备见 [../README.md](../README.md)，本文只写 Plane 特有部分。

| 项 | 位置 |
|---|---|
| 编排文件 | 部署仓库 `projects/plane/compose.yaml` |
| 变量清单 | [../deploy/env/plane.env](../deploy/env/plane.env)，实际填在 Dokploy 服务的 Environment |
| 运行机器 | 机器 B |
| 数据存放 | docker 命名卷（第三方栈不用 bind mount，原因见 [../layout.md](../layout.md) 第四节） |
| 域名 | 在 Dokploy 的 Domains 里指向 `proxy` 服务的 80 端口 |

## 一、环境要求

| 项目 | 要求 |
|---|---|
| 内存 | 13 个容器空载常驻约 3 GB，与机器 B 其他服务叠加计算 |
| 磁盘 | 镜像约 5 GB，另留 20 GB 给数据库与附件 |
| 镜像来源 | 全部经 Zot 的 `docker/` 前缀拉取，onDemand 模式首次请求时自动回源，**不需要预先建项目或推送** |
| 对外 | 不映射宿主机端口，由 Traefik 按域名路由到 `proxy` 容器 |

## 二、部署步骤

### 步骤 1：在 Dokploy 创建服务

Create Service → **Compose**，Provider 选 Git 指向部署仓库，Compose 路径填 `projects/plane/compose.yaml`，Server 选**机器 B**。

### 步骤 2：填环境变量

在服务的 Environment 编辑器里粘贴 `env/plane.env` 的内容后逐项修改。Dokploy 会把它写成 `.env` 放在 compose 同目录，供 `${VAR}` 插值。

必改项：

| 变量 | 说明 |
|---|---|
| `SECRET_KEY`、`LIVE_SERVER_SECRET_KEY` | 各用 `openssl rand -hex 32` 生成 |
| `POSTGRES_PASSWORD`、`RABBITMQ_PASSWORD`、`AWS_SECRET_ACCESS_KEY` | 各用 `openssl rand -hex 16` 生成 |
| `APP_DOMAIN`、`WEB_URL`、`CORS_ALLOWED_ORIGINS` | 换成真实域名，后两者带 `http://` |

保持不动：`SITE_ADDRESS=:80`、`CERT_EMAIL=` 留空、`MINIO_ENDPOINT_SSL=0`（全站 HTTP，不签证书）。公共变量用 `${{project.REGISTRY_CACHE}}` 引用，不要重复定义。

将来切 HTTPS 时，这三处要同步改：`WEB_URL` 与 `CORS_ALLOWED_ORIGINS` 换成 `https://`、`MINIO_ENDPOINT_SSL` 改为 `1`，否则附件与图片会加载失败。

**数据库密码必须在首次部署前定好**，卷初始化后再改不生效。

### 步骤 3：配域名

Domains → Add，Service Name 选 `proxy`，Container Port 填 `80`，域名填 `plane.example.com`，**不勾选 Let's Encrypt**。DNS 需已解析到机器 B 公网 IP。

### 步骤 4：部署

点 Deploy。首次要经代理缓存回源约 5 GB 镜像，`migrator` 迁移完成后 `api` 才启动，整体 3~10 分钟。在 Logs 里观察，`migrator` 退出码 0 属正常。

首次拉取会在短时间内向 Docker Hub 发起 13 个镜像的回源请求，可能触碰匿名拉取的速率限制。若某个镜像卡住，等几分钟重新 Deploy 即可，已缓存的部分不会重拉。

### 步骤 5：创建实例管理员

浏览器打开 `http://plane.example.com/god-mode/`，在 "Let's secure your instance" 页设置管理员邮箱与密码。

### 步骤 6：配置 SMTP

God Mode → Email 填写 SMTP 并发测试邮件。不配则邀请成员、找回密码、Magic Link 登录均不可用。

### 步骤 7：创建工作区

访问 `http://plane.example.com/`，登录后创建 Workspace 和 Project，邀请成员。

## 三、运维

日常操作走 Dokploy UI：Deploy 重新部署、Logs 看日志、Environment 改变量后重新部署。升级 Plane 只需把 Environment 里的 `APP_RELEASE` 改成新版本号再 Deploy。

数据备份在机器 B 上执行。Plane 的数据在 docker 命名卷里，Dokploy 只备份自己的配置，不碰业务数据：

```bash
docker exec -t $(docker ps -qf name=plane-plane-db) pg_dump -U plane plane | gzip > /srv/backup/plane-$(date +%F).sql.gz
```

```bash
docker run --rm -v plane_uploads:/data -v /srv/backup:/backup alpine tar czf /backup/plane-uploads-$(date +%F).tar.gz -C /data .
```

容器名与卷名的前缀由 Dokploy 生成，用 `docker ps` 与 `docker volume ls` 确认实际名称后再执行。

这两项都属于不可再生数据，必须每日备份，分级见 [../layout.md](../layout.md) 第一节。
