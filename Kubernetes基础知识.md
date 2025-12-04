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

---

### 11. Service Mesh 与 Gateway API

Service Mesh（如 Istio、Linkerd、Kuma）通过 **Sidecar** 把流量治理能力（熔断、限流、重试、灰度、可观测性、零信任 mTLS）从业务代码中剥离，适合大规模微服务场景。

- **控制面** 负责配置下发（Istiod、Kuma CP）。
- **数据面** 通常是 Envoy/Linkerd Proxy，拦截进出流量。
- **进阶能力**：
	- L7 细粒度路由与 AB/金丝雀发布。
	- 流量镜像、超时、重试、熔断。
	- mTLS、证书轮换、策略和遥测。
- **Gateway API** 是新版 Ingress/Service Mesh 标准，提供 `GatewayClass`、`HTTPRoute` 等资源，更易扩展多租户和多实现，推荐在新项目中取代传统 Ingress Annotation 方案。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
	name: canary-route
spec:
	parentRefs:
		- name: web-gateway
	rules:
		- matches:
				- path:
						type: PathPrefix
						value: /
			backendRefs:
				- name: web-v1
					weight: 80
				- name: web-v2
					weight: 20
```

?> **什么时候采用 Mesh？** 当服务数量较多、需要统一治理、安全要求高、或要实现跨集群流量策略时考虑引入；对于简单场景保持 Ingress + Service 即可，避免过度工程化。

---

### 12. Operator 与 CRD 生态

Kubernetes 允许通过 **CustomResourceDefinition（CRD）** 扩展 API，对应的 **Controller/Operator** 通过 Watch + Reconcile 实现自定义业务逻辑，让复杂系统（数据库、中间件、批任务）以声明式方式运行。

- **核心概念**：
	- Custom Resource 描述业务状态（如 `KafkaCluster`、`SparkApplication`）。
	- Controller 监听资源变化并执行操作（创建 Pod、配置网络、回收资源）。
	- Operator = CRD + Controller + 领域知识库。
- **常见工具链**：Operator SDK、Kubebuilder、KubeVirt、Crossplane。
- **应用场景**：自动化数据库运维（MySQL Operator）、大数据平台（Spark Operator）、云资源编排（Crossplane 将 AWS/Azure/GCP 资源映射为 CRD）。

!> 编写 Operator 需关注幂等性与状态机设计，确保 Reconcile 可重试；同时做好权限控制（精确 RBAC）、指标与日志，方便排障。

---

### 13. 存储与网络进阶

**存储（CSI）**

- CSI（Container Storage Interface）统一了第三方存储接入方式，动态 Provisioner 根据 PVC 自动创建卷。
- 关键功能：快照（VolumeSnapshot）、克隆、扩容、访问模式（RWO/RWX/RWX）。
- 实践建议：
	- StatefuSet + PVC + StorageClass 组合持久化数据库。
	- 使用 Topology 亲和（Zone、Rack）和 PodDisruptionBudget 保证数据安全。
	- 对性能敏感 workload 使用本地盘（Local PV）并结合备份策略。

**网络（CNI/eBPF）**

- CNI 插件：Flannel（简单）、Calico（BGP/NetworkPolicy）、Cilium（eBPF、L7 可视化）。
- 网络策略使用 `NetworkPolicy` 或 CiliumNetworkPolicy 实现零信任网段隔离。
- eBPF 技术可在内核态实现负载均衡、可观测性（Hubble）、安全审计。
- 大规模集群可引入 BGP/EVPN 与硬件交换机集成，降低 overlay 开销。

---

### 14. 多集群、联邦与混合云

随着业务全球化或多租户隔离需求，常见的多集群策略：

1. **Hub-Spoke 管控**：使用 Rancher、Red Hat ACM、Karmada、Clusternet 集中管理多集群。
2. **Service Mesh 多集群**：Istio、Linkerd 支持跨集群流量治理与身份统一。
3. **Multi-Cluster Ingress/Global LB**：通过云厂商 GSLB 或开源 Submariner、Skupper 建立跨集群网络。
4. **联邦（Federation/Karmada）**：在多集群间同步资源，实现跨区域容灾与就近调度。
5. **混合云/边缘**：KubeEdge、OpenYurt 把边缘节点纳入 K8s 控制，实现云边协同。

?> **实战建议**：
- 定义统一的 Namespace/Label/Annotation 约定，配合 GitOps 一次声明多集群生效。
- 跨集群网络务必考虑 DNS、一致的 Service CIDR，以及证书管理。
- 观测体系（集中式 Prometheus/Thanos、日志聚合）需要跨集群联通或 Sidecar 上传。

---

### 15. GitOps 与自动化交付

GitOps 通过「声明即事实、拉模式对齐」把集群状态与 Git 仓库保持一致，常用工具为 **Argo CD**、**Flux**。

- **流程**：开发提交 -> CI 构建镜像 -> 更新 Helm/Kustomize/Manifest -> GitOps Controller Watch 仓库并同步到集群。
- **优势**：可追溯、审核友好、可回滚；支持多环境分支、Promotion Flow。
- **关键能力**：
	- 多租户隔离（AppProject、Namespace 设限）。
	- Health Check、Sync Policy、自动漂移检测。
	- 与 Secrets Manager（Sealed Secrets、External Secrets）结合，实现安全配置分发。
- **管控建议**：
	- Git 仓库设置分支保护 + Code Review；
	- 结合 OPA/Gatekeeper/kyverno 做策略校验，阻断不合规配置；
	- 使用 Helmfile、Kustomize Patch、Jsonnet 等手段管理大规模模板。

借助这些进阶主题，你可以从「会用 Kubernetes」迈向「构建企业级平台」的阶段，覆盖服务治理、自动化运维、跨集群混合云以及 DevSecOps 全链路。

---

### 16. 环境搭建步骤（本地/云）

目标：快速从零到一，能起集群、部署应用、提供外部访问。

1) 本地最简：kind（推荐）

```powershell
# Windows PowerShell 安装 kind（需已安装 Docker Desktop）
winget install Kubernetes.kind

