# 部署与运维优化指南

本文针对 AIMangaStudio 的前端应用与潜在后端服务，给出可落地的容器化、发布、监控和性能优化实践，便于团队在 CI/CD 中快速复用。

## 1. 容器化与静态资源发布
### 前端（Vite SPA / 静态导出）
- 使用多阶段构建：在 `node:18-alpine` 中安装依赖并构建，再用 `nginx:alpine` 仅承载 `dist/` 产物。
- 环境变量在构建时注入（如 `VITE_API_BASE_URL`），不要写死密钥；生产配置通过 `.env.production` 或 CI/CD 注入。
- 示例 Dockerfile：
  ```Dockerfile
  FROM node:18-alpine AS builder
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci
  COPY . .
  RUN npm run build

  FROM nginx:alpine AS runner
  COPY --from=builder /app/dist /usr/share/nginx/html
  COPY ./nginx.conf /etc/nginx/conf.d/default.conf
  EXPOSE 80
  CMD ["nginx", "-g", "daemon off;"]
  ```
- 若走 CDN：构建后将 `dist/assets/*` 上传到对象存储并绑定 CDN，HTML 仍由应用服务器或静态站点托管；确保使用内容哈希命名并开启 gzip/Brotli。

### 后端（如引入 API 层）
- 采用多阶段构建，运行层使用精简镜像（distroless/alpine），并在入口处理中 `SIGTERM` 以支持优雅关停。
- 配置健康检查端点 `/healthz`、`/ready`；镜像中不写入密钥，依赖环境变量/Secret 管理。

## 2. 数据库迁移与备份
- 在 CI/CD 中单独 job 运行迁移（Liquibase/Flyway/Prisma 等），并与发布解耦：先发布兼容版本 → 执行迁移 → 清理旧字段。
- 大表变更使用在线迁移或分批（PT-OSC/gh-ost/pg_repack），避免长时间锁表。
- 备份策略：日全量 + 小时级增量/日志（Binlog/WAL）；开启加密与跨可用区存储，定期做恢复演练并记录 RPO/RTO。

## 3. 灰度发布与滚动更新
- **流量切分**：在入口网关按用户、地域、Header 或 Feature Flag 逐步放量，支持快速回滚。
- **K8s 滚动**：Deployment 设置 `maxUnavailable=0`、`maxSurge=1`，配置就绪/存活探针；Pod 捕获 `SIGTERM` 关闭新连接、等待在途请求。
- **前后端兼容**：前端透传版本或 Feature 标记，后端按标记返回兼容响应；保留上一版本镜像以便一键回滚。

## 4. 监控与日志
- **APM/Tracing**：使用 OpenTelemetry SDK（前端可用 otel-web）上报 Trace & Metrics；在网关保留 `traceparent` 头贯通链路。
- **日志聚合**：输出结构化 JSON，包含 `trace_id`/`user_id`/版本；通过 Fluent Bit/Vector 侧车收集到 ELK 或 CloudWatch，并按服务设置采样。
- **错误上报**：前端/后端接入 Sentry，上传 Source Map、版本号；对 PII 做脱敏，对高频错误做抑制与聚合。
- **关键告警**：区分 P0/P1/P2，重点监控 QPS、P95 延迟、错误率、队列堆积、慢查询、存储容量、鉴权失败率。

## 5. 性能优化清单
- **CDN 缓存**：静态资产长缓存（`max-age=31536000, immutable`），HTML/SSR 短缓存或 `no-store`；结合 `ETag` 与 `stale-while-revalidate`。
- **图片与媒体**：构建时或通过图片服务生成 WebP/AVIF，按 DPR/终端自适应；启用懒加载与占位图，边缘开启压缩。
- **接口缓存**：读多写少接口在应用侧/Redis 设置 TTL；网关对 GET 接口做短 TTL 缓存，缓存键包含路径+查询+关键 Header。
- **数据库**：为热点查询加覆盖/组合索引，避免 N+1；大分页用 Keyset；连接池上限、超时与重试策略与实例规格匹配。

## 6. CI/CD 流水线示例
1) Lint/Test/Build → 2) 镜像构建与漏洞扫描（Trivy/Cosign 签名） → 3) 迁移 Job → 4) 部署到灰度环境 → 5) 健康检查通过后滚动至全量 → 6) 监控与告警验证。

## 7. 运维自检清单
- 镜像是否使用不可变 tag（含 commit SHA），是否有 SBOM 与签名校验。
- CDN 缓存命中率、回源流量与 4xx/5xx 是否可视化。
- 备份/恢复演练记录是否符合预期 RPO/RTO；灰度回滚是否能在 <5 分钟完成。
- Sentry 高频错误与 APM 性能瓶颈是否有处理 SLA。
