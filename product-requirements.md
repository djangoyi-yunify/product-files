# K8s 原生通用 PaaS 中间件管理平台

## 项目概述

基于 K8s 构建的通用、可扩展、轻量级 PaaS 中间件管理平台，支持通过统一的 Controller 管理多种中间件的生命周期、备份、配置、监控等功能。平台采用插件化设计，新中间件只需通过插件注册配置注册即可接入。

## 第一期范围

| 模块 | 范围 |
|------|------|
| **支持中间件** | PostgreSQL、Redis |
| **生命周期管理** | 11 个核心操作（Create/Delete/Scale/Stop/Start/Restart/Failover/Upgrade 等） |
| **状态管理** | 状态 Agent 容器采集并上报到 Annotation，Controller 同步到 CRD Status |
| **备份恢复** | 单集群内恢复（不支持跨集群） |
| **监控** | 对接 Prometheus，监控 Agent 暴露 Metrics（第一期暂不实现） |
| **TLS 支持** | 集成 CertManager 自动生成证书，Secret 方式挂载 |
| **多进程容器** | 进程管理器 + gRPC 服务，Restart 不重启 Pod |
| **操作 Agent** | 通过 gRPC 接收 Controller 调用，执行特定中间件的配置管理和运维操作（第一期实现框架） |
| **统一操作入口** | Operation CRD 作为所有操作的统一入口和操作日志记录 |
| **Web UI** | 暂不实现 |
| **插件机制** | 插件注册配置格式：ConfigMap 存储，YAML 声明中间件定义（Component、Pod模板、参数模板、操作模板），CUE 用于配置验证和约束 |

---

## 用户需求

### 用户需求 1：统一 Controller 架构

#### 需求详情
- 采用统一的 Controller 架构，基于 Operator SDK 框架开发
- Controller 负责中间件的生命周期管理、资源编排、状态同步
- 通过插件注册配置（ConfigMap + YAML/CUE）注册新中间件插件，无需重新编译 Controller
- Controller 本身支持高可用（Deployment 多副本部署 + Leader Election）

#### 技术洞察
- **架构模式**：K8s Controller 模式，基于 **Operator SDK** 框架开发
- **扩展方式**：ConfigMap 插件注册配置，支持热更新和重启生效两种更新模式
- **优势**：利用 Operator SDK 的代码生成、测试框架、生命周期管理等能力，提高开发效率
- **核心接口**：CRD Schema 定义、资源渲染、操作执行、状态采集、备份恢复、指标暴露
- **优势**：架构简单、代码耦合度低、学习成本低
- **参考项目**：Kubernetes sample-controller、Prometheus Operator（简化版）

---

### 用户需求 2：通用中间件支持

#### 需求详情
- 平台通用设计，不限定具体中间件类型
- 第一期支持 **PostgreSQL** 和 **Redis**
- 支持任意中间件的所有部署模式（如 Redis 单机/主从/Sentinel/Cluster、PostgreSQL 单机/主从）
- 支持同一中间件的多个版本（如 Redis 6.x、7.x）
- 新中间件（甚至未来出现的中间件）只需**编写插件注册配置**即可接入，无需修改 Controller 代码

#### 技术洞察
- **多层级 CRD 设计**：采用三层 CRD 实现不同层级的抽象和职责分离

| CRD 类型 | 说明 | 职责 |
|----------|------|------|
| **Application**（顶层） | 用户创建 PaaS 实例的入口 | 维护实例元数据，自动创建 Component 实例 |
| **Component**（中间层） | 由 Application 自动创建 | 管理 Component 的生命周期，创建 WorkloadSet 实例 |
| **WorkloadSet**（底层） | 由 Component 自动创建 | 管理一组工作负载（如 Pod），实现 Pod 的创建、更新、删除 |

- **Component 与 WorkloadSet 关系**：
  - Component 和 WorkloadSet 是 **一对多关系**（N >= 1）
  - 每个 WorkloadSet 管理一组 Pod，这些 Pod 共享相同的 Resource 配置和中间件配置
  - 一个 Component 可以包含多个 WorkloadSet，满足复杂部署场景

