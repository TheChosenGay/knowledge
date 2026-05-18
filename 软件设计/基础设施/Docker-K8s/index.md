---
tags: [moc, 软件设计, 基础设施]
created: 2026-05-15
publish: true
---

# Docker / Kubernetes 专题

> 掌握容器化与云原生部署体系

## Docker 核心知识

### 基础概念
- 镜像（Image）/ 容器（Container）/ 仓库（Registry）
- Union FS 分层存储原理
- Namespace + Cgroup 隔离机制

### Dockerfile 最佳实践
- 多阶段构建（减小镜像体积）
- 合理利用缓存层
- 非 root 用户运行

### 网络模式
- bridge / host / overlay / none
- 容器间通信方案

---

## Kubernetes 核心知识

### 核心对象
- Pod / Deployment / Service / Ingress
- ConfigMap / Secret / PVC

### 调度机制
- Node 亲和性 / Pod 亲和性
- Taints & Tolerations
- Resource Requests & Limits

### 高可用
- HPA 水平自动扩缩
- RollingUpdate 滚动升级
- PodDisruptionBudget

---

## 练习题（待补充）

内容正在整理中，敬请期待。
