# Quiz RuoYi K8s Scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `quiz-ruoyi` 补齐一套单机 K8s 可复用部署脚手架，覆盖镜像构建、K8s 编排、生产配置和中文部署说明。

**Architecture:** 采用多阶段 Docker 构建前后端镜像，使用单独的 `deploy/` 目录存放部署资产。K8s 使用 `Namespace + StatefulSet + Deployment + Service + Ingress + PVC + Secret` 组合，前端通过 `/prod-api` 路由到后端。

**Tech Stack:** Docker, Kubernetes, Nginx, Spring Boot 3, Vue 3, Vite, MySQL 8, Redis 7

---

### Task 1: 补充部署设计与实施文档

**Files:**
- Create: `docs/superpowers/specs/2026-06-17-quiz-ruoyi-k8s-design.md`
- Create: `docs/superpowers/plans/2026-06-17-quiz-ruoyi-k8s-scaffold.md`

- [ ] 写入本次单机 K8s 设计边界、架构拆分、配置策略和验证方式。
- [ ] 写入本次实施计划，明确部署目录、镜像构建方式和 K8s 资源拆分。

### Task 2: 增加前后端 Docker 构建资产

**Files:**
- Create: `deploy/docker/backend/Dockerfile`
- Create: `deploy/docker/frontend/Dockerfile`
- Create: `deploy/docker/frontend/nginx.conf`

- [ ] 为后端创建多阶段 Dockerfile，执行 `mvn -pl ruoyi-admin -am -DskipTests package` 并输出 `ruoyi-admin.jar`。
- [ ] 为前端创建多阶段 Dockerfile，执行 `npm ci` 和 `npm run build:prod`，运行时使用 Nginx 托管静态资源。
- [ ] 为前端补充 SPA 路由和缓存友好的 Nginx 配置。

### Task 3: 增加 K8s 编排文件

**Files:**
- Create: `deploy/k8s/00-namespace.yaml`
- Create: `deploy/k8s/01-mysql.yaml`
- Create: `deploy/k8s/02-redis.yaml`
- Create: `deploy/k8s/03-backend.yaml`
- Create: `deploy/k8s/04-frontend.yaml`
- Create: `deploy/k8s/05-ingress.yaml`
- Create: `deploy/k8s/06-secrets.example.yaml`
- Create: `deploy/k8s/07-pvc.yaml`

- [ ] 创建命名空间和持久化卷声明。
- [ ] 为 MySQL 和 Redis 创建单副本 `StatefulSet` 与 `Service`。
- [ ] 为若依后端和前端创建 `Deployment` 与 `Service`。
- [ ] 创建 Ingress，把 `/prod-api` 转发到后端并保留 `/` 到前端。
- [ ] 创建 Secret 示例文件，统一集中管理数据库、Redis 和 Token 配置。

### Task 4: 调整后端生产配置以适配 K8s

**Files:**
- Modify: `ruoyi-server/ruoyi-admin/src/main/resources/application-prod.yml`
- Modify: `ruoyi-server/ruoyi-admin/src/main/resources/logback.xml`

- [ ] 将 `application-prod.yml` 改成环境变量优先的写法。
- [ ] 将日志目录改成可通过环境变量覆盖，避免容器内路径写死。

### Task 5: 编写部署说明并做静态验证

**Files:**
- Create: `deploy/DEPLOY_K8S.md`

- [ ] 编写中文部署说明，覆盖镜像构建、推送、部署顺序、导入 SQL 和排障命令。
- [ ] 用文本检查确认所有 YAML、Dockerfile 和配置占位已经落地。