- **Component 与 WorkloadSet 关系示例**：

  **场景 1：Redis Cluster 分片**
  - Redis Cluster 由 3 个分片组成，每个分片包含 1 主 1 从
  - Component（如 `redis-shard`）包含 3 个 WorkloadSet
  - 每个 WorkloadSet 管理 2 个 Pod（1 主 1 从）

  **场景 2：Elasticsearch 集群**
  - ES 集群包含多个角色（master、data、ingest 等）
  - 角色为 master 的节点放在同一个 WorkloadSet
  - 角色为 data 的节点放在另一个 WorkloadSet
  - 它们都属于同一个 Component（因为 ES 节点角色可变，无法在注册配置中事先定义）

- **配置方式**：通过插件注册配置声明中间件的 Application 定义、Component 定义、参数模板、操作逻辑
  - 每种中间件定义由一个 Application 构成
  - Application 由一个或多个 Component 构成（如 Kafka = broker + kraft，Redis = redis-node，Redis + Sentinel = redis-node + sentinel）
  - Component 包含了 Pod 模板、中间件参数、生命周期处理等相关定义
- **版本适配**：插件注册配置中定义版本与配置模板的映射关系
- **最佳实践**
  - 参考 kubeblocks 的 CRD 思路
  - 参考 Kubernetes 原生 Controller 模式，实现层级化的状态同步和 Reconcile 逻辑

---

### 用户需求 3：中间件生命周期管理

#### 需求详情
支持以下 12 个核心生命周期操作：

| 操作 | 说明 | 执行方式 |
|------|------|----------|
| **Create** | 创建中间件实例 | 用户请求触发 |
| **Delete** | 删除中间件实例及关联资源 | 用户请求触发 |
| **Vertical-Scale-Resource** | 调整 CPU/内存配置 | 用户请求触发 |
| **Vertical-Scale-Volume** | 调整存储容量 | 用户请求触发 |
| **Horizontal-Scale** | 调整副本数/分片数 | 用户请求触发 |
| **Stop** | 停止中间件服务 | 用户请求触发 |
| **Start** | 启动中间件服务 | 用户请求触发 |
| **Restart** | 重启中间件服务（通过多进程容器实现，不重启 Pod） | 用户请求触发 |
| **Reload** | 重载配置（不停机） | 用户请求触发 |
| **Failover** | 故障转移（主从切换） | 全自动或用户请求 |
| **UpgradeApp** | 升级注册配置版本（注册配置变更，如新增中间件、修改操作逻辑） | 用户请求触发 |
| **UpgradePaaS** | 升级中间件版本（如 Redis 7.2 → 7.4，PostgreSQL 14 → 15） | 用户请求触发 |

- 生命周期操作支持**全自动执行**和**用户请求触发**两种方式
- Controller 根据 CRD 状态和实际状态自动判断是否需要执行操作

#### 操作执行机制
- **声明式配置**：插件注册配置定义每种操作的步骤和命令模板
- **参数填充**：Controller 执行时使用实际的参数值替换模板中的占位符
- **异步执行**：基于 K8s Workqueue 实现异步操作队列
- **失败处理**：操作失败时记录 Condition，支持重试

#### 状态管理
- 中间件实际状态通过 Pod 中的 **Agent 容器**采集
- Agent 容器将状态上报到 Pod 的 Annotation
- Controller 从 Annotation 读取状态，更新到 CRD Status

#### Agent 容器设计
每种中间件 Pod 中包含以下 Agent 容器（可选/必选）：

| Agent 类型 | 功能 | 必选/可选 | 第一期范围 |
|------------|------|----------|----------|
| **状态 Agent** | 中间件节点的主从状态、角色探测和更新、节点健康状态检测 | **必选** | 实现 |
| **操作 Agent** | 特定中间件的配置管理、运维操作（由注册配置声明具体操作） | 可选 | 实现框架 |
| **日志 Agent** | 日志轮转和导出 | 可选 | 暂不实现 |
| **监控 Agent** | 暴露 Prometheus Metrics 端点 | 可选 | 暂不实现 |