# 创建单节点集群
kind create cluster --name dev

# 验证
kubectl cluster-info; kubectl get nodes
```

2) 本地单机：minikube

```powershell
winget install Kubernetes.minikube
minikube start --driver=docker
kubectl get po -A
```

3) 云上托管：AKS/EKS/GKE（面试常问）

- 使用托管控制面，工作节点按节点池管理；与 VPC/VNet 深度集成。
- 学会基本命令：`az aks create`、`aws eks update-kubeconfig` 等；理解云负载均衡、存储类与云盘映射、容器日志与监控接入。

---

### 17. 核心资源详解与模板

1) Pod：最小调度单位，包含一个或多个容器，共享网络命名空间与 Volume。

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: demo-pod
spec:
	containers:
		- name: app
			image: nginx:1.25
			ports:
				- containerPort: 80
			resources:
				requests:
					cpu: "100m"
					memory: "128Mi"
				limits:
					cpu: "500m"
					memory: "256Mi"
```

2) Deployment：无状态服务的声明式副本控制与滚动升级。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: web-deploy
spec:
	replicas: 3
	strategy:
		type: RollingUpdate
		rollingUpdate:
			maxSurge: 1
			maxUnavailable: 0
	selector:
		matchLabels:
			app: web
	template:
		metadata:
			labels:
				app: web
		spec:
			containers:
				- name: web
					image: nginx:1.25
					readinessProbe:
						httpGet:
							path: /
							port: 80
						initialDelaySeconds: 3
						periodSeconds: 5
					livenessProbe:
						httpGet:
							path: /
							port: 80
						initialDelaySeconds: 10
						periodSeconds: 10
```

3) Service：稳定访问入口，ClusterIP/NodePort/LoadBalancer。

```yaml
apiVersion: v1
kind: Service
metadata:
	name: web-svc
spec:
	type: ClusterIP
	selector:
		app: web
	ports:
		- name: http
			port: 80
			targetPort: 80
```

4) Ingress：HTTP 入口（或用 Gateway API）。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: web-ing
	annotations:
		nginx.ingress.kubernetes.io/rewrite-target: /
spec:
	ingressClassName: nginx
	rules:
		- host: web.local
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: web-svc
								port:
									number: 80
```

5) ConfigMap/Secret：配置与密钥管理（建议结合 External Secrets/Sealed Secrets）。

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: db-secret
type: Opaque
data:
	username: YWRtaW4=
	password: c2VjcmV0
