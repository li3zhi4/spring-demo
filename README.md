# spring-demo

一个用于 CI/CD 测试的简单 Spring Boot 应用（Java 8 + Spring Boot 2.7.18）。

- 接口：`GET /api/hello` → 返回 `Hello, CI/CD!`
- 端口：8080

## 目录结构

```
├── .github/workflows/
│   ├── ci.yml          # CI：编译 + 测试 + 产物上传
│   └── cd.yml          # CD：构建 Docker 镜像推送 GHCR
├── src/                # 源码与测试
├── Dockerfile          # 多阶段构建镜像
├── k8s/                # Kubernetes 部署清单
└── pom.xml
```

## 本地开发

### 环境要求

- JDK 8
- Maven 3.6+
- （可选）Docker

### 编译、测试、运行

```bash
mvn clean package                       # 编译 + 测试 + 打包
java -jar target/spring-demo-0.0.1-SNAPSHOT.jar   # 启动（端口 8080）
curl http://localhost:8080/api/hello    # 验证：Hello, CI/CD!
```

### Docker 构建与运行

```bash
docker build -t spring-demo:local .
docker run -p 8080:8080 spring-demo:local
```

## CI/CD 流程

```
push / PR 到 main
      │
      ▼
┌─────────────────────────┐
│ CI  (.github/workflows/ci.yml)     │
│ ① checkout 拉取代码               │
│ ② setup-java（Temurin JDK 8）     │
│ ③ mvn -B clean package（编译+测试）│
│ ④ 上传 jar 构建产物               │
└─────────────────────────┘
      │ 成功
      ▼
┌─────────────────────────┐
│ CD  (.github/workflows/cd.yml)     │
│ ① 登录 GHCR（GITHUB_TOKEN）       │
│ ② Buildx 多平台构建               │
│ ③ 推送镜像：                      │
│    ghcr.io/li3zhi4/spring-demo    │
│    ├── :latest                     │
│    └── :<commit-sha>               │
└─────────────────────────┘
      │
      ▼
   部署（k8s / 服务器）→ 见下文
```

### 触发条件

| 工作流 | 触发 | 作用 |
|--------|------|------|
| CI | push / PR 到 `main` | 编译、跑测试、上传 jar 产物 |
| CD | push 到 `main` | 构建镜像并推送到 GHCR |

### 镜像

- 仓库地址：`ghcr.io/li3zhi4/spring-demo`
- 为公开镜像，任意环境可直接 `docker pull`，无需认证

## Kubernetes 部署

镜像已发布到 GHCR，可直接在 K8s 集群中使用。部署清单见 `k8s/` 目录。

### 一键部署

```bash
kubectl apply -f k8s/
```

清单包含：

- **Deployment** `spring-demo`：2 副本，滚动更新（`RollingUpdate`），带就绪/存活探针（探测 `/api/hello`）
- **Service** `spring-demo`：NodePort 类型，集群内端口 8080 映射到节点端口 **30080**

### 验证

```bash
kubectl get pods,svc          # 等待 Pod Running 且 READY 2/2
kubectl rollout status deployment/spring-demo   # 查看滚动更新状态
curl http://<节点IP>:30080/api/hello            # 期望输出 Hello, CI/CD!
```

### 更新镜像（新版本发布）

每次 push 到 main，CD 都会推送新的 `<commit-sha>` 标签。更新集群中的镜像：

```bash
# 方式一：指定新的 commit sha 标签
kubectl set image deployment/spring-demo spring-demo=ghcr.io/li3zhi4/spring-demo:<新sha>

# 方式二：手动更新 latest 后触发滚动重启
kubectl rollout restart deployment/spring-demo
```

### 生产建议

- **镜像 tag 不要用 `latest`**：生产环境应固定使用 `:<sha>` 或语义化版本 `v1.0.0`，保证可回滚、可追溯
- **回滚**：`kubectl rollout undo deployment/spring-demo`（回滚到上一个版本）
- **扩缩容**：`kubectl scale deployment/spring-demo --replicas=5`
- **Ingress**：NodePort 只适合测试，生产建议改用 Ingress + TLS 暴露服务
- **私有镜像**：若镜像设为私有，需要创建 `imagePullSecrets`（用 GHCR 的 Personal Access Token 生成）
- **CD 自动部署**：如需 CI/CD 全自动，可在 CD 工作流末尾加一步 `kubectl set image`（集群侧配置 webhook / ArgoCD 等 GitOps 工具效果更佳）