- **状态 Agent**：
  - 采集中间件状态（Redis Info、PostgreSQL Status 等）
  - 执行 Status Check（角色判断、主从状态探测、节点健康检测）
  - 通过本地命令获取状态（如 `redis-cli role`、`pg_isready` 等）
  - 上报状态到 Pod Annotation，供 Controller 同步到 CRD Status
- **操作 Agent**：通过 gRPC 接收 Controller 调用，执行特定中间件的配置管理和运维操作（如权限配置、切换操作等），具体操作在注册配置中声明
- **日志 Agent**：负责中间件日志的轮转和导出到日志系统
- **监控 Agent**：采集并暴露中间件业务指标（QPS、延迟、连接数等）

#### 技术洞察
- **Agent 容器**：轻量级 Sidecar，与中间件容器同 Pod
- **状态模型**：K8s 标准 Status 子资源模式
- **操作执行**：基于 K8s Workqueue 实现异步操作队列
- **操作 Agent**：
  - 与多进程容器分离，独立部署
  - 通过 gRPC 接收 Controller 调用
  - 具体操作在注册配置中声明（如操作名称、参数模板等）
  - 第一期实现框架，具体操作由注册配置定义
- **失败处理**：操作失败时记录 Condition，支持重试

---

### 用户需求 4：备份恢复

#### 需求详情
支持完整的备份恢复功能：

**4.1 备份类型**
- 全量备份
- 增量备份（第一期暂不实现，按需备份 + 定时备份）
- 按需备份
- 定时备份（Cron 表达式）

**4.2 备份对象**
- 数据备份（数据库 dump）
- 配置文件备份
- 关联元数据备份

**4.3 一致性保证**
- 平台提供冻结/解冻 API 供中间件插件实现一致性备份
- PostgreSQL：使用 pg_dump 或 WAL 日志
- Redis：使用 BGSAVE 或 RDB/AOF

**4.4 恢复粒度**
- 全量恢复
- 恢复到指定备份点

**4.5 存储后端**
- K8s PVC（用户自备 StorageClass）
- 对象存储（S3、MinIO、阿里云 OSS、腾讯云 COS 等）
- NFS

**4.6 生命周期管理**
- 备份保留策略（按数量、按时间）
- 备份过期自动清理
- 备份加密（第一期暂不实现）

**4.7 恢复范围**
- **单集群内恢复（第一期）**
- 跨集群恢复（暂不需要）

#### 技术洞察
- **备份插件**：每种中间件实现备份接口，定义备份命令和恢复命令
- **存储抽象**：定义 Storage Backend Interface，支持 PVC/S3/NFS 等多种后端
- **一致性方案**：
  - PostgreSQL：pg_dump + WAL
  - Redis：BGSAVE + SYNC
- **备份格式**：标准格式（tar.gz + checksum）
- **工具建议**：可复用现有备份工具（pg_dump、redis-cli BGSAVE），避免重复造轮子

---

### 用户需求 5：监控

#### 需求详情
**5.1 监控数据采集**
- 对接 **Prometheus**（用户已有或自建）
- 监控 Agent 容器暴露 Prometheus Metrics 端点（可选，第一期暂不实现）
- Controller 生成 ServiceMonitor 或 PodMonitor CRD

**5.2 监控指标范围**
- 基础 Pod 指标（CPU、内存、磁盘、网络）- K8s 原生采集
- 中间件业务指标（QPS、延迟、连接数等）- 监控 Agent 采集并暴露（可选）

**5.3 告警功能**
- **暂不实现**（第一期只实现监控采集，告警功能后续迭代）

#### 技术洞察
- **集成方案**：Prometheus Operator 生态（ServiceMonitor/PodMonitor）
- **指标规范**：遵循 Prometheus 指标命名规范（中间件类型_指标名）
- **预置指标**：每种中间件插件提供默认 Metrics 定义
- **监控 Agent**：可选组件，负责采集并暴露中间件业务指标

---

### 用户需求 6：存储和网络

#### 需求详情
**6.1 存储需求**
- 中间件数据存储：K8s PVC（块存储）
  - 用户指定 StorageClass
  - 支持在线存储扩容（依赖 StorageClass 支持 VolumeExpansion）
  - 暂不支持本地存储（LocalPV）
  - 暂不需要存储快照（通过备份实现数据保护）
