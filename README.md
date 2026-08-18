# 目录说明

Dokploy + Harbor 的两机部署方案：机器 A 做自动化部署，机器 B 跑应用服务。

## 操作手册

| 文档 | 内容 |
|---|---|
| [devops/README.md](devops/README.md) | 两台机器的环境要求与全部操作步骤 |
| [devops/harbor/README.md](devops/harbor/README.md) | Harbor 安装、初始化与运维 |
| [plane-docker-deploy.md](plane-docker-deploy.md) | Plane 的变量配置与初始化 |

## 仓库骨架

| 路径 | 对应 | 说明 |
|---|---|---|
| `devops/deploy-repo/` | 部署仓库 | 存放各项目 compose 与变量清单，由 Dokploy 从 Gitee 直接拉取，不需克隆到服务器 |
| `devops/template/` | 每个应用仓库 | 只含 Dockerfile 与 nginx 配置，触发由 Gitee webhook 直达 Dokploy |

## 三处集中管理

| 对象 | 位置 |
|---|---|
| 源代码 | Dokploy（机器 A）统一拉取，机器 B 不放源码 |
| 编排 | 部署仓库 `projects/<project>/`，Compose 服务用 git provider 拉取，仍受版本控制 |
| docker 镜像 | Harbor（机器 A），自研走 `apps`、公共走 `dockerhub` 代理缓存 |
| 环境变量 | Dokploy 项目共享变量 + 服务变量，`${{project.VAR}}` 引用 |

## 约定

所有文件 LF 换行，由 `.editorconfig`、`.gitattributes`、`.vscode/settings.json` 三处保证。