```

---

### 18. 命令速查与常见操作

- 集群与命名空间：
	- `kubectl config get-contexts; kubectl config use-context X`
	- `kubectl get ns; kubectl create ns dev`
- 资源查看与过滤：
	- `kubectl get po -A`
	- `kubectl get deploy -n dev -o wide`
	- `kubectl get svc -l app=web -o yaml`
- 变更与灰度：
	- `kubectl rollout status deploy/web-deploy`
	- `kubectl rollout restart deploy/web-deploy`
	- `kubectl set image deploy/web-deploy web=nginx:1.26 --record`
- 调试与排障：
	- `kubectl logs deploy/web-deploy -f`
	- `kubectl exec -it deploy/web-deploy -- sh`
	- `kubectl describe po/<pod>`
- 资源编辑：
	- `kubectl apply -f file.yaml`
	- `kubectl diff -f file.yaml`
	- `kubectl delete -f file.yaml`

---

### 19. 故障排查与高频问题

1) Pod CrashLoopBackOff：
- 检查容器日志、探针配置、资源限制是否过小、启动命令是否正确。

2) Readiness 未就绪导致流量 503：
- 调整 `readinessProbe`；确保依赖（DB/Cache）连通；应用启动完成后再对外暴露。

3) 节点资源不足/驱逐（Evicted）：
- 查看 `kubectl top node/pod`；设置 Resource Requests/Limits；为关键服务设定 `PriorityClass` 与 `PodDisruptionBudget`。

4) Service 无法访问：
- Label 不匹配、targetPort 错误、NetworkPolicy 阻断。

5) Ingress 404/证书问题：
- 域名解析、TLS Secret、路径匹配规则；推荐迁移到 Gateway API 简化配置。

6) 存储挂载失败：
- PVC 绑定状态、StorageClass 是否支持动态供给；检查访问模式（RWO/RWX）。

---

### 20. 面试高频问答（精炼版）

Q: Kubernetes 的核心控制循环是什么？
A: Controller 通过 Watch 资源变化，触发 Reconcile，将实际状态驱动向期望状态（声明式）。

Q: Deployment 与 StatefulSet 区别？
A: Deployment 用于无状态服务，Pod 可任意替换；StatefulSet 保持稳定身份与有序滚动，常用于带持久化的服务。

Q: Service 类型怎么选？
A: 集群内用 ClusterIP；需要外部访问 NodePort/LoadBalancer；云上常用 LoadBalancer 结合 Ingress/Gateway API。

Q: 为什么需要探针？
A: Liveness 保证容器自愈（重启）；Readiness 控制流量切入，避免未就绪时被负载均衡转发。

Q: HPA 如何工作？
A: 基于资源指标（如 CPU/自定义指标）动态调节副本数，需 Metrics Server/Prometheus Adapter 提供指标。

Q: 什么情况下引入 Service Mesh？
A: 服务数量多、治理需求强（灰度、熔断、mTLS、可观测）、跨集群路由；小规模可省略以降低复杂度。

Q: Operator 的价值？
A: 把复杂系统的运维知识封装为声明式 API，自动化生命周期管理，提升一致性与可靠性。

Q: 如何保障多租户与安全？
A: RBAC、NetworkPolicy、PodSecurity（或 PSP 替代方案）、命名空间隔离、OPA/Gatekeeper/kyverno 策略治理、镜像签名与漏洞扫描。

Q: GitOps 与传统流水线区别？
A: Git 为唯一真实来源，控制器拉取并对齐集群状态；更易审计回滚，适合多环境与多集群。

这些内容覆盖从基础到进阶与面试高频考点，建议配合实际动手部署与问题模拟巩固记忆。

---

### 21. 容器运行时与镜像仓库

- 运行时演进：Docker -> containerd（主流）/CRI-O；Kubelet 通过 CRI 与运行时解耦。
- 镜像优化：多阶段构建、瘦身基础镜像（alpine/distroless）、分层去重、避免 `latest`。
- 安全与合规：镜像签名（Cosign/Notary v2）、漏洞扫描（Trivy/Grype）、私有仓库（Harbor/ACR/ECR/GCR）。
- 拉取策略：`imagePullPolicy` 与拉取密钥（ImagePullSecret）；结合缓存与预拉取减少冷启动。

---

### 22. 调度与亲和/反亲和、污点与容忍

- NodeSelector/NodeAffinity/PodAffinity/AntiAffinity：按标签与拓扑进行约束与分布（同机/跨机架）。
- Taints/Tolerations：节点打污点，只有具备容忍的 Pod 可调度上去，常用于系统/专用节点。
- TopologySpreadConstraints：控制 Pod 在不同拓扑域的均匀分布，避免「热点节点」。
- PriorityClass：为关键服务设置更高优先级，资源紧张时优先保留。
- PodDisruptionBudget（PDB）：限制同时被驱逐的 Pod 数，保障服务可用性。
- 常见面试题：如何确保高可用与就近访问？答：亲和 + AntiAffinity + Spread + PDB + 多副本。

---

### 23. 安全与策略治理（PodSecurity/OPA/kyverno）

- Pod Security：以标准级别（privileged/baseline/restricted）控制容器能力，替代 PSP。
- OPA/Gatekeeper：以 Rego 编写策略，准入时校验资源是否合规（如必须设置资源限制）。
- kyverno：更易读的策略引擎（YAML 风格），支持校验、变更、生成等动作。
- 运行时安全：Falco/Cilium Tetragon 进行行为检测与告警；配合 audit 日志与 SIEM。
- Secrets 管理：KMS/KeyVault/Secrets Manager；Sealed Secrets/External Secrets 将密钥拉取或加密存储。
- 网络与零信任：NetworkPolicy/CiliumNetworkPolicy，结合 mTLS 与证书轮换实现端到端安全。

---

### 24. 模板化与配置管理（Helm/Kustomize）

- Helm：Chart 包管理，支持 values 参数化、依赖、Hooks；用于应用打包与交付。
- Kustomize：基于资源的覆盖与组合（patch/overlay），不需要模板语法适合 GitOps。
- 实战建议：
	- 小型项目用 Kustomize 简单覆盖；复杂应用/生态用 Helm，配合 Helmfile 统一编排。
	- 避免在模板里写复杂逻辑，保持 values/overlay 可读；通过 CI 做 `helm lint`/`kubeconform` 校验。

---

### 25. 可观测性：指标、日志、链路追踪

- 指标（Metrics）：Prometheus + Grafana；多集群用 Thanos/Loki 远程读写与聚合。
- 日志（Logs）：EFK/ELK（Elastic/Fluentd/Kibana）或 Loki + Promtail；统一日志结构与采集规则。
- 追踪（Traces）：OpenTelemetry + Tempo/Jaeger；Mesh 下可直接采集 Sidecar 指标与追踪。
- 报警（Alerting）：Alertmanager，结合服务级 SLO/SLA、Error Budget；落地告警分级与值班轮值。
- 性能分析：eBPF（Cilium/Hubble、Pixie）进行细粒度流量与系统调用观测。

---

### 26. 备份与容灾（Backup/DR）

- Velero：备份 Kubernetes 资源与 PV 数据到对象存储，支持跨集群恢复与迁移。
- 灾备策略：同城双活/两地三中心、多区域多集群，结合联邦与全局流量管理。
- 数据一致性：数据库需自身主备/复制机制；K8s 层仅负责卷与配置的备份与恢复流程。

---

### 27. 性能优化与成本控制

- 资源与自动化：Requests/Limits 合理设置；HPA/VPA/Cluster Autoscaler 组合弹性。
- 节点类型：针对 CPU/内存/IO 优化的实例；抢占式/Spot 节省成本但结合 PDB 与优雅降级。
- 镜像与启动：减小镜像；开启镜像缓存；尽量少 init 容器耗时；使用 Readiness 控制切流。
- 网络与存储：选择合适 CNI/CSI；性能敏感用本地盘或高 IOPS 云盘；避免跨区通信。
- 观测与优化闭环：Profiling（应用/系统）、监控面板与持续调参。

---

### 28. 控制面内部原理与 API

- API Server：统一入口，认证/鉴权（RBAC）、准入控制（Mutating/Validating）、etcd 存储。
- Controller Manager/Scheduler：控制循环与调度策略（评分/优选）；Informer 缓存与事件。
- etcd：强一致 KV 存储，注意备份与性能（压缩、碎片整理、节点大小限制）。
- 扩展机制：CRD/Admission Webhook/Aggregated API；Operator 以 Reconcile 实现领域逻辑。
- 考点提示：解释一次 `kubectl apply` 的完整路径：客户端 -> APIServer（认证/授权/准入）-> etcd 落盘 -> Controller 观察到变更 -> 创建/更新底层对象 -> Kubelet 拉起容器。

---

### 29. Go 服务在 K8s 的工程化实践

- 镜像与构建：Go 静态编译，使用多阶段构建，最终镜像推荐 `distroless` 或 `alpine`，减小攻击面。
- 健康检查：在 Go 内暴露 `/healthz`（liveness）与 `/readyz`（readiness），探针与服务启动逻辑解耦。
- 配置管理：`env` + ConfigMap + Secret；优先 12-Factor，避免二进制内置配置。
- 资源与并发：估算 Go 服务的 `GOMAXPROCS`、连接池与队列长度；结合 Requests/Limits、防止 CPU 抢占造成的尾延时。
- 可观测性：内置 Prometheus 指标（`promhttp`），日志使用结构化（Zap/zerolog），追踪使用 OpenTelemetry。
- 弹性与流控：结合 Mesh 或在应用层支持超时、重试、熔断、限流（令牌桶/漏桶），避免级联故障。

---

### 30. API 网关与 Gateway API（面向 Web 后端）

- 能力对比：传统 Nginx Ingress vs Gateway API（更强的可扩展与多租户）。
- 典型需求：路径/主机路由、金丝雀权重、Header/Cookie 路由、限速、WAF、认证（OAuth2/JWT）。
- 实现选择：
	- 云网关（APIM/ALB/NLB/Cloud Armor）+ K8s 网关。
	- 开源方案：Kong/Gloo/Traefik/Envoy Gateway；与 Mesh 协同做南北向/东西向统一治理。
- 面试要点：如何在 Gateway API 下实现金丝雀？答：通过 `HTTPRoute` 后端权重与匹配规则，并结合 `BackendPolicy` 或实现自定义策略。

---

### 31. 服务治理与 Mesh 策略（与 Go 应用协同）

- 灰度与 AB：按权重/标签/用户分组路由，镜像流量用于压测与回归。
- mTLS 与零信任：东西向默认加密，证书轮换自动化；Go 客户端可选禁用双向直连，交给 Sidecar。
- 熔断/重试/超时：统一由 Mesh 管理，Go 端保留合理重试与超时，避免双重重试放大故障。
- 可观测性：Sidecar 采集请求指标与追踪，结合 Go 应用指标做端到端关联。
- 多集群：通过 Mesh 多集群网关与服务导出，实现跨域流量与就近访问。

---

### 32. CI/CD 与 GitOps（适配 Go 后端）

- 流程建议：
	- CI：lint（golangci-lint）+ 测试（`go test`）+ 安全扫描（gosec/trivy）+ 构建镜像（BuildKit）+ 推送仓库。
	- CD：更新 Helm/Kustomize 的版本与镜像标签；GitOps 控制器（Argo CD/Flux）自动对齐集群。
- 策略：分支保护、语义化版本、自动回滚、漂移检测；环境分层（dev/stage/prod）。
- 供应链安全：SBOM（Syft）、签名与验证（Cosign）、策略准入（kyverno/OPA）。
- 面试要点：如何保障“宣告成功但实际未生效”的问题？答：GitOps + HealthCheck + Progressive Delivery（Argo Rollouts/Flagger）。

---

### 33. 观测与性能（Go 专项）

- 指标：请求延时/吞吐、错误率、依赖外调用时长、GC 停顿（`go_memstats_*`）、goroutine 数、连接池指标。
- 日志：结构化 + 关联 ID（trace_id/span_id/request_id），限流与采样策略避免日志风暴。
- 追踪：OpenTelemetry SDK（Go），与 Mesh/网关追踪打通，端到端可视化。
- Profiling：`pprof`（CPU/Heap/Block/Mutex），线上只在受控窗口开启并做好限流与采样；结合 eBPF 做系统级分析。
- 性能优化：减少内存分配热点、避免全局锁、批量化 IO、合理超时与重试、控制 goroutine 扇出与背压。

---

### 34. 数据库与有状态服务（Web 后端关注）

- 落地原则：数据库主从/集群应由自身组件保证一致性与高可用；K8s 负责编排与故障自愈，但不替代数据库一致性机制。
- StatefulSet + Headless Service：稳定身份与服务发现；PVC 与 StorageClass 动态供给，选择合适盘型与拓扑。
- 连接管理：服务端连接池、迁移时连接耗尽策略、优雅停机（`preStop` + readiness 切流）。
- 备份与迁移：Velero + operator 自带备份；跨集群/跨区域迁移的 DNS 与数据同步策略。
- 常见坑：RWX 访问模式误用、PV 绑定拓扑不一致、滚动升级打断复制、网络策略阻断心跳。

---

### 35. 平台工程与 SRE 职责矩阵（面向企业落地）

平台工程的目标是提供“可复用、可自助、可治理”的内部开发者平台（IDP），SRE 的目标是以工程化方式保障可靠性与效率。典型职责矩阵如下：

- 平台能力（Platform Engineering）
	- 统一交付：应用模板（Helm/Kustomize）、GitOps、环境分层与租户管理。
	- 基础设施抽象：网络/存储/安全策略的标准化与自助接口（Portal/API）。
	- 策略治理：RBAC、PodSecurity、OPA/kyverno 策略库与准入控制；镜像签名与供应链安全。
	- 可观测性即服务：集中 Prometheus/Grafana/Loki/Tempo，标准化指标与日志/追踪接入。
	- 成本与资源：配额/限额、成本看板、自动化弹性（HPA/VPA/CA），Spot/预留策略与预算告警。

- SRE（Site Reliability Engineering）
	- SLO/SLI/错误预算：定义并监控服务级目标，基于错误预算管理变更节奏。
	- 变更与发布：渐进式交付（Argo Rollouts/Flagger）、自动回滚、发布冻结与审批流。
	- 事件管理：值班/告警分级、Runbook 与演练、事后复盘与行动项闭环（Blameless）。
	- 容量与韧性：压测、容量规划、故障注入与演练（Chaos Mesh）、跨区/多集群容灾。
	- 安全与合规：运行时安全监控、审计日志、策略合规与渗透测试协同。

?> 面试建议：结合你的经历，描述“平台标准化 + GitOps + 策略治理 + SLO 驱动的发布流程”，展示从工具到机制的闭环。

---

### 36. 渐进式发布（Argo Rollouts / Flagger）示例

目标：在不引入 Service Mesh 的前提下，实现金丝雀与蓝绿发布，并带自动回滚能力；有 Mesh 时可更细粒度路由与指标来源。

1) Argo Rollouts 金丝雀（按权重推进 + 自动回滚）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
	name: web-rollout
spec:
	replicas: 4
	strategy:
		canary:
			canaryService: web-canary
			stableService: web-stable
			steps:
				- setWeight: 20
				- pause: {duration: 60}
				- setWeight: 50
				- pause: {duration: 120}
				- setWeight: 100
			analysis:
				templates:
					- name: error-rate-check
						prometheus:
							address: http://prometheus.default.svc.cluster.local
							query: |
								sum(rate(http_requests_total{job="web",status=~"5.."}[1m]))
								/
								sum(rate(http_requests_total{job="web"}[1m]))
							threshold: 0.02
				startingStep: 1
	selector:
		matchLabels:
			app: web
	template:
		metadata:
			labels:
				app: web
		spec:
			containers:
				- name: web
					image: nginx:1.26
					ports:
						- containerPort: 80
```