- 备份存储：对象存储（S3/MinIO/云存储）或 NFS

**6.2 网络需求**
- Service 类型：由中间件应用场景决定（ClusterIP/NodePort/LoadBalancer）
- 网络模型：Overlay（默认）
- 网络隔离：NetworkPolicy（可选项，用户按需配置）
- DNS 特性：Headless Service、SRV 记录

**6.3 准入和调度**
- 节点亲和性：Pod Affinity/Anti-Affinity（用于跨节点、跨 AZ 部署）
- 污点容忍：Taints 和 Tolerations（用于专有节点）
- 资源配额：不限制（ResourceQuota 可由管理员自行配置）

**6.4 命名空间**
- 用户可指定中间件安装到哪个 Namespace

**6.5 TLS 支持**
- 平台提供通用的 TLS 支持，自动为中间件实例生成证书
- TLS 加密以下通信：
  - 客户端 ↔ 中间件
  - 中间件 ↔ 中间件（复制通信）
- 证书生成：集成 **CertManager** 自动生成证书
- 证书挂载：以 Secret 方式挂载到中间件 Pod
- 用户在 Application CRD 中通过配置字段启用 TLS

#### 技术洞察
- **证书管理**：集成 CertManager，通过 Certificate CRD 自动生成和续期证书
- **Secret 卷**：生成的证书以 Secret 形式存储，挂载到 Pod 的卷中
- **中间件配置**：Agent 容器负责将证书路径和配置写入中间件配置文件
- **StorageClass**：用户在 Application CRD 中指定 storageClassName
- **存储扩容**：依赖 StorageClass 支持 VolumeExpansion
- **Service 设计**：每种中间件根据场景创建对应的 Service
- **Headless Service**：用于有状态应用的稳定网络标识
- **拓扑分布**：通过 Pod Topology Spread Constraints 实现跨 AZ 分布

---

### 用户需求 7：高可用和故障恢复

#### 需求详情
**7.1 高可用设计**
- 可用性级别由中间件部署拓扑决定（平台提供能力，用户决定配置）
- 跨可用区（AZ）部署：支持
- PodDisruptionBudget（PDB）：支持，可配置开关
- HA 模式：
  - 单机（开发/测试环境）
  - 主从/副本（基础 HA）
  - 多副本 + 自动故障转移（生产级 HA）

**7.2 故障恢复机制**
- **Pod 崩溃**：自动重启或重新调度
- **Host 故障**：Pod 漂移到其他Host（数据由 PVC 保证）
- **主节点故障**：依赖中间件自身机制（Redis Sentinel、PostgreSQL Streaming Replication），平台只监控状态
- **存储故障**：通过备份恢复
- **Controller 故障**：Controller 本身 HA（Deployment 多副本 + Leader Election）

#### 技术洞察
- **Controller HA**：使用 K8s Lease 实现 Leader Election
- **PDB**：每个 Component CRD 或 WorkloadSet CRD 可配置 PDB 规则（minAvailable/maxUnavailable）
- **跨 AZ**：通过 topologyKey: topology.kubernetes.io/zone 实现
- **故障检测**：Controller 定期检查 Pod/Service 状态，触发对应操作

---

### 用户需求 8：中间件扩展机制

#### 需求详情
通过插件注册配置（Plugin Registration Config）实现中间件扩展，新中间件只需注册插件即可接入，无需修改 Controller 代码。注册配置采用版本控制，确保配置的稳定性和可追溯性。

**8.1 注册配置组成**
每种中间件的注册配置由一组 ConfigMap 组成：

| ConfigMap 类型 | 内容 | 说明 |
|---------------|------|------|
| **配置清单 ConfigMap** | 版本信息、支持的中间件版本、对其他 ConfigMap 的引用 | 每个中间件类型可能有多个配置清单 ConfigMap，代表不同的配置版本，中间件实例化时，需要在 CR 字段中明确指定所使用的配置清单版本 |
| **Component 配置 ConfigMap** | Component 定义 | 定义 Component 的名称、镜像、资源规格、配置参数模板、副本数范围等 |
| **Pod 模板 ConfigMap** | Pod 模板配置 | 定义 Pod 的规格、容器、环境变量等 |
| **参数模板 ConfigMap** | PaaS 默认参数 | 中间件的配置参数默认值 |
| **操作模板 ConfigMap** | 生命周期操作模板 | 定义 create/delete/scale 等操作的执行逻辑 |

