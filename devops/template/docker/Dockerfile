# 多阶段构建：打包与运行分离，最终镜像不含源码和构建工具
# 前端项目示例，后端替换构建阶段即可（见 devops/README.md）

# ---------- 构建阶段 ----------
FROM node:20-alpine AS builder

WORKDIR /app

# 依赖单独一层，package.json 不变时直接命中缓存
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ---------- 运行阶段 ----------
FROM nginx:1.27-alpine

COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

# Dokploy 按健康状态判定部署成败，必须保留
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost/healthz > /dev/null || exit 1
