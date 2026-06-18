# Quiz RuoYi 单机 K8s 部署说明

这套部署脚手架适用于单机 Kubernetes 学习场景，目标是尽量贴近企业环境的发布流程，而不是解决高可用问题。

## 目录说明

- `deploy/docker/backend/Dockerfile`：后端镜像构建文件
- `deploy/docker/frontend/Dockerfile`：前端镜像构建文件
- `deploy/docker/frontend/nginx.conf`：前端静态站点配置
- `deploy/k8s/*.yaml`：Kubernetes 资源清单

## 前置条件

部署前请确认：

- 服务器已安装并可使用 Docker
- 服务器已安装并可使用 Kubernetes
- `kubectl` 已连接到目标集群
- 集群里已经有默认 `StorageClass`
- 集群里已经安装 Ingress Controller

如果你的集群不是 `nginx ingress controller`，请把 `deploy/k8s/05-ingress.yaml` 里的 `ingressClassName: nginx` 改成你的实际值，例如 `traefik`。

## 第一步：准备镜像

以下命令默认在项目根目录执行：

```bash
cd /Users/wuyp/Documents/code/quiz-ruoyi
```

### 构建后端镜像

```bash
docker build -f deploy/docker/backend/Dockerfile -t your-registry/quiz-ruoyi-backend:latest .
```

### 构建前端镜像

```bash
docker build -f deploy/docker/frontend/Dockerfile -t your-registry/quiz-ruoyi-frontend:latest .
```

### 推送镜像

```bash
docker push your-registry/quiz-ruoyi-backend:latest
docker push your-registry/quiz-ruoyi-frontend:latest
```

然后把以下文件中的镜像地址改成你的真实仓库地址：

- `deploy/k8s/03-backend.yaml`
- `deploy/k8s/04-frontend.yaml`

## 第二步：准备密钥

编辑 `deploy/k8s/06-secrets.example.yaml`，至少替换以下值：

- `mysql-root-password`
- `mysql-password`
- `redis-password`
- `token-secret`

如果你不想直接改示例文件，可以复制一份：

```bash
cp deploy/k8s/06-secrets.example.yaml deploy/k8s/06-secrets.local.yaml
```

后续命令里把示例文件替换成你自己的本地文件即可。

## 第三步：部署基础资源

按顺序执行：

```bash
kubectl apply -f deploy/k8s/00-namespace.yaml
kubectl apply -f deploy/k8s/07-pvc.yaml
kubectl apply -f deploy/k8s/06-secrets.example.yaml
kubectl apply -f deploy/k8s/01-mysql.yaml
kubectl apply -f deploy/k8s/02-redis.yaml
kubectl apply -f deploy/k8s/03-backend.yaml
kubectl apply -f deploy/k8s/04-frontend.yaml
kubectl apply -f deploy/k8s/05-ingress.yaml
```

## 第四步：导入若依初始化 SQL

当前 YAML 只会自动创建数据库 `ry_vue`，不会自动导入若依表结构。请在 MySQL Pod 就绪后手工导入。

先查看 Pod：

```bash
kubectl get pods -n quiz-ruoyi
```

进入 MySQL：

```bash
kubectl exec -it -n quiz-ruoyi mysql-0 -- sh
```

在本地另一个终端执行拷贝：

```bash
kubectl cp ruoyi-server/sql/ry_20260417.sql quiz-ruoyi/mysql-0:/tmp/ry_20260417.sql
kubectl cp ruoyi-server/sql/quartz.sql quiz-ruoyi/mysql-0:/tmp/quartz.sql
```

然后在容器内导入：

```bash
mysql -u root -p"$MYSQL_ROOT_PASSWORD" ry_vue < /tmp/ry_20260417.sql
mysql -u root -p"$MYSQL_ROOT_PASSWORD" ry_vue < /tmp/quartz.sql
```

## 第五步：检查服务状态

### 查看资源

```bash
kubectl get all -n quiz-ruoyi
kubectl get ingress -n quiz-ruoyi
kubectl get pvc -n quiz-ruoyi
```

### 查看日志

```bash
kubectl logs -f -n quiz-ruoyi deploy/ruoyi-backend
kubectl logs -f -n quiz-ruoyi deploy/ruoyi-frontend
kubectl logs -f -n quiz-ruoyi mysql-0
kubectl logs -f -n quiz-ruoyi redis-0
```

## 第六步：域名与访问

请把 `deploy/k8s/05-ingress.yaml` 里的域名：

```text
quiz-ruoyi.example.com
```

改成你自己的真实域名，并把 DNS 解析到你的服务器公网 IP。

访问路径说明：

- 前端首页：`http://你的域名/`
- 后端接口前缀：`http://你的域名/prod-api`

Ingress 已经负责把 `/prod-api` 转发到后端，并去掉前缀，所以前端无需改动。

## 常见排查

### 1. 前端能打开但登录失败

重点检查：

- `deploy/k8s/05-ingress.yaml` 的 `/prod-api` 路由是否生效
- `ruoyi-backend` 是否正常启动
- MySQL 表是否已经导入
- Redis 密码是否和 Secret 一致

### 2. MySQL Pod 启动失败

重点检查：

- `quiz-ruoyi-secret` 是否存在
- `mysql-root-password` 是否为空
- PVC 是否成功绑定

### 3. 后端启动失败

重点检查：

- `MYSQL_URL`、`MYSQL_USERNAME`、`MYSQL_PASSWORD`
- `REDIS_HOST`、`REDIS_PASSWORD`
- 数据库是否已导入若依表

### 4. Ingress 不生效

重点检查：

- 集群里是否真的装了 Ingress Controller
- `ingressClassName` 是否和当前集群一致
- 域名是否解析到服务器公网 IP
