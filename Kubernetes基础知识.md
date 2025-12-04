## Kubernetes基础知识

### 1. Kubernetes 是什么

Kubernetes（简称 K8s）是 Google 开源的容器编排平台，负责在集群中自动部署、扩缩、滚动升级和自愈容器化应用。它解决了单机 Docker 难以管理大量容器、跨主机通信复杂、调度与高可用难度大的问题，是云原生技术栈的核心底座。

#### 1.1 主要能力

- **声明式部署**：通过 YAML/JSON 描述应用期望状态，K8s 控制面自动让实际状态收敛到期望状态。
- **自动调度与弹性**：根据资源需求、亲和/反亲和、污点等策略把 Pod 调度到最合适的 Node，并支持 HPA/VPA 实现弹性伸缩。
- **服务治理**：Service、Ingress 提供稳定访问入口，内置负载均衡与服务发现。
- **自愈**：探针失败或节点异常时自动重建 Pod，保证应用可用性。
- **配置与密钥管理**：ConfigMap、Secret 解耦配置，支持热更新与安全分发。
- **多租户与安全**：Namespace、RBAC、NetworkPolicy 让隔离与权限控制更精细。

#### 1.2 典型使用场景

?> **常见业务**：微服务后端、批处理任务、机器学习训练/推理、边缘计算、CI/CD 构建集群等。

- 快速上线/下线新版本，支持蓝绿、金丝雀发布。
- 高并发业务自动扩缩容，提升资源利用率。
- 与云厂商 LoadBalancer、CSI、CNI 集成，构建混合云或多云架构。

---

### 2. 集群核心架构

Kubernetes 集群由 **控制平面（Control Plane）** 与 **工作节点（Node）** 组成。

| 角色 | 关键组件 | 说明 |
| --- | --- | --- |
| API Server | `kube-apiserver` | 集群统一入口，提供 REST API，负责认证、授权、准入控制。 |
| 数据存储 | `etcd` | 强一致性键值库，保存所有集群状态。建议高可用部署。 |
| 调度 | `kube-scheduler` | 根据资源、拓扑、亲和/反亲和等策略为新 Pod 选择最优节点。 |
| 控制器 | `kube-controller-manager` | 负责副本控制、节点管理、HPA、ServiceAccount 等控制循环。 |
| 节点代理 | `kubelet` | 运行在 Node 上，监听 PodSpec 并与容器运行时交互，汇报 Pod 状态。 |
| 网络代理 | `kube-proxy` | 维护 Service 虚拟 IP，基于 iptables/ipvs 实现四层负载均衡。 |
| 容器运行时 | containerd / CRI-O / Docker Shim(废弃) | 负责真正拉取镜像、创建/销毁容器。 |

> **控制面高可用**：建议至少 3 个控制节点 + 3 节点的 etcd 集群，通过负载均衡对外暴露 API。

---

### 3. 基础资源对象速查

- **Namespace**：逻辑隔离租户或环境。
- **Pod**：最小调度单位，可包含多个共享网络与 Volume 的容器。
- **ReplicaSet / Deployment**：声明副本数、滚动升级策略，Deployment 是更常用的上层控制器。
- **StatefulSet**：有序部署、稳定网络标识、持久化存储，常用于数据库、消息队列。
- **DaemonSet**：在每个（或一组）节点都部署一份 Pod，例如日志收集、监控 Agent。
- **Job / CronJob**：一次性或周期性批任务。
- **Service**：为一组 Pod 暴露稳定的虚拟 IP（ClusterIP/NodePort/LoadBalancer/ExternalName）。
- **Ingress**：七层路由控制器，把 HTTP(S) 请求转发到不同 Service，可配合 TLS、Rewrite、WAF。
- **ConfigMap / Secret**：注入配置与敏感信息（Base64 编码），支持挂载为文件或环境变量。
- **PersistentVolume(PV)/PersistentVolumeClaim(PVC)**：抽象不同存储后端，提供 Pod 持久化数据能力。
- **HorizontalPodAutoscaler(HPA)**：基于 CPU/内存或自定义指标自动伸缩副本数。
- **ResourceQuota / LimitRange**：限制 Namespace 资源上限与 Pod/Container 的默认配额。

---

### 4. 从部署到访问的工作流

1. **编写镜像**：使用 Dockerfile 构建业务镜像并推送到镜像仓库（Harbor、ACR、ECR 等）。
2. **声明资源**：编写 Deployment/StatefulSet/Service/Ingress 等 YAML，描述期望状态。
3. **kubectl apply**：将 manifest 提交到 API Server。
4. **控制循环**：Scheduler 选择 Node，kubelet 拉取镜像并启动 Pod，Controller 持续校正状态。
5. **流量接入**：通过 Service 在集群内互访，通过 Ingress/LoadBalancer 对外暴露。
6. **监控与告警**：利用 Metrics Server、Prometheus、Grafana 或云监控观察资源、探针、事件。

!> **常见问题排查思路**：先看 `kubectl get pod -o wide` 与 `describe` 事件，再查看 `kubectl logs`，必要时登录 Node 检查容器运行时和系统资源。

---

### 5. 典型 YAML 模板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: demo-web
	namespace: prod