**Component 设计原则**：
- 按运行的二进制划分 Component（Component 内所有 Pod 运行相同的中间件二进制）
- 使用相同的资源配置和配置参数模板
- 运行时角色由生命周期操作逻辑处理，状态通过状态 Agent 上报
- 如果角色确定后保持不变，可按角色划分 Component（如 Kafka = broker + kraft）

示例（Redis 注册配置）：
```yaml
# 示例：配置清单 ConfigMap
redis-manifest-v1.0.0
  supported-versions: "7.2,7.4"
  allowed-upgrade-from: ""
  components-ref: "redis-components-v1.0.0"
  podtemplates-ref: "redis-podtemplates-v1.0.0"
  params-ref: "redis-params-v1.0.0"
  operations-ref: "redis-operations-v1.0.0"

# 示例：其他 ConfigMap
redis-components-v1.0.0      # 示例：Component 配置（如 redis-node，3 副本）
redis-podtemplates-v1.0.0    # 示例：Pod 模板
redis-params-v1.0.0          # 示例：参数模板
redis-operations-v1.0.0      # 示例：操作模板
```

**8.2 配置格式**
- 主要格式：**YAML**
- 辅助格式：**CUE**（用于配置验证、模板约束、数据校验）

**8.3 版本控制规则**
- **版本唯一性**：每个版本一经发布不可修改（只增不改）
- **版本格式**：语义化版本（如 v1.0.0, v1.1.0, v2.0.0）
- **多版本支持**：一个注册配置版本可支持多个中间件版本（如 v1.0.0 支持 Redis 7.2, 7.4）
- **向下兼容**：高版本支持低版本声明的部分或所有中间件版本来提供向下兼容

**8.4 升级约束**
- **注册配置版本升级**：必须从配置清单中 `allowed-upgrade-from` 声明的前置版本升级
  - 例如：v2.0.0 的 `allowed-upgrade-from: "v1.0.0,v1.1.0"`
  - 表示 v1.0.0 和 v1.1.0 都可以升级到 v2.0.0
- **中间件版本升级**：必须在当前注册配置版本 `supported-versions` 声明的范围内

**8.5 实例与版本绑定**
- **创建实例**：用户在 Application CRD 中通过 `pluginVersion` 字段指定使用的注册配置版本
- **实例迁移**：用户可选择将已有实例迁移到新版本，或保持当前版本不变
- **版本锁定**：实例创建后使用固定版本的注册配置，中间件版本升级也受该版本约束

**8.6 部署模式配置**
- 注册配置定义中间件的 Component 构成（如 Redis = redis-node，Redis + Sentinel = redis-node + sentinel，Kafka = kafka-broker）
- 注册配置定义每个 Component 的默认参数模板
- Application CRD 实例通过组件字段指定使用哪些 Component 及副本数
- 例如：PostgreSQL 集群可指定主库 x1，从库 x2

**8.7 生命周期操作配置**
- 注册配置使用**声明式配置**定义操作逻辑
- 每种操作（create/delete/scale 等）在注册配置中定义步骤和命令模板
- Controller 根据 CRD 状态和实际状态，在合适的时机执行操作
- 执行时使用实际的参数值填充模板中的占位符

#### 技术洞察
- **CUE 用途**：配置验证、模板约束、多版本配置管理
- **ConfigMap 组织**：按中间件类型和版本命名（如 `redis-manifest-v1.0.0`）
- **版本管理**：通过配置清单 ConfigMap 的 `supported-versions` 和 `allowed-upgrade-from` 实现版本约束
- **声明式操作**：注册配置定义"做什么"，Controller 决定"何时做"和"怎么做"
- **配置分层**：
  - **注册配置**：定义中间件如何接入（Component 构成、操作模板、参数模板），版本受控
  - **中间件运行时配置**：中间件自身的配置文件（redis.conf、postgresql.conf 等），由状态 Agent 管理（如 Reload 操作）

---

### 用户需求 9：多进程容器设计

