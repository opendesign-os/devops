# devops：Dokploy + Zot 双机部署

**机器 A** 是控制平面（Dokploy 构建、Zot 镜像仓库、Traefik），并跑 Plane；**机器 B** 跑自研应用。服务落在哪台由 Dokploy 服务的 `Server` 字段决定。全站 HTTP，两机之间走 WireGuard 隧道，Dokploy 面板与 Zot UI 只经 SSH 隧道访问。

## 架构

```mermaid
flowchart LR
  dev["开发者本机"]
  github["GitHub<br/>本仓库 devops<br/>手册 · Zot 配置 · 骨架"]
  gitee["Gitee<br/>应用仓库 · 部署仓库"]
  user["用户浏览器"]

  subgraph A["机器 A · 隧道 10.8.0.1"]
    dokploy["Dokploy :3000<br/>拉代码 → 构建镜像"]
    zot["Zot :5000<br/>apps/ 自研镜像<br/>docker/ 代理缓存"]
    tfa["Traefik :80"]
    plane["Plane<br/>13 容器 · 命名卷"]
  end

  subgraph B["机器 B · 隧道 10.8.0.2"]
    tfb["Traefik :80"]
    app["自研应用<br/>/srv/appdata"]
  end

  dev -->|push| github
  dev -->|push| gitee
  github -->|"克隆到 /srv/devops"| zot
  gitee -->|"webhook :80"| tfa
  tfa --> dokploy
  dev -.->|"SSH 隧道 3000 / 5000"| dokploy
  dokploy -->|推镜像| zot
  dokploy -.->|"SSH 下发 compose 与 .env"| app
  plane -->|拉镜像| zot
  app -.->|"拉镜像 · WireGuard"| zot
  tfb --> app
  user -->|":4141"| plane
  user -->|各应用域名| tfb
```

代码托管分两处：**本仓库在 GitHub**，只放手册、Zot 配置与骨架，克隆到机器 A；**应用仓库与部署仓库在 Gitee**，Dokploy 从 Gitee 检出代码与编排，webhook 也由 Gitee 发。

镜像只有一个产地：Dokploy 在机器 A 构建后推 Zot，两台机器都从 Zot 拉，公共镜像也经它的 `docker/` 前缀回源缓存。

## 文档

| 文档 | 内容 |
|---|---|
| [docs/deploy.md](docs/deploy.md) | 全部部署操作步骤，从零到跑起来，唯一来源 |
| [docs/layout.md](docs/layout.md) | 代码、部署文件、变量、镜像、CI/CD 五类资产的存放规范 |
| [docs/registry.md](docs/registry.md) | Zot 的配置项、授权、保留策略、回源上游与运维，不含安装步骤 |
| [docs/plane.md](docs/plane.md) | 在机器 A 用 Dokploy 部署 Plane：变量必改项、初始化、备份与升级 |
| [docs/proxy.md](docs/proxy.md) | 借本机 Clash 给服务器临时开代理，及其删除 |

新装照 [docs/deploy.md](docs/deploy.md) 从上到下做一遍；日常运维查它的第八节。

## 目录

本仓库要克隆到机器 A 的 `/srv/devops`：Zot 从 `registry/` 就地启动，手册也随之在机器上。机器 B 不需要检出，它的编排与 `.env` 由 Dokploy 经 SSH 下发。克隆是 [docs/deploy.md](docs/deploy.md) 第二节第 1 步，也是全流程的第一步。

| 目录 | 服务器上怎么用 | 说明 |
|---|---|---|
| `docs/` | 机器 A 上直接读 | 全部文档，`less /srv/devops/docs/deploy.md` |
| `registry/` | 机器 A 就地启动 Zot | `config.json` 与 `compose.yaml`，容器就地读 |
| `deploy/` | 只作对照，不参与部署 | 部署仓库骨架，真正参与部署的那份由 Dokploy 从 Gitee 检出 |
| `template/` | 只作对照 | 应用仓库骨架，在本机复制到每个新项目 |