spec:
	replicas: 3
	revisionHistoryLimit: 2
	strategy:
		type: RollingUpdate
		rollingUpdate:
			maxUnavailable: 1
			maxSurge: 1
	selector:
		matchLabels:
			app: demo-web
	template:
		metadata:
			labels:
				app: demo-web
		spec:
			containers:
				- name: web
					image: registry.local/demo/web:1.0.0
					ports:
						- containerPort: 8080
					resources:
						requests:
							cpu: "200m"
							memory: "256Mi"
						limits:
							cpu: "500m"
							memory: "512Mi"
					envFrom:
						- configMapRef:
								name: demo-web-config
					readinessProbe:
						httpGet:
							path: /healthz
							port: 8080
						initialDelaySeconds: 5
						periodSeconds: 10
					livenessProbe:
						httpGet:
							path: /live
							port: 8080
						initialDelaySeconds: 15
						periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
	name: demo-web-svc
	namespace: prod
spec:
	type: ClusterIP
	selector:
		app: demo-web
	ports:
		- port: 80
			targetPort: 8080
```

如果需要对外暴露，可追加 Ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: demo-web-ingress
	namespace: prod
	annotations:
		nginx.ingress.kubernetes.io/rewrite-target: /
spec:
	ingressClassName: nginx
	tls:
		- hosts:
				- demo.example.com
			secretName: demo-tls
	rules:
		- host: demo.example.com
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: demo-web-svc
								port:
									number: 80
```

---

### 6. kubectl 常用命令

| 操作 | 命令 |
| --- | --- |
| 查看集群信息 | `kubectl cluster-info` / `kubectl get nodes -o wide` |
| 切换上下文 | `kubectl config use-context <ctx>` |
| 查看资源 | `kubectl get pod,svc,deploy -A` |
| 查看事件 | `kubectl get events --sort-by=.lastTimestamp` |
| 查看详情 | `kubectl describe pod <name>` |
| 查看 Pod 日志 | `kubectl logs <pod> [-c container]` |
| 进入容器 | `kubectl exec -it <pod> -- /bin/sh` |
| 应用/删除 manifest | `kubectl apply -f xxx.yaml` / `kubectl delete -f xxx.yaml` |
| 临时运行 Pod | `kubectl run tmp --rm -it --image=busybox -- sh` |
| 资源占用 | `kubectl top pod` / `kubectl top node`（需安装 Metrics Server） |

?> 建议将 `kubectl` 与 `krew`、`kubectx/kubens`、`stern` 等工具结合使用，提升排障效率。

---

### 7. 健康检查与扩缩容

- **探针**：Readiness 决定是否加入负载均衡；Liveness 决定是否重启容器；Startup 用于慢启动应用。
- **滚动升级**：Deployment 默认滚动更新，支持设置 `maxSurge`、`maxUnavailable`，也可配置 `partition` 进行灰度。
- **回滚**：`kubectl rollout undo deployment/xxx` 可以快速回退。
- **自动扩容**：
	- HPA：按指标（CPU/内存或自定义 metrics）调整副本数。
	- VPA：自动建议或更新请求/限制。
	- Cluster Autoscaler：在云环境下按需扩容/缩容节点。

---

### 8. 日志、监控与排错

1. **日志**：`kubectl logs` 只看当前容器，建议结合 ELK / Loki / Fluentd 收集集中式日志。
2. **监控**：常见组合为 Prometheus + Alertmanager + Grafana，或使用云厂商托管监控。
3. **事件**：`kubectl describe` 和 `kubectl get events` 是排障第一手资料。
4. **网络故障**：检查 CNI 插件（Calico、Flannel、Cilium），并排查 `kube-proxy` 与 Node iptables。
5. **节点健康**：`kubectl get node` + `kubectl describe node` 查看污点、磁盘、内存压力，必要时 `systemd` 与 `journalctl`。
6. **镜像问题**：确认镜像拉取凭据（ImagePullSecret）、仓库可达性、Tag 是否存在。

---

### 9. 最佳实践速记

!> **安全第一**：

1. 开启 RBAC、审计日志，避免使用 `system:masters` 级别凭据。
2. 通过 NetworkPolicy/Service Mesh 限制跨 Namespace 访问。
3. Secret 使用 KMS 加密，敏感信息避免写在镜像或 Git 仓库。

?> **运维建议**：

- 资源请求/限制必须填写，结合 Prometheus 数据持续调优。
- 利用 Taints/Tolerations、NodeSelector、NodeAffinity 控制工作负载位置。
- 定期备份 etcd（快照 + 证书），并演练恢复流程。
- 采用 GitOps（Argo CD / Flux）或 Helm/Kustomize 管理大量 YAML，保障可追溯。
- 利用 `kubectl diff` 或 `kube-score`、`polaris` 等工具在上线前做配置审计。

---

### 10. 学习路径推荐

1. 熟悉容器与 Docker 基础（镜像、容器、网络、存储）。
2. 搭建本地实验环境：kind、minikube、K3s、Rancher Desktop。
3. 掌握核心资源：Pod/Deployment/Service/Ingress/ConfigMap/Secret。
4. 学习存储（PV/PVC/CSI）、网络（CNI、Service/Ingress）、安全（RBAC、OPA）。
5. 持续实践：CI/CD（Tekton/Argo Workflows）、Service Mesh（Istio/Linkerd）、Observability（Prometheus、Jaeger）。

通过以上内容，你可以快速建立 Kubernetes 的整体认知，并在项目中落地高可用的容器化平台。




