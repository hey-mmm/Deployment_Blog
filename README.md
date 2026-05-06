
# 📝 Typecho 博客系统 Kubernetes 云原生部署方案

> **架构描述**：基于 Kubernetes 的高可用 Typecho 博客集群，采用微服务架构分离应用与数据存储。

本项目旨在展示如何利用云原生技术栈（Kubernetes, Docker, MySQL）构建一个安全、可扩展的个人博客平台。项目严格遵循基础设施即代码（IaC）原则，实现了应用配置与敏感信息的分离。

---

## 🛠️ 核心架构组件

本系统由以下核心组件构成，实现了应用层与数据层的解耦：

| 组件名称 | 技术栈 | 描述 | 职责 |
| :--- | :--- | :--- | :--- |
| **前端应用** | Typecho (Docker) | 轻量级开源博客系统 | 处理 HTTP 请求，渲染页面 |
| **数据库** | MySQL 5.7 | 关系型数据库 | 存储博客文章、评论及配置 |
| **编排平台** | Kubernetes (K8s) | 容器编排引擎 | 负责 Pod 的调度、扩缩容与生命周期管理 |
| **存储方案** | HostPath PV | 本地持久化存储 | 确保容器重启后数据不丢失 |

### 🔐 安全与配置管理
- **ConfigMap**：用于管理非敏感配置。
- **Secret**：用于管理敏感凭证，Base64 加密存储。

---

## 🚀 部署指南



### 1. 环境初始化
如果您是在本地虚拟机或物理机搭建环境，请先执行初始化脚本（参考 `init.txt` 和 `download.txt`）：
```bash
# 1. 配置系统环境
chmod +x init.txt && ./init.txt

# 2. 安装 Docker, Containerd, Kubernetes 
chmod +x download.txt && ./download.txt
```

### 2. 创建命名空间
系统将部署在独立的 `blog` 命名空间中，以实现资源隔离。
```bash
# 创建 blog 命名空间
kubectl create namespace blog
```

### 3. 部署基础设施
按顺序执行以下部署文件，确保依赖关系正确：

```bash
# 1. 创建持久化卷和配置管理
# 包含：Typecho PV, MySQL PV, ConfigMap, Secret
kubectl apply -f blog-sources.txt

# 2. 部署 MySQL 数据库
# 包含：MySQL Deployment, PVC, Service
kubectl apply -f blog-mysql-deploment.txt

# 3. 部署 Typecho 应用
# 包含：Typecho Deployment, PVC, Service
kubectl apply -f typecho-deploment.txt
```

### 4. 验证部署
部署完成后，检查 Pod 状态是否为 `Running`：
```bash
kubectl get pods -n blog

# 预期输出：
# NAME                       READY   STATUS    RESTARTS   AGE
# mysql-5b594cdb5f-abcde     1/1     Running   0          2m
# typecho-7d6f8c9b4e-xyzab   1/1     Running   0          1m
```

---

## 🔗 访问服务

应用部署成功后，您可以通过以下方式访问：

- **博客前台访问**：
  - URL: `http://<Node-IP>:30080`
  - 端口映射：NodePort `30080` -> 容器端口 `80`
  - **首次访问**：请按照 Typecho 安装向导完成初始化。

- **数据库连接**（内部）：
  - Host: `blog-mysql` (ClusterIP Service)
  - Port: `3306`
  - DBName: `typecho`
  - User/Pass: `root` / `1` (由 Secret 注入)

---

## 📂 项目文件结构说明

本项目严格遵循单一职责原则，文件结构清晰：

| 文件名 | 内容概要 | 说明 |
| :--- | :--- | :--- |
| `blog-sources.txt` | PV, ConfigMap, Secret | **核心配置层**。包含存储定义、环境变量及敏感凭证。 |
| `blog-mysql-deploment.txt` | MySQL 控制器与服务 | **数据层**。包含 StatefulSet、PVC 及 Headless Service。 |
| `typecho-deploment.txt` | Typecho 控制器与服务 | **应用层**。包含 Deployment、PVC 及 NodePort Service。 |
| `init.txt` | 系统初始化脚本 | 用于 Ubuntu 系统的基础环境配置。 |
| `download.txt` | K8s 安装脚本 | 一键安装 Docker, K8s 及 Calico 网络插件。 |

---

## 🛡️ 最佳实践与优化

### 1. 配置与代码分离
本项目未将数据库密码硬编码在 YAML 中，而是通过 **Kubernetes Secret** 动态注入。这符合 [12-Factor App](https://12factor.net/) 设计原则，确保了代码仓库的安全性。

### 2. 数据持久化
利用 **PersistentVolume (PV)** 和 **PersistentVolumeClaim (PVC)** 机制，将容器内的 `/var/lib/mysql` 和 `/app/usr` 目录挂载到宿主机的 `/root/k8s/blog/data/` 目录。即使 Pod 被销毁重建，数据依然保留。

### 3. 环境一致性
通过 ConfigMap 统一管理 `dbname` 和 `host`，避免了因环境差异（开发/测试/生产）导致的配置错误。

---

```