#### 需求详情
为实现中间件进程重启时不重启 Pod，采用多进程容器架构：

**9.1 多进程容器架构**
- Pod 中包含一个**多进程容器**，运行进程管理器作为 1 号进程
- 进程管理器负责管理中间件进程的启动、停止、重启、故障自动拉起
- 进程管理器提供 **gRPC 服务**，供 Controller 调用执行重启等操作

**9.2 进程管理器职责**

| 功能 | 说明 |
|------|------|
| 进程管理 | 启动、停止、重启中间件进程 |
| 故障自动拉起 | 中间件进程异常退出后自动重新拉起 |
| 健康检查 | 监控中间件进程状态 |
| gRPC 服务 | 提供接口供外部调用（接收重启、重载等控制指令） |
| 信号处理 | SIGHUP（日志轮转）、SIGTERM（优雅关闭）、SIGUSR1（重新加载配置）等 |

**9.3 容器组成**

| 容器 | 职责 |
|------|------|
| **多进程容器** | 1 号进程是进程管理器，管理中间件进程；提供 gRPC 服务接收控制指令 |
| **状态 Agent 容器** | 轻量级 Sidecar，负责状态采集和上报，具体职责见"用户需求 3：Agent 容器设计" |

**9.4 重启流程**
- 用户请求重启或 Controller 检测到需要重启
- Controller 通过 gRPC 调用进程管理器执行重启
- 进程管理器重启中间件进程，不重启 Pod

#### 技术洞察
- **进程管理器**：轻量级进程管理器（如 dumb-init 的扩展，或自定义实现）
- **gRPC 服务**：Controller 与进程管理器之间的控制通道
- **状态查询**：状态 Agent 通过本地命令行工具获取状态（redis-cli、pg_isready 等），不依赖 gRPC
- **重启优势**：避免 Pod 重启带来的网络中断、数据迁移等开销

---

### 用户需求 10：统一操作入口与操作日志

#### 需求详情
引入新的 **Operation CRD**，作为对 PaaS 所有操作的统一入口和操作日志记录。相比 Application CRD，Operation CRD 更通用，支持多种操作类型。

**10.1 API 操作方式**
- 用户直接使用 kubectl YAML 操作 CRD
- 创建/更新/删除 Application、Component、WorkloadSet、Operation 等资源
- 无需额外的 API Server 或 kubectl 插件

**10.2 操作触发方式**
- 用户创建 Operation CRD 实例触发操作（Operation 是统一操作入口）
- 用户直接修改 Application/Component CRD 触发变更（如 replicas、resources）
- Controller 监听 CRD 事件，自动执行 Reconcile 逻辑

**10.1 Operation CRD 职责**

| 功能 | 说明 |
|------|------|
| **统一操作入口** | 所有操作（生命周期管理、PaaS 专有操作、备份恢复等）都通过创建 Operation CRD 实例发起 |
| **操作类型** | 支持 CRUD 操作（创建/更新/删除 Application）、gRPC 调用等多种操作 |
| **操作日志记录** | 记录操作历史，包括操作类型、操作时间、操作参数、操作结果（不含操作者） |
| **操作状态跟踪** | 记录操作进度、结果、失败原因，支持操作重试 |

**10.2 Operation 步骤类型**

每个 Operation 可包含多个步骤（Steps），步骤在注册配置中预先定义，Operation 通过引用操作模板执行。支持以下步骤类型：

| 步骤类型 | 说明 | 示例 |
|----------|------|------|
| **CRD 资源操作** | 创建/更新/删除 Component、WorkloadSet 等自定义资源 | Create WorkloadSet、Delete Component |
| **K8s 资源操作** | 创建/更新/删除 Pod、Service、ConfigMap、Secret 等原生 K8s 资源 | Create Service、Update ConfigMap |
| **gRPC 调用** | 调用操作 Agent 执行特定操作 | Redis ACL 配置、Kafka Topic 配置 |

**10.3 组合操作示例**
- 多个步骤按配置顺序依次执行
- 典型组合操作示例：**Failover**（停止主节点 → 提升从节点 → 更新状态）

**10.4 Operation CRD 字段设计**

> 以下为示例设计，具体实现时可能调整。

