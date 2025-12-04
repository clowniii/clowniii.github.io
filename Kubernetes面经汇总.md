# Kubernetes 面经汇总（问答版）

> 面向 Web 资深后端（Go）+ 云原生开发 + DevOps/SRE 的系统化题库。建议结合你的项目经历作答，突出“原理 + 实践 + 取舍”。

## 一、基础原理与架构

问：Kubernetes 的核心思想是什么？
答：声明式期望状态（Desired State），控制循环（Watch/Reconcile）驱动实际状态（Actual State）向期望对齐。API Server 作为统一入口，etcd 存储状态，Controller/Scheduler 负责控制与调度，Kubelet 在节点上执行容器生命周期。

问：一次 `kubectl apply` 背后发生了什么？
答：客户端提交资源到 API Server → 认证/授权（RBAC）→ 准入控制（Mutating/Validating）→ etcd 持久化 → 控制器通过 Informer 感知变化执行 Reconcile（创建/更新底层对象）→ Scheduler 为待调度 Pod 绑定节点 → Kubelet 拉取镜像、创建容器 → 探针就绪后 Service/Ingress/Gateway 对其转发流量。

——

## 速记卡片：Kubernetes 面试高频知识点

- 原理：声明式、控制循环、API Server/etcd/Controller/Scheduler/Kubelet 分工
- 编排：Pod/Deployment/StatefulSet/DaemonSet/Job/CronJob 区别与场景
- 网络：CNI 选型、Service/Ingress/Gateway API、金丝雀/A-B 路由
- 存储：PVC/PV/StorageClass/CSI、StatefulSet/Headless Service、RWX/RWO
- 安全：RBAC、PodSecurity、NetworkPolicy、OPA/kyverno、Secrets/KMS
- 观测：Prometheus/Grafana、ELK/Loki、OTel/Jaeger、SLO/SLI/Runbook
- 交付：GitOps、Helm/Kustomize、Argo CD/Flagger、渐进式发布
- SRE：错误预算、自动回滚、事件管理、容量与韧性、平台工程能力矩阵
- Go 加分：多阶段构建、健康探针、Prom/OTel、pprof、背压与熔断

## 情景题模板：Kubernetes 面试实战答题

### 1. 线上 5xx 突增排查
问：如果你遇到线上 5xx 错误率突增，如何排查？
答：
1. 网关/入口层：检查 Ingress/Gateway/Service，是否有流量异常、路由变更、证书或限流问题。
2. 应用层：查看 Pod 日志与事件，是否 CrashLoop、资源耗尽、探针未就绪、依赖超时。
3. 下游依赖：数据库/缓存/第三方接口，连接池耗尽、慢查询、网络策略阻断。
4. 资源与策略：节点资源、PDB、优先级、污点/容忍，是否有驱逐或调度异常。
5. 策略与安全：OPA/kyverno 是否有新策略生效，Secrets/配置变更。
6. 观测与报警：Prometheus 指标、日志追踪、SLO/SLI 告警，定位异常时间段与影响面。
7. 回滚与缓解：如有渐进式发布，触发自动回滚或手动回退，通知 SRE/平台团队协同。

### 2. 设计跨集群就近路由 + 金丝雀 + 自动回滚体系
问：如何设计一个支持跨集群就近路由、金丝雀发布与自动回滚的发布体系？
答：
1. 多集群管控：用 Rancher/ACM/Karmada 集中管理，统一命名/标签规范。
2. Gateway API 层：按 Region/Zone/权重/Header 路由，实现就近访问与金丝雀流量分配。
3. 渐进式发布：Argo Rollouts/Flagger，结合 Prometheus/Otel 指标做自动推进与回滚。
4. SLO/SLI 驱动：定义关键指标（延时、错误率），与发布流程绑定，错误预算不足自动冻结。
5. 策略治理：OPA/kyverno 校验配置与安全，避免不合规变更上线。
6. 观测与复盘：统一指标/日志/追踪，事后复盘与行动项闭环。

### 3. OPA/kyverno 策略样例速查
问：常见 OPA/kyverno 策略有哪些？
答：
- 强制资源限制：禁止无 requests/limits 的 Pod 创建
- 禁止特权容器：拒绝 `privileged: true` 的 Pod
- 镜像签名校验：只允许签名镜像拉取
- 命名规范：强制命名空间/资源名符合正则
- 自动注入标签/注释：如 owner、team、env
- 限制 Service 类型：禁止 NodePort/LoadBalancer 非审批创建
- 强制探针：必须设置 liveness/readiness

