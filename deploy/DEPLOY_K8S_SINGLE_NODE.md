# Quiz RuoYi 单机 Kubernetes 部署手册

本文档适用于当前腾讯云单节点 Kubernetes 学习环境。部署方式为：

```text
Mac 本机构建 linux/amd64 镜像
        ↓
docker save 导出镜像包
        ↓
scp 上传腾讯云服务器
        ↓
ctr 导入 Kubernetes 的 containerd
        ↓
kubectl 部署 MySQL、Redis、若依后端和前端
```

该方案不依赖镜像仓库，适合单机学习和验证完整发布流程。它不是高可用生产方案。

## 一、当前环境

本文档按以下已验证环境编写：

- 本地开发机：Apple Silicon Mac，Docker Desktop
- 云服务器：腾讯云 Ubuntu 22.04.5 LTS，公网 IP `111.229.254.93`
- Kubernetes：单节点 `v1.36.1`
- 容器运行时：containerd `2.2.1`
- Pod 网络：Flannel
- 默认 StorageClass：`local-path`
- Ingress Controller：ingress-nginx `v1.15.1`
- 公网端口：TCP 80、443 已放行
- 项目目录：`/Users/wuyp/Documents/code/quiz-ruoyi`

如果服务器公网 IP 发生变化，请替换本文所有 `111.229.254.93`。

## 二、部署前检查

### 1. Mac 本机检查

在 Mac 终端执行：

```bash
docker info
ssh ubuntu@111.229.254.93
```

确认 Docker Desktop 正常，并且可以 SSH 登录服务器。

### 2. 服务器检查

SSH 登录服务器后执行：

```bash
kubectl get nodes -o wide
kubectl get storageclass
kubectl get pods -n local-path-storage
kubectl get pods -n ingress-nginx
curl -I http://127.0.0.1
```

预期结果：

- Kubernetes 节点为 `Ready`
- `local-path` 为默认 StorageClass
- Local Path Provisioner 为 `1/1 Running`
- Ingress Controller 为 `1/1 Running`
- `curl` 返回 `HTTP 404`

这里的 `404` 是正常结果，表示请求已经到达 Ingress，但业务路由尚未部署。

## 三、Mac 构建 linux/amd64 镜像

以下命令全部在 Mac 项目根目录执行：

```bash
cd /Users/wuyp/Documents/code/quiz-ruoyi
```

### 1. 构建若依后端镜像

```bash
docker build \
  --platform linux/amd64 \
  -f deploy/docker/backend/Dockerfile \
  -t quiz-ruoyi-backend:latest \
  .
```

后端 Dockerfile 会在 Maven 构建阶段执行：

```bash
mvn -pl ruoyi-admin -am -DskipTests package
```

首次构建需要下载 Maven 依赖，耗时可能较长。

### 2. 构建若依前端镜像

```bash
docker build \
  --platform linux/amd64 \
  -f deploy/docker/frontend/Dockerfile \
  -t quiz-ruoyi-frontend:latest \
  .
```

前端 Dockerfile 会执行 `npm ci` 和 `npm run build:prod`。

### 3. 下载 MySQL 和 Redis 镜像

```bash
docker pull --platform linux/amd64 mysql:8.4
docker pull --platform linux/amd64 redis:7.4-alpine
```

提前在 Mac 下载基础服务镜像，可以避免腾讯云服务器直接访问 Docker Hub 时出现超时或镜像代理错误。

### 4. 验证镜像架构

```bash
docker image inspect quiz-ruoyi-backend:latest \
  --format '{{.Os}}/{{.Architecture}}'

docker image inspect quiz-ruoyi-frontend:latest \
  --format '{{.Os}}/{{.Architecture}}'

docker image inspect mysql:8.4 \
  --format '{{.Os}}/{{.Architecture}}'

docker image inspect redis:7.4-alpine \
  --format '{{.Os}}/{{.Architecture}}'
```

四条命令都必须输出：

```text
linux/amd64
```

### 5. 导出全部运行镜像

```bash
docker save \
  -o /tmp/quiz-ruoyi-images-amd64.tar \
  quiz-ruoyi-backend:latest \
  quiz-ruoyi-frontend:latest \
  mysql:8.4 \
  redis:7.4-alpine

ls -lh /tmp/quiz-ruoyi-images-amd64.tar
```

## 四、准备单机部署清单

不要直接修改仓库中的示例清单。创建一份仅用于当前服务器的部署副本：