| 字段 | 类型 | 说明 |
|------|------|------|
| `spec.templateRef` | string | 引用的操作模板（在注册配置中定义） |
| `spec.targetRef` | ObjectReference | 目标资源引用（Application 或其他） |
| `spec.parameters` | map[string]string | 操作参数 |
| `status.phase` | string | 操作状态（Pending/Running/Succeeded/Failed） |
| `status.currentStep` | int | 当前执行到的步骤序号 |
| `status.startTime` | time | 操作开始时间 |
| `status.completionTime` | time | 操作完成时间 |
| `status.error` | string | 失败原因 |

**10.5 与其他 CRD 的关系**
- Operation CRD 独立于 Application、Component、WorkloadSet
- Operation 通过 `targetRef` 引用目标资源（通常是 Application）
- Operation 通过 `templateRef` 引用注册配置中预定义的操作模板

**10.6 操作日志保留策略**
- 可配置的日志保留策略
- 按时间保留（如保留最近 30 天）
- 按数量保留（如保留最近 100 条）
- 支持自动清理过期日志

**10.7 操作触发方式**
- 用户创建 Operation CRD 实例触发操作
- Controller 监听 Operation 事件，按步骤顺序执行操作
- 步骤执行结果更新到 Operation Status

#### 技术洞察
- **操作入口统一**：所有操作通过 Operation CRD 发起，便于审计和追溯
- **操作通用性**：Operation 支持 CRD 操作和 gRPC 调用，覆盖所有运维场景
- **步骤编排**：通过注册配置预定义操作步骤，支持多步骤组合操作
- **操作日志存储**：操作日志作为 Operation CRD 的实例存储在 etcd 中
- **日志保留策略**：通过 TTL 或配额控制日志保留范围，避免 etcd 膨胀
- **参考设计**：参考 Kubernetes Job/Event 的设计模式

---

## 用户需求 11：日志管理

#### 需求详情
**11.1 平台组件日志**
- Controller 日志：通过标准输出（stdout）提供，用户自行通过 K8s 日志系统采集（如 Loki、EFK）
- Agent 容器日志：作为中间件 Pod 的 Sidecar 容器，日志通过标准输出提供，用户自行采集

**11.2 中间件访问日志**
- 由用户在中间件配置中自行启用和配置
- 平台不强制记录，但提供相应的存储卷挂载支持
- 如需导出，由用户自行对接外部日志系统

#### 技术洞察
- **日志采集方式**：所有组件日志通过 K8s 标准日志机制采集
- **日志存储**：用户负责配置日志存储后端（EFK/Loki/云日志服务）
- **Agent 日志**：Agent 容器的日志与中间件 Pod 日志一起采集

---

## 用户需求 12：多租户隔离和命名空间管理

#### 需求详情
**12.1 命名空间隔离**
- 中间件实例部署在用户指定的 Kubernetes Namespace 中
- 每个 Namespace 内的中间件实例相互隔离
- 命名空间是资源隔离的基本单元

**12.2 资源配额**
- 平台层面不内置 ResourceQuota 限制
- ResourceQuota 由管理员根据需要在 Kubernetes 层面自行配置

**12.3 网络隔离**
- 租户级别的网络隔离不在平台层面实现
- 如有需要，由管理员通过 Kubernetes NetworkPolicy 自行配置

**12.4 跨命名空间管理**
- 支持跨 Namespace 的资源管理视图
- 管理员可以查看和管理任意 Namespace 内的中间件实例
- 普通用户只能访问其指定的 Namespace

#### 技术洞察
- **隔离策略**：基于 Kubernetes Namespace 的软隔离
- **管理权限**：通过 Kubernetes RBAC 控制谁能管理哪些 Namespace 的资源
- **视图能力**：Controller 遍历所有 Namespace（受 RBAC 限制），收集资源状态

---

## 用户需求 13：平台自身运维

#### 需求详情
**13.1 Controller 镜像升级**
- 由用户控制升级时机，采用 Kubernetes 滚动更新策略
- 用户通过 `kubectl set image` 或修改 Deployment 更新 Controller 镜像版本
- Controller Deployment 配置 RollingUpdate 策略，逐个升级 Pod，对服务影响最小

