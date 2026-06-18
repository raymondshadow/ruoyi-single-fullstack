# Quiz RuoYi 单机 K8s 部署设计

## 目标

为 `quiz-ruoyi` 生成一套可复用的单机 Kubernetes 部署脚手架，用于练习接近真实环境的若依项目发布流程。范围包括前端、后端、MySQL、Redis、Ingress、持久化和部署文档，不包含业务功能改造和高可用能力。

## 设计边界

- 保持若依现有前后端代码结构不变。
- 以单机 K8s 为前提，优先练习对象编排和配置注入，不追求数据库高可用。
- MySQL 与 Redis 也部署在当前 K8s 集群中，便于学习 `StatefulSet`、`PVC`、`Secret` 的使用。
- 前端继续使用 `/prod-api` 作为生产 API 前缀，由 Ingress 负责转发和去前缀。
- 后端生产配置改为支持环境变量覆盖，避免把真实密码直接写入仓库。

## 部署架构

### 命名空间

- 单独创建 `quiz-ruoyi` 命名空间，隔离本项目资源。

### 前端

- 使用多阶段 Docker 构建 Vue 3 + Vite 项目。
- 运行时由 Nginx 提供静态资源服务。
- 使用 `Service` 暴露容器 80 端口。

### 后端

- 使用多阶段 Docker 构建 `ruoyi-admin.jar`。
- 运行时使用 Java 17 镜像。
- 通过环境变量注入生产环境中的数据库、Redis、Token 密钥和上传目录。
- 上传目录挂载独立 `PVC`。

### MySQL

- 使用单副本 `StatefulSet`。
- 数据目录挂载 `PVC`。
- 通过 Secret 提供 root 密码、业务库名、业务账号和业务密码。
- 数据库仅自动创建库，不在 YAML 中内嵌大量若依 SQL，初始化表结构由部署文档指引手工导入。

### Redis

- 使用单副本 `StatefulSet`。
- 开启 AOF 持久化。
- 密码通过 Secret 注入。

### Ingress

- 使用两个 Ingress 资源：
  - `/prod-api(/|$)(.*)` 转发到后端，并去掉 `/prod-api` 前缀。
  - `/` 转发到前端。
- 默认示例使用 `nginx` ingress class；如果集群使用 `traefik`，只需要调整 `ingressClassName`。

## 配置策略

- 真实敏感信息不提交到仓库。
- 仓库提供 `06-secrets.example.yaml` 示例模板，使用 `stringData` 便于本地编辑。
- `application-prod.yml` 改为读取环境变量，保留默认值，方便本地和 K8s 双场景复用。

## 验证方式

- 校验 YAML 目录结构和关键镜像引用是否齐全。
- 校验 Dockerfile 是否能映射到当前项目的实际构建命令。
- 校验生产配置占位是否与 K8s 环境变量名称一致。
- 部署层验证由文档给出标准 `kubectl` 和 `docker build` 命令。
