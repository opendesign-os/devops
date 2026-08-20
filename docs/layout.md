# 存放规范

覆盖五类资产：**项目代码、部署文件、环境变量、镜像文件、CI/CD 定义**，外加本仓库在机器 A 上的检出、运行时才产生的业务数据与凭据。

每一份都必须能回答三个问题：**权威副本在哪、丢了能不能重建、要不要备份**。本文是这三个问题的唯一答案，其余文档只引用不重复。

## 一、总表

### 归属：每类资产住在哪

| 资产 | 权威副本 | 本仓库骨架 | 服务器上的落点 |
|---|---|---|---|
| 项目代码 | Gitee 应用仓库 | `template/` | 机器 A 的 Dokploy 检出；机器 B 没有 |
| 部署文件 | 各栈自己的 Gitee 部署仓库 | `deploy/projects/` | 机器 A 检出，再由 Dokploy 下发到服务的运行机器 |
| 部署手册与骨架 | 本仓库（GitHub `opendesign-os/devops`） | 全部 | 机器 A 的 `/srv/devops`，Zot 就在 `/srv/devops/registry` 下启动 |
| 环境变量 | 机器 A 的 Dokploy 数据库 | `deploy/env/`（清单，不是真值） | 运行机器上的 `.env`，由 Dokploy 写 |
| 镜像文件 | 机器 A 的 Zot | 无，构建产物不入库 | `/srv/registry/data/` |
| CI/CD 定义 | **本方案不产生此类文件**，见第二节 | 无 | 无 |
| 业务数据 | 服务所在机器的命名卷或 `/srv/appdata` | 无，运行时产生 | 自研应用在机器 B；Plane 在机器 A |
| 仓库凭据 | 机器 A 的 `htpasswd` | 无，现场生成 | `/srv/registry/htpasswd` |

仓库里只有三个骨架目录（`template/`、`deploy/`、`registry/`），其余四类都不进版本控制 —— 要么是构建产物，要么是运行时状态，要么是密钥。

机器 A 的 `/srv/devops` 是本仓库的检出，Zot 就在 `/srv/devops/registry` 下启动、就地读里面的配置，克隆与更新见 [deploy.md](deploy.md) 第二节第 1 步。机器 B 没有检出，它的编排与 `.env` 由 Dokploy 经 SSH 下发。检出里的文件不手工改 —— 改动一律回本仓库 push 再 pull。

服务落在哪台机器由 Dokploy 服务的 **Server** 字段决定，同一项目的服务可以分散在两台上；下面各节按当前分布写：自研应用在机器 B，Plane 在机器 A。

### 存续：丢了能不能重建

| 资产 | 派生副本 | 丢失后果 | 备份频率 |
|---|---|---|---|
| 项目代码 | 机器 A 的检出 | 无，重新拉取 | 由 Gitee 负责 |
| 部署文件 | 机器 A 检出、运行机器上的下发件 | 无，重新拉取 | 由 Gitee 负责 |
| 部署手册与骨架 | 机器 A 的 `/srv/devops` | 无，重新克隆 | 由 Git 远端负责 |
| **环境变量与服务定义** | 运行机器上的 `.env` | **不可重建**，等于丢掉全部部署配置 | 每日 |
| 自研镜像 | 机器 B 本地镜像层 | 可重新构建，但历史版本没了就无法回滚 | 每周 |
| 公共镜像缓存 | 无 | 无，下次拉取自动回源 | 不备份 |
| **业务数据** | 无 | **不可重建** | 每日 |
| 仓库凭据 | 无 | 可重建，但两台机器都要重新登录 | 变更时留存 |

两处加粗的是全套设施里仅有的**不可再生数据**：Dokploy 的库和各应用的业务卷（Plane 在机器 A，自研应用在机器 B）。其余全部可以从 Gitee 与上游重建。备份预算优先给这两项。

## 二、CI/CD 定义放在哪