```bash
rm -rf /tmp/quiz-ruoyi-single-node
mkdir -p /tmp/quiz-ruoyi-single-node/k8s
mkdir -p /tmp/quiz-ruoyi-single-node/sql

cp deploy/k8s/00-namespace.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp deploy/k8s/01-mysql.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp deploy/k8s/02-redis.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp deploy/k8s/03-backend.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp deploy/k8s/04-frontend.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp deploy/k8s/05-ingress.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp deploy/k8s/07-pvc.yaml \
  /tmp/quiz-ruoyi-single-node/k8s/

cp ruoyi-server/sql/ry_20260417.sql \
  /tmp/quiz-ruoyi-single-node/sql/

cp ruoyi-server/sql/quartz.sql \
  /tmp/quiz-ruoyi-single-node/sql/
```

### 1. 修正 MySQL 8.4 启动参数

MySQL 8.4 已移除 `default_authentication_plugin`。删除示例清单中的旧参数：

```bash
sed -i '' \
  '/--default-authentication-plugin=mysql_native_password/d' \
  /tmp/quiz-ruoyi-single-node/k8s/01-mysql.yaml
```

Mac 的 `sed` 使用 `-i ''`。不要把这个写法直接复制到 Ubuntu。

官方参考：

- [MySQL 8.4 已移除的参数](https://dev.mysql.com/doc/refman/8.4/en/added-deprecated-removed.html)

### 2. 替换后端镜像名

```bash
sed -i '' \
  's#your-registry/quiz-ruoyi-backend:latest#quiz-ruoyi-backend:latest#' \
  /tmp/quiz-ruoyi-single-node/k8s/03-backend.yaml
```

### 3. 替换前端镜像名

```bash
sed -i '' \
  's#your-registry/quiz-ruoyi-frontend:latest#quiz-ruoyi-frontend:latest#' \
  /tmp/quiz-ruoyi-single-node/k8s/04-frontend.yaml
```

两个 Deployment 都已经使用：

```yaml
imagePullPolicy: IfNotPresent
```

因此镜像导入 containerd 后，Kubernetes 会优先使用节点本地镜像。

### 4. 配置本地域名

在没有正式域名时，使用测试域名 `quiz-ruoyi.test`：

```bash
sed -i '' \
  's#quiz-ruoyi.example.com#quiz-ruoyi.test#g' \
  /tmp/quiz-ruoyi-single-node/k8s/05-ingress.yaml
```

浏览器访问前，需要在 Mac 的 `/etc/hosts` 增加：

```text
111.229.254.93 quiz-ruoyi.test
```

可以执行：

```bash
echo '111.229.254.93 quiz-ruoyi.test' \
  | sudo tee -a /etc/hosts
```

验证解析：

```bash
ping -c 1 quiz-ruoyi.test
```

## 五、上传镜像和部署文件

在 Mac 执行：

```bash
scp /tmp/quiz-ruoyi-images-amd64.tar \
  ubuntu@111.229.254.93:/tmp/

scp -r /tmp/quiz-ruoyi-single-node \
  ubuntu@111.229.254.93:~/
```

上传完成后登录服务器：

```bash
ssh ubuntu@111.229.254.93
```

## 六、服务器导入镜像

以下命令全部在腾讯云服务器执行。

### 1. 导入 Kubernetes 镜像空间

```bash
sudo ctr -n k8s.io images import \
  --platform linux/amd64 \
  /tmp/quiz-ruoyi-images-amd64.tar
```

### 2. 检查镜像

```bash
sudo ctr -n k8s.io images list \
  | grep -E 'quiz-ruoyi|mysql|redis'
```

应能看到：

```text
docker.io/library/quiz-ruoyi-backend:latest
docker.io/library/quiz-ruoyi-frontend:latest
docker.io/library/mysql:8.4
docker.io/library/redis:7.4-alpine
```

不同版本的 `ctr` 可能显示完整 digest，但镜像名和 tag 必须存在。

导入成功后清理服务器镜像包：

```bash
rm -f /tmp/quiz-ruoyi-images-amd64.tar
```

不要删除 containerd 中已经导入的镜像。

## 七、创建 Namespace、PVC 和 Secret

进入部署目录：

```bash
cd ~/quiz-ruoyi-single-node
```

### 1. 创建 Namespace

```bash
kubectl apply -f k8s/00-namespace.yaml
```

### 2. 创建 PVC

```bash
kubectl apply -f k8s/07-pvc.yaml
kubectl get pvc -n quiz-ruoyi
```

当前 StorageClass 使用 `WaitForFirstConsumer`，Pod 创建前 PVC 显示 `Pending` 属于正常行为。

### 3. 创建 Secret

不要把生产密码写入 Git 或普通 YAML 文件。通过终端输入：

```bash
read -s -p 'MySQL root password: ' MYSQL_ROOT_PASSWORD
echo

read -s -p 'MySQL ruoyi password: ' MYSQL_PASSWORD
echo

read -s -p 'Redis password: ' REDIS_PASSWORD
echo

TOKEN_SECRET="$(openssl rand -hex 32)"

kubectl create secret generic quiz-ruoyi-secret \
  -n quiz-ruoyi \
  --from-literal=mysql-root-password="$MYSQL_ROOT_PASSWORD" \
  --from-literal=mysql-database="ry_vue" \
  --from-literal=mysql-user="ruoyi" \
  --from-literal=mysql-password="$MYSQL_PASSWORD" \
  --from-literal=redis-password="$REDIS_PASSWORD" \
  --from-literal=token-secret="$TOKEN_SECRET"

unset MYSQL_ROOT_PASSWORD MYSQL_PASSWORD REDIS_PASSWORD TOKEN_SECRET
```

验证 Secret 存在：

```bash
kubectl get secret quiz-ruoyi-secret -n quiz-ruoyi
```

不要使用以下命令输出 Secret 明文：

```text
kubectl get secret ... -o yaml
```

## 八、部署 MySQL 和 Redis

### 1. 部署 MySQL

```bash
kubectl apply -f k8s/01-mysql.yaml

kubectl rollout status statefulset/mysql \
  -n quiz-ruoyi \
  --timeout=300s
```

检查状态：

```bash
kubectl get pod,pvc -n quiz-ruoyi
kubectl logs -n quiz-ruoyi mysql-0 --tail=100
```

预期：

- `mysql-0` 为 `1/1 Running`
- `mysql-data-pvc` 为 `Bound`

### 2. 部署 Redis

```bash
kubectl apply -f k8s/02-redis.yaml

kubectl rollout status statefulset/redis \
  -n quiz-ruoyi \
  --timeout=300s
```

检查状态：

```bash
kubectl get pod,pvc -n quiz-ruoyi
kubectl logs -n quiz-ruoyi redis-0 --tail=100
```

预期：

- `redis-0` 为 `1/1 Running`
- `redis-data-pvc` 为 `Bound`

### 3. 验证 Redis

```bash
kubectl exec -n quiz-ruoyi redis-0 -- \
  sh -c 'redis-cli -a "$REDIS_PASSWORD" ping'
```

预期输出：

```text
PONG
```

命令可能输出 Redis CLI 的密码警告，该警告不影响验证结果。

## 九、导入若依初始化 SQL

SQL 文件已经上传到：

```text
~/quiz-ruoyi-single-node/sql/
```

直接通过标准输入导入，不需要把 SQL 再复制到 Pod：

```bash
kubectl exec -i -n quiz-ruoyi mysql-0 -- \
  sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" ry_vue' \
  < sql/ry_20260417.sql

kubectl exec -i -n quiz-ruoyi mysql-0 -- \
  sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" ry_vue' \
  < sql/quartz.sql
```

验证若依核心表：

```bash
kubectl exec -n quiz-ruoyi mysql-0 -- \
  sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -D ry_vue \
  -e "SHOW TABLES LIKE '\''sys_user'\''; \
      SELECT user_name, status FROM sys_user;"'
```

应该能看到 `sys_user` 表和默认用户数据。

## 十、部署若依后端

### 1. 应用后端清单

```bash
kubectl apply -f k8s/03-backend.yaml

kubectl rollout status deployment/ruoyi-backend \
  -n quiz-ruoyi \
  --timeout=300s
```

### 2. 查看状态和日志

```bash
kubectl get pods -n quiz-ruoyi -o wide

kubectl logs -n quiz-ruoyi \
  deploy/ruoyi-backend \
  --tail=200
```

预期后端 Pod 为 `1/1 Running`。

如果后端反复重启，执行：

```bash
kubectl describe pod -n quiz-ruoyi \
  -l app=ruoyi-backend

kubectl logs -n quiz-ruoyi \
  deploy/ruoyi-backend \
  --previous \
  --tail=200
```

重点检查：

- MySQL 表是否已经导入
- MySQL 和 Redis Pod 是否正常
- Secret 名称和字段是否正确
- 后端是否使用 `prod` Profile

### 3. 集群内验证后端端口

```bash
kubectl run backend-test \
  -n quiz-ruoyi \
  --rm -it \
  --restart=Never \
  --image=busybox:1.36 \
  -- wget -S -O- http://ruoyi-backend:8080/
```

如果服务器无法拉取 `busybox`，可以跳过该测试，等待 Ingress 部署后从公网验证。

## 十一、部署若依前端

```bash
kubectl apply -f k8s/04-frontend.yaml

kubectl rollout status deployment/ruoyi-frontend \
  -n quiz-ruoyi \
  --timeout=300s
```

检查：

```bash
kubectl get pods -n quiz-ruoyi -o wide

kubectl logs -n quiz-ruoyi \
  deploy/ruoyi-frontend \
  --tail=100
```

预期前端 Pod 为 `1/1 Running`。

## 十二、部署 Ingress

```bash
kubectl apply -f k8s/05-ingress.yaml
kubectl get ingress -n quiz-ruoyi
```

预期两个 Ingress 的 Host 都是：

```text
quiz-ruoyi.test
```

验证前端：

```bash
curl -I \
  -H 'Host: quiz-ruoyi.test' \
  http://127.0.0.1/
```

预期返回：

```text
HTTP/1.1 200 OK
```

验证后端验证码接口：

```bash
curl \
  -H 'Host: quiz-ruoyi.test' \
  http://127.0.0.1/prod-api/captchaImage
```

预期返回 JSON，而不是 Nginx 404。

## 十三、Mac 公网访问

确认 Mac 的 `/etc/hosts` 已配置：

```text
111.229.254.93 quiz-ruoyi.test
```

在 Mac 执行：

```bash
curl -I http://quiz-ruoyi.test/

curl http://quiz-ruoyi.test/prod-api/captchaImage
```

浏览器访问：

```text
http://quiz-ruoyi.test/
```

若依默认账号通常为：

```text
账号：admin
密码：admin123
```

如果默认密码被初始化 SQL 修改，以数据库中的实际数据为准。

## 十四、完整验收

服务器执行：

```bash
kubectl get nodes -o wide
kubectl get pods -n quiz-ruoyi -o wide
kubectl get svc -n quiz-ruoyi
kubectl get ingress -n quiz-ruoyi
kubectl get pvc -n quiz-ruoyi
```

应满足：

- `mysql-0`：`1/1 Running`
- `redis-0`：`1/1 Running`
- 若依后端：`1/1 Running`
- 若依前端：`1/1 Running`
- 三个 PVC：`Bound`
- Ingress Host：`quiz-ruoyi.test`
- Mac 浏览器可以打开登录页
- 验证码和登录接口可以正常访问

## 十五、常见问题

### 1. Pod 为 ImagePullBackOff

先检查实际镜像名：

```bash
kubectl describe pod -n quiz-ruoyi <Pod名称>
```

再检查 containerd 是否存在对应镜像：

```bash
sudo ctr -n k8s.io images list \
  | grep -E 'quiz-ruoyi|mysql|redis'
```

确认 Deployment 或 StatefulSet 使用：

```yaml
imagePullPolicy: IfNotPresent
```

### 2. MySQL 报 unknown variable

如果日志包含：

```text
unknown variable 'default-authentication-plugin=...'
```

说明部署清单仍保留了 MySQL 8.4 已移除的旧参数。检查：

```bash
grep authentication k8s/01-mysql.yaml
```

正常情况下不应有输出。

### 3. MySQL 或 Redis PVC 一直 Pending

```bash
kubectl get storageclass
kubectl describe pvc -n quiz-ruoyi mysql-data-pvc
kubectl describe pvc -n quiz-ruoyi redis-data-pvc
kubectl get pods -n local-path-storage
```

只有 PVC、尚未创建消费 Pod 时，`WaitForFirstConsumer` 导致的 `Pending` 是正常的。

### 4. 后端连接数据库失败

```bash
kubectl logs -n quiz-ruoyi deploy/ruoyi-backend --tail=200
kubectl get svc -n quiz-ruoyi
kubectl get pods -n quiz-ruoyi
```

后端通过 Kubernetes Service 名访问：

```text
MySQL：mysql:3306
Redis：redis:6379
```

不要在 Pod 配置中使用 `localhost`。

### 5. 前端打开但登录失败

依次检查：

```bash
curl \
  -H 'Host: quiz-ruoyi.test' \
  http://127.0.0.1/prod-api/captchaImage

kubectl get ingress -n quiz-ruoyi

kubectl logs -n quiz-ruoyi \
  deploy/ruoyi-backend \
  --tail=200
```

`/prod-api` 必须由 Ingress 转发到后端，并通过 rewrite 去掉该前缀。

### 6. 公网仍然返回 Ingress 404

当前 Ingress 根据 Host 路由。使用公网 IP 直接请求不会匹配 `quiz-ruoyi.test`。

正确测试：

```bash
curl -I \
  -H 'Host: quiz-ruoyi.test' \
  http://111.229.254.93/
```

浏览器访问前必须配置 `/etc/hosts`。

### 7. Pod 显示 Running 但一直不 Ready

```bash
kubectl describe pod -n quiz-ruoyi <Pod名称>
kubectl logs -n quiz-ruoyi <Pod名称> --tail=200
```

重点查看 Readiness Probe 和容器事件。

## 十六、更新前后端版本

修改代码后，在 Mac 重新构建：

```bash
cd /Users/wuyp/Documents/code/quiz-ruoyi

docker build \
  --platform linux/amd64 \
  -f deploy/docker/backend/Dockerfile \
  -t quiz-ruoyi-backend:latest \
  .

docker build \
  --platform linux/amd64 \
  -f deploy/docker/frontend/Dockerfile \
  -t quiz-ruoyi-frontend:latest \
  .

docker save \
  -o /tmp/quiz-ruoyi-app-images-amd64.tar \
  quiz-ruoyi-backend:latest \
  quiz-ruoyi-frontend:latest

scp /tmp/quiz-ruoyi-app-images-amd64.tar \
  ubuntu@111.229.254.93:/tmp/
```

服务器执行：

```bash
sudo ctr -n k8s.io images import \
  --platform linux/amd64 \
  /tmp/quiz-ruoyi-app-images-amd64.tar

kubectl rollout restart deployment/ruoyi-backend \
  -n quiz-ruoyi

kubectl rollout restart deployment/ruoyi-frontend \
  -n quiz-ruoyi

kubectl rollout status deployment/ruoyi-backend \
  -n quiz-ruoyi \
  --timeout=300s

kubectl rollout status deployment/ruoyi-frontend \
  -n quiz-ruoyi \
  --timeout=300s

rm -f /tmp/quiz-ruoyi-app-images-amd64.tar
```

## 十七、查看日志

```bash
kubectl logs -f -n quiz-ruoyi deploy/ruoyi-backend
kubectl logs -f -n quiz-ruoyi deploy/ruoyi-frontend
kubectl logs -f -n quiz-ruoyi mysql-0
kubectl logs -f -n quiz-ruoyi redis-0
```

退出日志跟踪按 `Ctrl+C`。

## 十八、停止或删除环境

### 1. 仅停止前后端

```bash
kubectl scale deployment/ruoyi-backend \
  -n quiz-ruoyi \
  --replicas=0

kubectl scale deployment/ruoyi-frontend \
  -n quiz-ruoyi \
  --replicas=0
```

恢复：

```bash
kubectl scale deployment/ruoyi-backend \
  -n quiz-ruoyi \
  --replicas=1

kubectl scale deployment/ruoyi-frontend \
  -n quiz-ruoyi \
  --replicas=1
```

### 2. 删除整个项目

```bash
kubectl delete namespace quiz-ruoyi
```

注意：删除 Namespace 会删除 Deployment、StatefulSet、Service、Ingress、Secret 和 PVC。Local Path Provisioner 的回收策略为 `Delete`，对应数据可能同时被删除。

执行删除前应先备份 MySQL 数据。

## 十九、清理 Mac 临时文件

确认服务器部署完成后，在 Mac 执行：

```bash
rm -f /tmp/quiz-ruoyi-images-amd64.tar
rm -f /tmp/quiz-ruoyi-app-images-amd64.tar
rm -rf /tmp/quiz-ruoyi-single-node
```

## 二十、下一阶段

单机 IP 部署跑通后，再逐步增加：

1. 使用正式域名替换 `quiz-ruoyi.test`
2. 配置 cert-manager 或正式 TLS 证书
3. 使用腾讯云 TCR 保存镜像
4. 使用固定版本号替代 `latest`
5. 增加资源 requests、limits 和备份策略
6. 使用 GitHub Actions 或公司 CI/CD 自动构建和发布

