# Plane 配置与初始化（社区版 v1.4.1）

Plane 以 Dokploy 的 **Compose 服务**接入，编排文件在部署仓库，变量填在 Dokploy UI。建服务、配域名、Deploy 的通用流程见 [deploy.md](deploy.md) 第七节，本文只写 Plane 特有的部分。

本文引用的文件路径以本仓库已克隆在机器 A 的 `/srv/devops` 为前提，克隆步骤见 [deploy.md](deploy.md) 第五节第 1 步。

| 项 | 位置 |
|---|---|
| 编排文件 | Gitee 部署仓库 `plane` 根目录的 `compose.yaml`，骨架是本仓库 [../deploy/projects/plane/compose.yaml](../deploy/projects/plane/compose.yaml)（机器 A 上 `/srv/devops/deploy/projects/plane/compose.yaml`） |
| 变量清单 | [../deploy/env/plane.env](../deploy/env/plane.env)（服务器上 `/srv/devops/deploy/env/plane.env`），实际填在 Dokploy 服务的 Environment |
| 运行机器 | 机器 A，创建 Compose 服务时 Server 选机器 A |
| 数据存放 | 机器 A 的 docker 命名卷（第三方栈不用 bind mount，原因见 [layout.md](layout.md) 第四节） |
| 访问地址 | `http://<机器A公网IP>:4141`，compose 里 `proxy` 映射 `4141:80`，不经 Traefik |

## 一、环境要求

| 项目 | 要求 |
|---|---|
| 内存 | 13 个容器空载常驻约 3 GB，与机器 A 的 Dokploy 构建、Zot 叠加算；8 GB 机器靠准备阶段挂的 4 GB swap 撑构建高峰 |
| 磁盘 | 镜像约 5 GB，另留 20 GB 给数据库与附件；机器 A 的 120 GB 还要给 Zot 预留 60 GB，先算余量 |
| 镜像来源 | 全部经 Zot 的 `docker/` 前缀拉取，onDemand 模式首次请求时自动回源，**不需要预先建项目或推送** |
| 对外 | `proxy` 映射宿主机 4141 到容器 80，安全组要放行 4141 |

## 二、变量配置

在服务的 Environment 编辑器里粘贴 `deploy/env/plane.env` 的内容后逐项修改，原文在服务器上直接取：`cat /srv/devops/deploy/env/plane.env`。Dokploy 会把它写成 `.env` 放在 compose 同目录，供 `${VAR}` 插值。

必改项：

| 变量 | 说明 |
|---|---|
| `SECRET_KEY`、`LIVE_SERVER_SECRET_KEY` | 各用 `openssl rand -hex 32` 生成 |
| `POSTGRES_PASSWORD`、`RABBITMQ_PASSWORD`、`AWS_SECRET_ACCESS_KEY` | 各用 `openssl rand -hex 16` 生成 |
| `APP_DOMAIN`、`WEB_URL`、`CORS_ALLOWED_ORIGINS` | 换成 `<机器A公网IP>:4141`，后两者带 `http://` |

保持不动：`SITE_ADDRESS=:80`（容器内 Caddy 的监听端口，与宿主机的 4141 无关）、`CERT_EMAIL=` 留空、`MINIO_ENDPOINT_SSL=0`（全站 HTTP，不签证书）。公共变量用 `${{project.REGISTRY_CACHE}}` 引用，不要重复定义。

用外部 S3/OSS 的话，删掉 compose 里的 `plane-minio` 服务，把 `AWS_*` 换成对象存储的地址与密钥。

将来切 HTTPS 时，这三处要同步改：`WEB_URL` 与 `CORS_ALLOWED_ORIGINS` 换成 `https://`、`MINIO_ENDPOINT_SSL` 改为 `1`，否则附件与图片会加载失败。

**数据库密码必须在首次部署前定好**，卷初始化后再改不生效。

## 三、初始化

按 [deploy.md](deploy.md) 第七节建好服务并填完变量后，点 Deploy。首次要经代理缓存回源约 5 GB 镜像，`migrator` 迁移完成后 `api` 才启动，整体 3~10 分钟。在 Logs 里观察，`migrator` 退出码 0 属正常。

首次拉取会在短时间内向 Docker Hub 发起 13 个镜像的回源请求，可能触碰匿名拉取的速率限制。若某个镜像卡住，等几分钟重新 Deploy 即可，已缓存的部分不会重拉。

### 1 · 创建实例管理员

浏览器打开 `http://<机器A公网IP>:4141/god-mode/`，在 "Let's secure your instance" 页设置管理员邮箱与密码。

### 2 · 配置 SMTP

God Mode → Email 填写 SMTP 并发测试邮件。不配则邀请成员、找回密码、Magic Link 登录均不可用。

### 3 · 创建工作区

访问 `http://<机器A公网IP>:4141/`，登录后创建 Workspace 和 Project，邀请成员。

## 四、运维与备份

日常操作走 Dokploy UI：Deploy 重新部署、Logs 看日志、Environment 改变量后重新部署。升级 Plane 只需把 Environment 里的 `APP_RELEASE` 改成新版本号再 Deploy。

数据备份在机器 A 上执行。Plane 的数据在 docker 命名卷里，Dokploy 只备份自己的配置，不碰业务数据：

```bash
docker exec -t $(docker ps -qf name=plane-plane-db) pg_dump -U plane plane | gzip > /srv/backup/plane-$(date +%F).sql.gz
```

```bash
docker run --rm -v plane_uploads:/data -v /srv/backup:/backup alpine tar czf /backup/plane-uploads-$(date +%F).tar.gz -C /data .
```

容器名与卷名的前缀由 Dokploy 生成，用 `docker ps` 与 `docker volume ls` 确认实际名称后再执行。

这两项都属于不可再生数据，必须每日备份，分级见 [layout.md](layout.md) 第一节。