**本方案不产生 CI/CD 文件。** 这不是遗漏，是设计选择 —— CI 的各项职责被拆给了已有组件：

| CI 的常规职责 | 本方案由谁承担 |
|---|---|
| 触发 | Gitee WebHooks → Dokploy |
| 拉取代码 | Dokploy |
| 跑测试与 lint | **Dockerfile 的构建阶段** |
| 构建镜像 | Dokploy（机器 A） |
| 推送镜像 | Dokploy → Zot |
| 部署 | Dokploy → 服务的运行机器 |
| 回滚 | Dokploy 的 Deployments |

测试写在 `docker/Dockerfile` 的构建阶段里（如 `RUN npm test`）：测试不过 → 构建失败 → 镜像不产生 → 部署不发生。只有一处定义，不会出现「CI 绿了但构建挂了」这种两套环境不一致的问题。

代价要清楚，别以为白拿：

- **反馈慢** —— 要等镜像构建到那一步才知道测试挂了
- **拦不住合并请求** —— webhook 是 push 触发，PR / MR 阶段没有检查
- **做不了轻量检查** —— 想只跑一次 lint 也得走完整构建
- **日志混在一起** —— 测试输出夹在构建日志里

### 将来要引入独立 CI 的话

| 项 | 规定 |
|---|---|
| 文件位置 | **应用仓库**根目录，按所选平台的约定目录放。不放部署仓库 |
| 允许做 | 测试、lint、安全扫描、生成报告 |
| 禁止做 | 构建并推送镜像、直接部署到任一运行机器 |

最后一条是硬边界：**镜像的产地必须唯一**。CI 和 Dokploy 各推各的，`apps/<app>` 下就有了两个来源，回滚时分不清某个 tag 是谁推的、对应哪次提交，保留策略的计数也会被打乱。

要换就整体换：要么全交给 Dokploy（当前选择），要么全交给 CI、并把 Dokploy 降级成「只拉取已有镜像来部署」。不能各推各的。

## 三、机器 A：控制平面

```
/etc/dokploy/                      # Dokploy 自建，不要手工修改
                                   # 内含 ★ 元数据库（服务定义与全部环境变量）、
                                   # 源码检出、构建缓存、Traefik 配置，以及跑在 A 的
                                   # 服务（Plane）的 compose 与 .env。
                                   # 子目录随版本变化，用 sudo ls /etc/dokploy 查看实际结构

/srv/devops/                       # 本仓库的检出，更新用 git pull，不手工改
└── registry/                      # 唯一参与运行的目录：Zot 在此 docker compose up -d
    ├── compose.yaml               # 编排，容器就地读
    └── config.json                # 授权与保留策略，容器就地读，不含明文密码

/srv/registry/                     # Zot 的运行时状态，不进版本控制
├── htpasswd                       # ★ ci / deploy / admin 的 bcrypt 口令
└── data/                          # 镜像层，对应容器内 /var/lib/registry
    ├── apps/
    └── docker/

/srv/backup/
├── dokploy/                       # 每日 pg_dumpall
└── registry/                      # 每周 rsync 镜像层

/var/lib/docker/
├── overlay2/                      # 构建缓存与中间层，可随时清空
└── volumes/                       # ★ 跑在 A 的第三方栈命名卷（Plane）
```

机器 A 上人工维护的只有 `/srv/registry/htpasswd`（现场生成，不入库）；`/srv/devops` 只用 `git pull` 更新；`/etc/dokploy/` 全部由 Dokploy 自己管。

Plane 跑在机器 A，数据在 `/var/lib/docker/volumes/` 的命名卷里，理由见第四节。

## 四、机器 B：运行平面

```
/etc/dokploy/                      # Dokploy 经 SSH 下发的 compose 与 .env，不要手工改
                                   # 改了会在下次部署时被覆盖

/srv/
├── appdata/<app>/                 # 自研应用的持久化数据，bind mount
└── backup/                        # 数据库导出与卷打包

/var/lib/docker/
├── overlay2/                      # 镜像层，可重拉
└── volumes/                       # ★ 跑在 B 的第三方栈命名卷
```