（可在面试时举例 YAML 或 Rego 片段，展示实际落地能力）

问：Deployment、StatefulSet、DaemonSet 区别？
答：Deployment 管理无状态副本与滚动升级；StatefulSet 保持稳定身份、有序滚动，常配合 PVC 用于有状态服务；DaemonSet 在每个（或匹配的）节点上都运行一个 Pod，适合日志采集/监控代理。

## 二、核心对象与编排

问：Pod 是什么？为什么是最小调度单位？
答：Pod 是一组紧耦合容器的集合，共享网络命名空间与存储卷。调度、探针、自愈等以 Pod 为粒度，容器间通过 localhost 通信，便于组合 Sidecar 等模式。

问：Service 的类型与使用场景？
答：ClusterIP（集群内访问），NodePort（暴露到每个节点固定端口），LoadBalancer（云上负载均衡器），Headless（无 ClusterIP，用于服务发现与有状态集群）。按需求选择，通常外部访问用 LoadBalancer+Ingress/Gateway。

问：如何设计健康检查？
答：Liveness 保证容器自愈，Readiness 控制流量切入（未就绪不转发），Startup 校验启动完成再启用其他探针。探针逻辑需与应用启动/依赖准备解耦，避免误杀与雪崩。

问：滚动升级如何确保零停机？
答：合理 `maxUnavailable/maxSurge`，使用 readiness 探针与优雅停止（preStop + terminationGracePeriod），结合 PDB 限制同时驱逐数，必要时灰度（金丝雀）或蓝绿。

## 三、调度与高可用

问：亲和/反亲和、拓扑分散如何保障高可用？
答：NodeAffinity/PodAffinity/AntiAffinity 控制同机/跨机分布；TopologySpreadConstraints 在不同区域/机架均匀分散，避免单点热点；结合多副本与 PDB 保证升级/故障期间可用性。

问：Taints/Tolerations 的用途？
答：对节点打污点，仅容忍（Tolerations）的 Pod 可调度上去。常用于系统、GPU 或专用节点的隔离。

问：PriorityClass 有什么作用？
答：为关键服务设置更高优先级，资源紧张或驱逐时优先保留，避免业务被低优先级作业挤压。

## 四、网络与入口

问：CNI 插件如何选择？
答：Flannel（简单）、Calico（BGP/NetworkPolicy）、Cilium（eBPF、L3-L7 可观测/安全）。根据规模、网络策略与性能需求取舍。

问：Ingress 与 Gateway API 的区别？
答：Ingress 是早期标准，能力多依赖实现的 Annotation；Gateway API 提供 `GatewayClass/HTTPRoute` 等更规范的资源与扩展性，支持多租户与更丰富路由策略，适合新项目。

问：如何实现金丝雀与 A/B？
答：在 Gateway API/Ingress 层按权重/Header/Cookie 路由；或用 Argo Rollouts/Flagger 做渐进式发布，与 Prometheus 指标绑定自动回滚；Mesh 场景可在东西向实现更细粒度流量治理。

## 五、存储与有状态服务

问：PVC、PV、StorageClass 的关系？
答：用户声明 PVC，动态供给器根据 StorageClass 创建 PV 并绑定；PVC 指定访问模式（RWO/RWX）与资源大小。选择匹配的 CSI 插件与盘型，关注拓扑（Zone/Rack）与性能。

问：StatefulSet 常见实践与坑？
答：配合 Headless Service 稳定身份，PVC 动态供给；滚动升级需考虑数据库复制与连接耗尽；注意 RWX/RWO 模式、PV 拓扑一致性与网络策略对心跳/复制的影响。

## 六、安全与策略治理

问：如何实现多租户与安全合规？
答：RBAC 控制权限，命名空间隔离；PodSecurity（baseline/restricted）限制容器能力；NetworkPolicy/CiliumNetworkPolicy 做零信任隔离；OPA/Gatekeeper/kyverno 在准入阶段校验与修正策略；Secrets 使用 KMS/Sealed/External Secrets 管理。

问：镜像供应链安全要点？
答：SBOM（Syft）、漏洞扫描（Trivy/Grype）、签名与验证（Cosign/Notary v2）、私有仓库与最小镜像（distroless），禁用 `latest` 与不可信源。

## 七、可观测性与故障排查