**13.2 注册配置管理**
- 注册配置采用版本化管理，直接创建新版本（如 `redis-manifest-v2`），旧版本保留
- 每个版本一经发布不可修改（遵循 8.4 版本控制规则）
- 注册配置更新对 Controller 影响：
  - Controller 持续监听所有注册配置 ConfigMap
  - 新增注册配置版本时，Controller 自动感知，无需重启
  - 现有实例继续使用其绑定的注册配置版本，不受影响
  - 新实例可选择使用新版本或继续使用旧版本

**13.3 Controller 健康检查**
- Liveness Probe：检测 Controller 是否存活，失败则重启
- Readiness Probe：检测 Controller 是否就绪，未就绪前不接受 Reconcile 请求

#### 技术洞察
- **Controller 升级**：利用 K8s Deployment 的滚动更新能力，用户完全控制升级时机
- **注册配置隔离**：多版本注册配置并存，实例绑定固定版本，实现平滑升级
- **热更新**：注册配置变更由 Controller 通过 Informer 机制自动感知，无需重启
- **参考设计**：参考 Helm Chart 的版本管理方式

---

## 用户需求 14：成本管理和优化（第一期可暂缓实现）

#### 需求详情
**14.1 成本统计**
- 统计各 Namespace/用户的中间件资源使用量（CPU、内存、存储）
- 生成资源使用报表，支持按时间维度查看
- 便于运维团队了解资源消耗情况

**14.2 闲置资源检测（可支持，优先级不高）**
- 自动检测长期不活跃的中间件实例（如无连接、无操作的实例）
- 提供闲置实例列表，提醒用户清理

**14.3 自动扩缩容（可支持，优先级不高）**
- HPA（Horizontal Pod Autoscaler）：根据 CPU/内存负载自动调整副本数
- 时间维度扩缩容：根据时间计划自动调整资源（如非工作时间缩减非核心实例）

#### 技术洞察
- **成本统计**：通过 Prometheus 或 K8s Metrics Server 采集资源使用数据
- **闲置检测**：基于连接数、操作日志等维度判断实例活跃度
- **自动扩缩容**：利用 K8s HPA 机制，结合自定义指标实现
- **实现优先级**：第一期可暂不实现，作为后续迭代功能

---

## 用户需求 15：非阻塞编排（关键架构约束）

#### 需求详情
采用多层级 CRD 设计后，Controller 在编排中间件实例时，必须确保**不能阻塞其他中间件实例的编排**：

| 约束项 | 说明 |
|--------|------|
| **Reconcile 并发处理** | 使用 K8s Workqueue + 多 Worker 并行处理，每个实例的 Reconcile 独立进行 |
| **故障隔离** | 单个实例的 Reconcile 失败/超时不影响其他实例的处理，不产生级联阻塞 |
| **资源限制** | 单个 Reconcile 过程有资源使用上限和超时控制（如 CPU、内存、超时时间） |
| **K8s API 限流** | 遵守 K8s API Server 的 rate limit，不占用全部 API quota 留给其他请求 |

**实现要点：**
- 配置多个 Worker（如 5-10 个）并行处理 Reconcile 事件
- 单个 Reconcile 设置超时（如 5-10 分钟），超时后跳过，避免长时间占用队列
- K8s API 调用使用 client-go 的 rate limiter 控制请求速率
- 错误和重试不影响队列中其他实例的处理

---

## 技术约束和限制

### 1. 轻量级要求
- 平台本身不引入结构化数据库依赖
- 所有状态存储在 K8s CRD 中
- Controller 只依赖 K8s API 和 etcd

### 2. 可扩展性要求
- 新中间件接入无需修改 Controller 核心代码
- 插件注册配置驱动，无需重新编译
- 支持热更新（Controller 配置）和重启生效（中间件配置）两种模式

### 3. 兼容性要求
- K8s 版本兼容（建议 1.20+）
- 中间件版本兼容策略
- CRD 版本升级兼容性

### 4. 第一期交付范围
- Controller + API（不含 Web UI）
- PostgreSQL + Redis 支持
- 备份恢复（单集群内）
- 监控采集（不含告警）