机器 B 上**没有源码、没有编排文件、没有 CI 定义、没有 nginx、没有 certbot**，也没有本仓库的检出。人工只碰 `/srv/`。

### 业务数据为什么分两种放法

| 应用类型 | 放法 | 理由 |
|---|---|---|
| 自研应用 | bind mount 到 `/srv/appdata/<app>/` | 路径可见、备份直观、能单独挂盘；容器内 UID 由自己的 Dockerfile 决定，可控 |
| 第三方栈（Plane 等） | docker 命名卷 | 上游镜像各自用不同 UID（postgres 是 999、minio 是 root），bind mount 要精确 chown，且上游升级后可能变 |

分界线就一句：**Dockerfile 是自己写的就用 bind mount，不是自己写的就用命名卷**。别为了目录整齐去和上游镜像的权限模型较劲。

命名卷的实际位置是 `/var/lib/docker/volumes/<卷名>/_data`，落在服务所在的那台机器上，可以直接 `ls`，只是不建议直接写。

## 五、镜像仓库内的划分

```
registry.internal:5000/
├── apps/<app-name>:<tag>          # 自研镜像，仅 ci 账号可推
└── docker/                        # Docker Hub 代理缓存，首次拉取时自动生成
    ├── library/<name>:<tag>        # 官方镜像，如 docker/library/postgres:15.7-alpine
    └── <org>/<name>:<tag>          # 组织镜像，如 docker/makeplane/plane-backend:v1.4.1
```

`docker/` 下的内容**不需要也不应该手工推送**，Zot 的 sync 扩展在 onDemand 模式下按客户端请求自动回源。地址映射规则由 `config.json` 里 `content.destination` 决定，改了它所有 compose 的镜像地址都要跟着改。

### tag 与回滚深度

自研镜像同时打两个 tag：`<app>:<git-short-sha>` 用于回滚定位，`<app>:latest` 用于人读。

回滚能回多远由 Zot 的保留策略决定：`apps/**` 保留最近 10 次推送，`docker/**` 保留 30 天内被拉过的。参数与调法见 [registry.md](registry.md) 第四节。

## 六、路径与地址的单一事实来源

以下常量只在 Dokploy 的**项目共享变量**里定义一次，任何文档、compose、脚本都用变量引用，不要写死：

| 变量 | 值 | 用处 |
|---|---|---|
| `REGISTRY` | `registry.internal:5000` | 自研镜像仓库地址 |
| `REGISTRY_NS` | `apps` | 自研镜像命名空间 |
| `REGISTRY_CACHE` | `registry.internal:5000/docker` | 公共镜像前缀，留空则各服务直连 Docker Hub |
| `APPDATA_ROOT` | `/srv/appdata` | 自研应用所在机器（机器 B）上 bind mount 的根 |

清单见 [deploy/env/common.env](../deploy/env/common.env)。

## 七、禁止事项

- 机器 B 上手工放源码或编排文件 —— 下次部署会被 Dokploy 覆盖，且造成两处事实
- 手工修改 `/etc/dokploy/` 下的任何内容 —— 同上
- 直接改机器 A 的 `/srv/devops` 里的文件 —— 权威副本在本仓库，改动要走 push 再 pull，否则下次 `git pull` 冲突或被覆盖
- 让 CI 也构建推送镜像 —— 镜像产地必须唯一，见第二节
- 把真实变量值提交进 `deploy/env/` —— 那里只放 `change-me` 占位清单，真值填在 Dokploy UI 里
- 往 `docker/` 前缀推镜像 —— 该前缀由 sync 托管，手工推的东西会被保留策略当作缓存清掉
- 把业务数据写进容器内非卷路径 —— 重新部署即丢失
- 在 compose 里写死 `registry.internal:5000` —— 用 `${REGISTRY}`，换地址时才不用全仓搜索替换