问：如何搭建统一观测？
答：指标（Prometheus + Grafana）、日志（ELK/Loki + Promtail）、追踪（OpenTelemetry + Jaeger/Tempo）。统一接入规范，定义 SLO/SLA 与报警分级，建立 Runbook。

问：排查 CrashLoopBackOff 的步骤？
答：查看容器日志与 `describe` 事件，检查探针/启动命令/资源限制，本地复现实验，必要时开启临时 `command` 与更高日志等级定位；避免探针触发重启风暴。

问：Readiness 长期未就绪怎么办？
答：确认依赖（DB/Cache）连通、应用初始化完成后才就绪；调优探针阈值与 `initialDelaySeconds`；在网关层避免未就绪分配流量。

## 八、交付与 GitOps

问：为什么选择 GitOps？
答：以 Git 为唯一事实来源，控制器拉模式对齐集群状态，具备可追溯、可回滚与多环境推广（Promotion）的优势；结合策略校验避免不合规配置。

问：Helm 与 Kustomize 取舍？
答：Helm 适合复杂生态与打包发布，支持依赖与 hooks；Kustomize 更轻量，适合 GitOps 与基于资源的覆盖。大规模场景可用 Helmfile 或分层 overlay 管理。

## 九、Operator 与 CRD

问：为什么需要 Operator？
答：将复杂系统（数据库、中间件、批任务）的运维知识封装为 CRD + Controller 的声明式 API，自动化生命周期管理（部署、扩容、备份、故障自愈）。关键是幂等性与状态机设计，精确 RBAC 与观测可视。

## 十、SRE 与平台工程

问：SLO/SLI/错误预算如何落地？
答：定义关键用户体验指标（延时、错误率等），以 SLO 为目标监控 SLI，错误预算驱动变更节奏（预算不足时减缓发布，强化质量工作），与发布策略/回滚/审批联动。

问：平台工程的目标与能力？
答：提供可复用、可自助、可治理的内部平台：统一交付（模板、GitOps）、策略治理（安全与合规）、统一观测与成本管理（配额/预算/弹性）；抽象基础设施细节，提升开发与运维效率。

## 十一、渐进式发布（金丝雀/蓝绿/自动回滚）

问：如何用 Argo Rollouts 做金丝雀并自动回滚？
答：通过 Rollout 的 `steps` 设置权重推进与 `analysis` 绑定 Prometheus 指标（如错误率/延时阈值），在达到阈值时自动回滚；结合 `stableService/canaryService` 分流稳定与金丝雀版本。

问：Flagger 的工作方式？
答：持续观察目标 Deployment/Service 的版本变更，按 `stepWeight` 推进权重，结合指标模板与告警进行回滚/通知；支持 Kubernetes/Ingress/Mesh 多种 provider。

问：蓝绿的优缺点？
答：优点是切换简单、回滚迅速；缺点是资源占用较高、数据库迁移需精心设计。可通过 `activeService/previewService` 控制切换，配合数据层双写或迁移窗口。

## 十二、多集群与混合云

问：多集群的常见策略？
答：集中管控（Rancher/ACM/Karmada）、Mesh 跨集群路由与统一身份、全局负载均衡（GSLB）、联邦同步、云边协同（KubeEdge/OpenYurt）。关键是统一命名/标签规范、网络/DNS/证书一致性与集中观测。

## 十三、性能与成本

问：如何在保证性能的同时优化成本？
答：合理设置 Requests/Limits 与自动化弹性（HPA/VPA/CA），选择合适实例与盘型；对高并发服务优化镜像与启动路径、使用本地缓存与就近访问；利用 Spot/预留但与 PDB/优雅降级配合；持续观测与调参闭环。

## 附：Go 后端在 K8s 的加分点

问：Go 服务的容器化与可观测最佳实践？
答：使用多阶段构建与瘦身镜像（distroless），暴露 `/healthz` `/readyz`，内置 Prometheus 指标与 OTel 追踪，结构化日志携带 trace/request ID；控制 goroutine 扇出与背压，合理超时与重试，避免级联故障；结合 Mesh 统一治理并保持应用端最小但必要的保护。

——

使用建议：
- 面试前按章节做“要点背诵 + 项目案例”准备，每题至少关联一个你做过的实践与一次线上故障复盘。
- 碰到开放题，按“原理 → 方案 → 取舍 → 风险与治理 → 指标与回滚”结构作答，展现体系化思考。