说明：
- `stableService`/`canaryService` 需预先创建并指向同一 Rollout 的不同版本权重；
- `analysis` 绑定 Prometheus 指标做自动回滚判断（错误率阈值）。

2) Flagger 金丝雀（依托 Service/Ingress 或 Mesh）

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
	name: web
spec:
	provider: kubernetes
	targetRef:
		apiVersion: apps/v1
		kind: Deployment
		name: web-deploy
	service:
		port: 80
	analysis:
		interval: 1m
		threshold: 5
		maxWeight: 100
		stepWeight: 20
		metrics:
			- name: error-rate
				interval: 30s
				threshold: 1
				templateRef:
					name: error-rate
		alerts:
			- name: on-call
				severity: high
```

说明：
- `provider` 可为 kubernetes/istio/appmesh/nginx 等；
- 通过 `metrics` 模板与告警绑定实现推进与回滚；`threshold` 控制失败次数阈值。

3) 蓝绿发布（Argo Rollouts）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
	name: web-bluegreen
spec:
	replicas: 3
	strategy:
		blueGreen:
			activeService: web-active
			previewService: web-preview
			autoPromotionEnabled: false
			scaleDownDelaySeconds: 60
	selector:
		matchLabels:
			app: web
	template:
		metadata:
			labels:
				app: web
		spec:
			containers:
				- name: web
					image: nginx:1.26
```

实践建议：
- 将渐进式发布与 SLO/错误预算绑定，自动化回滚与审批策略；
- 指标来源统一用 Prometheus/Otel，避免“双重重试/双重采样”的干扰；
- 在网关/Gateway API 层协同 Header/Cookie 路由，支持精准金丝雀与 A/B。




