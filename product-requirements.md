# 基于Kubernetes的轻量级通用PaaS中间件管理平台需求文档

## 项目概述

本项目旨在构建一个基于Kubernetes的轻量级、可扩展的通用PaaS中间件管理平台。该平台能够管理多种中间件（如Redis、Kafka、MySQL等），提供中间件的全生命周期管理、备份恢复、配置管理、监控告警等核心功能。平台通过配置驱动的方式支持扩展，用户可以轻松添加对新中间件类型的支持。平台本身保持轻量级，不引入对结构化数据库、缓存等外部中间件的依赖，所有功能实现基于Kubernetes原生能力和声明式配置。

## 专有词汇中-英对照表

| 中文术语 | 英文术语 |
|---------|---------|
| Kubernetes | K8s |
| 自定义资源定义 | CRD (Custom Resource Definition) |
| 运算符 | Operator |
| 命令行接口 | CLI (Command Line Interface) |
| 基于角色的访问控制 | RBAC (Role-Based Access Control) |
| 传输层安全 | TLS (Transport Layer Security) |
| 证书颁发机构 | CA (Certificate Authority) |
| 开放容器倡议 | OCI (Open Container Initiative) |
| 点到点恢复 | PITR (Point-In-Time Recovery) |
| 滚动更新 | Rolling Update |
| 蓝绿部署 | Blue-Green Deployment |
| 原地升级 | In-place Update |
| 应用程序编程接口 | API (Application Programming Interface) |
| 表示状态传输 | REST (Representational State Transfer) |
| 基本认证 | Basic Auth |
| 开放授权 | OAuth |
| 多因素认证 | MFA (Multi-Factor Authentication) |

## 用户需求

### 用户需求1：目标用户和使用场景

#### 需求详情

**目标用户**：
- 作为产品提供给外部客户使用

**使用场景**：
- 支持开发测试环境和生产环境

**使用方式**：
- 通过Web界面进行操作
- 通过原生kubectl工具进行管理

#### 技术洞察

1. **Kubernetes CRD架构**：由于用户需要通过kubectl原生工具管理平台，平台必须基于Kubernetes CRD（Custom Resource Definition）实现。用户可以像管理K8s原生资源（如Deployment、Service）一样管理中间件实例。

2. **声明式API设计**：平台应遵循Kubernetes的声明式API设计理念，用户通过YAML文件描述期望状态，平台负责将实际状态调整为期望状态。

3. **Web界面与K8s API集成**：Web界面应通过Kubernetes API Server进行操作，这样可以复用K8s的认证授权机制，并确保Web界面和kubectl操作的一致性。

4. **环境差异化配置**：支持开发和生产环境意味着平台需要考虑不同环境下的配置差异，如资源配额、备份策略、监控告警阈值等。

### 用户需求2：部署规模和环境要求

#### 需求详情

**集群管理**：
- 不需要支持多集群管理，仅管理单个Kubernetes集群

**资源消耗**：
- 平台需要保持轻量级，资源占用少
- 没有定量的资源限制要求，但从定性角度要求轻量级

**Kubernetes环境兼容性**：
- 支持云厂商托管的Kubernetes集群（如ACK、EKS、GKE等）
- 支持自建Kubernetes集群
- 支持Kubernetes 1.21及以上版本

#### 技术洞察

1. **轻量级架构设计**：为实现轻量级目标，平台应：
   - 采用单一进程架构，避免多组件部署
   - 使用Kubernetes原生资源（CRD、Controller）而非引入额外依赖
   - 避免使用重量级框架和库
   - 优化资源使用，如设置合理的资源请求和限制

2. **Kubernetes版本兼容性**：
   - 使用Kubernetes client-go库，并设置合理的API版本兼容范围
   - 避免使用过新或过旧的K8s API特性
   - 考虑使用Kubernetes的API deprecation机制处理版本变更

3. **云厂商适配**：
   - 避免依赖特定云厂商的CSI（Container Storage Interface）实现
   - 使用通用的存储类（StorageClass）定义
   - 支持标准的Kubernetes资源对象

### 用户需求3：中间件生命周期管理

#### 需求详情

**核心生命周期操作**：
1. **create**：创建中间件实例
2. **delete**：删除中间件实例
3. **vertical-scale-resource**：调整CPU和内存资源
4. **vertical-scale-volume**：调整存储资源
5. **horizontal-scale**：调整副本数量
6. **stop/start/restart**：停止/启动/重启中间件实例
7. **failover**：故障转移
8. **reload**：重载配置
9. **upgrade**：版本升级

#### 技术洞察

1. **Kubernetes Operator模式**：
   - 使用一个或多个通用CRD负责所有类型中间件的编排
   - 通用CRD通过type字段标识中间件类型，通过spec字段传递中间件特定的配置
   - 实现Reconciliation Loop，持续监控CRD状态并调整实际状态
   - 使用Finalizer机制处理资源清理和依赖关系

2. **生命周期操作实现**：
   - **create/delete**：通过K8s Deployment/StatefulSet管理Pod，通过Service/Ingress暴露服务
   - **vertical-scale-resource**：修改Deployment/StatefulSet的resources字段
   - **vertical-scale-volume**：修改PVC/StorageClass，可能需要数据迁移
   - **horizontal-scale**：修改Deployment的replicas字段或StatefulSet的replicas字段
   - **stop/start/restart**：通过修改Deployment的replicas为0或原值，或发送信号给进程
   - **failover**：实现主从切换逻辑，可能需要使用Leader Election
   - **reload**：发送SIGHUP信号或调用中间件的重载API
   - **upgrade**：滚动更新Pod镜像，处理数据迁移和兼容性检查

3. **状态机设计**：
   - 定义中间件实例的状态：Pending、Creating、Running、Stopping、Stopped、Upgrading、Failed、Deleting等
   - 状态转换需要考虑幂等性和错误恢复

4. **依赖管理**：
   - 使用OwnerReference建立资源间的父子关系
   - 实现级联删除和依赖清理

### 用户需求4：配置管理

#### 需求详情

**必选功能**：
- 支持配置文件的动态修改和热加载
- 支持配置模板和参数化
- 支持配置校验和预检查

**可选功能**：
- 支持配置版本管理和回滚

#### 技术洞察

1. **配置管理策略**：
   - 使用Kubernetes ConfigMap存储配置文件
   - 支持通过CRD的spec字段传递配置参数
   - 使用模板引擎（如Go Template、CUE）渲染配置文件

2. **动态配置和热加载**：
   - 监听ConfigMap变更，触发中间件重载
   - 对于不支持热加载的中间件，触发滚动重启

3. **配置校验**：
   - 在CRD定义中使用validation规则（Kubernetes 1.16+支持）
   - 实现Admission Webhook进行复杂校验
   - 提供配置预检查接口，在应用前验证配置有效性

4. **配置版本管理（可选）**：
   - 使用Kubernetes的annotation记录配置版本
   - 可以集成GitOps工具（如ArgoCD）实现配置版本管理
   - 支持回滚到历史配置版本

5. **敏感信息处理**：
   - 使用Kubernetes Secret存储密码、密钥等敏感信息
   - 通过环境变量或Volume挂载注入到Pod
   - 使用配置模板中的占位符，运行时替换敏感信息

### 用户需求5：扩展机制

#### 需求详情

**扩展方式**：
- 采用配置驱动方式
- 扩展粒度为整个中间件类型（如添加对Elasticsearch的支持）

**扩展能力**：
- 支持热加载（不重启平台就能加载新的扩展）
- 扩展包的配置文件格式包括YAML、CUE、Go Template等
- 需要扩展市场/扩展仓库的概念
- 扩展包分发支持OCI镜像仓库，并考虑支持多种分发方式

#### 技术洞察

1. **配置驱动架构**：
   - 定义扩展包的元数据格式（如manifest.yaml），描述中间件类型、版本、支持的配置参数等
   - 扩展包包含中间件的部署模板（Helm Chart/Kustomize）、配置模板、监控指标定义、生命周期操作脚本等
   - 平台加载扩展包后，通过通用CRD的type字段引用中间件类型，无需动态注册新的CRD

2. **扩展包结构设计**：
   ```
   extension/
   ├── manifest.yaml          # 扩展包元数据
   ├── templates/             # 配置和部署模板
   │   ├── deployment.yaml
   │   ├── service.yaml
   │   └── configmap.yaml
   ├── monitoring/            # 监控指标定义
   │   └── metrics.yaml
   └── scripts/              # 辅助脚本（可选）
       ├── backup.sh
       └── restore.sh
   ```

3. **通用CRD设计**：
   - 通用CRD（如Middleware）包含type字段标识中间件类型（如redis、mysql、kafka等）
   - spec字段包含中间件特定的配置参数（资源、副本、版本等）
   - 平台控制器根据type字段加载对应的扩展包，使用扩展包中的模板渲染Kubernetes资源
   - 用户通过kubectl apply创建中间件实例时，指定type字段即可

4. **热加载实现**：
   - 监听扩展包目录或OCI镜像仓库的变更
   - 加载新的扩展包到平台的扩展包缓存
   - 更新平台的配置模板缓存
   - 无需重启Controller，用户即可通过通用CRD创建新类型的中间件实例

5. **OCI镜像仓库分发**：
   - 将扩展包打包为OCI镜像（artifact type为middleware-extension）
   - 使用标准的OCI registry（如Docker Hub、Harbor、阿里云ACR等）
   - 支持版本标签和签名验证
   - 使用oras工具或自定义客户端拉取扩展包

6. **扩展市场/仓库**：
   - 扩展市场可以是一个公开的Git仓库或简单的Web服务
   - 提供扩展包的索引、搜索、元数据查询功能
   - 支持用户提交和分享自定义扩展

7. **多格式支持**：
   - YAML：简单易读，适合配置定义
   - CUE：强类型约束，适合复杂配置校验
   - Go Template：灵活的模板渲染，适合动态配置生成
   - 平台应支持多种格式，或提供转换机制

### 用户需求6：备份和恢复

#### 需求详情

**备份能力**：
- 支持全量备份和增量备份
- 备份存储位置包括本地存储和对象存储（S3/OSS等）
- 支持定时自动备份和手动触发备份
- 支持备份恢复到指定时间点（PITR）
- 支持备份的保留策略（保留多少个备份、保留多长时间）

**备份数据加密**：
- 支持备份数据加密，由用户配置决定

#### 技术洞察

1. **备份策略设计**：
   - 全量备份：定期执行完整数据备份
   - 增量备份：基于上次备份的变更数据，减少备份时间和存储空间
   - 差异备份：基于上次全量备份的变更数据

2. **存储后端**：
   - 本地存储：使用PVC挂载到备份Pod
   - 对象存储：支持S3兼容的存储服务（AWS S3、阿里云OSS、MinIO等）
   - 使用CSI或S3 SDK实现统一的存储接口

3. **备份调度**：
   - 使用Kubernetes CronJob实现定时备份
   - 支持Cron表达式定义备份计划
   - 提供手动触发备份的接口（kubectl命令或Web界面操作）

4. **PITR实现**：
   - 对于支持WAL（Write-Ahead Log）的中间件（如PostgreSQL、MySQL），利用WAL日志实现时间点恢复
   - 记录备份的时间戳和LSN（Log Sequence Number）
   - 恢复时应用WAL日志到指定时间点

5. **保留策略**：
   - 基于数量：保留最近N个备份
   - 基于时间：保留最近N天/周的备份
   - 基于标签：保留特定标签的备份
   - 自动清理过期的备份

6. **备份加密**：
   - 支持对称加密（AES-256）和非对称加密（RSA）
   - 加密密钥可以存储在Kubernetes Secret中
   - 支持使用KMS（Key Management Service）管理密钥

7. **备份验证**：
   - 提供备份完整性校验（checksum）
   - 支持备份恢复测试（可选）

### 用户需求7：监控和告警

#### 需求详情

**监控**：
- 使用Prometheus生态
- 依赖集群中已有的Prometheus
- 中间件实例暴露metrics endpoint，Prometheus直接抓取

**告警**：
- 平台根据用户配置，生成PrometheusRule CRD
- 告警通知使用Prometheus Alertmanager

#### 技术洞察

1. **Prometheus集成**：
   - 中间件Pod暴露/metrics endpoint（使用Prometheus客户端库）
   - 通过ServiceMonitor（Prometheus Operator）或Pod annotation配置Prometheus抓取
   - 支持自定义监控指标

2. **监控指标类型**：
   - Counter：计数器（如请求总数）
   - Gauge：仪表盘（如当前连接数）
   - Histogram：直方图（如请求延迟分布）
   - Summary：摘要（如请求延迟的P50、P95、P99）

3. **告警规则管理**：
   - 定义告警规则模板（CPU使用率、内存使用率、连接数、QPS、延迟等）
   - 用户通过CRD配置告警规则参数
   - 平台生成PrometheusRule CRD，Prometheus Operator加载规则

4. **Alertmanager集成**：
   - 配置Alertmanager的接收器（receiver）和路由（route）
   - 支持多种通知渠道：邮件、钉钉、企业微信、Slack、Webhook等
   - 支持告警分组、抑制、静默等高级功能

5. **告警级别**：
   - Critical：严重告警，需要立即处理
   - Warning：警告告警，需要关注
   - Info：信息告警，仅记录

6. **告警去重和聚合**：
   - 利用Alertmanager的group_by和inhibit规则
   - 避免告警风暴

### 用户需求8：权限和多租户

#### 需求详情

**多租户**：
- 不需要支持多租户

**权限管理**：
- 平台本身不实现权限管理
- 依赖Kubernetes原生RBAC机制

#### 技术洞察

1. **Kubernetes RBAC集成**：
   - 用户通过kubectl操作时，权限由K8s RBAC控制
   - Web界面通过K8s API Server进行认证授权
   - 使用ServiceAccount Token或kubeconfig进行认证

2. **资源隔离**：
   - 使用Kubernetes Namespace进行逻辑隔离
   - 不同Namespace的资源互不可见
   - 通过NetworkPolicy实现网络隔离（可选）

3. **无需多租户架构**：
   - 简化架构设计，避免多租户相关的复杂性
   - 专注于单租户场景的功能实现

### 用户需求9：API和集成

#### 需求详情

**API**：
- 如果提供REST API，只会提供给Web界面使用
- 目前不确定Web界面的开发过程中是否使用REST API

**集成**：
- 暂不考虑与其他系统集成（如CI/CD、日志系统、配置中心等）

#### 技术洞察

1. **REST API设计（可选）**：
   - 如果提供，遵循RESTful设计原则
   - 使用JWT或Session进行认证（虽然Web界面本身不实现认证）
   - 提供中间件CRD的CRUD操作接口
   - 提供生命周期操作接口（启动、停止、扩缩容等）

2. **无外部集成依赖**：
   - 保持平台轻量级，不引入外部依赖
   - 所有功能基于Kubernetes原生能力
   - 用户可以通过kubectl和K8s API进行自定义集成

### 用户需求10：运维和维护

#### 需求详情

**平台部署**：
- 支持Helm Chart部署

**平台升级**：
- 主要考虑滚动升级
- 其他升级方式（蓝绿部署等）作为可选项，优先级低

**平台健康检查和自愈**：
- 不需要平台自身的健康检查和自愈

**平台日志和监控**：
- 需要日志，直接在容器的stdout输出

**操作审计和事件记录**：
- 需要提供平台级的操作审计日志
- 需要提供事件记录（记录中间件实例的状态变化）

**诊断工具**：
- 仅需要诊断平台本身的工具，优先级低

#### 技术洞察

1. **Helm Chart部署**：
   - 提供标准的Helm Chart，包含所有必要的Kubernetes资源
   - 支持values.yaml自定义配置
   - 遵循Helm最佳实践（使用命名模板、提供合理的默认值等）

2. **滚动升级**：
   - 使用Kubernetes Deployment的滚动更新策略
   - 配置maxSurge和maxUnavailable参数
   - 支持健康检查（readinessProbe和livenessProbe）

3. **蓝绿部署（可选）**：
   - 可以使用Kubernetes原生方式实现（两个Deployment + Service selector切换）
   - 不依赖外部服务（如Istio、Argo Rollouts）
   - 作为可选项，优先级低

4. **日志输出**：
   - 平台组件（Controller、Web服务）的日志输出到stdout
   - 使用结构化日志格式（如JSON），便于日志收集和分析
   - 可以被K8s日志收集系统（如Fluentd、Logstash）收集

5. **操作审计日志**：
   - 记录用户的所有操作（创建、删除、修改、扩缩容等）
   - 包含操作时间、操作用户、操作对象、操作结果等信息
   - 可以输出到stdout或发送到审计日志系统

6. **事件记录**：
   - 使用Kubernetes Event记录中间件实例的状态变化
   - 事件类型：Normal、Warning
   - 事件原因：Created、Updated、Deleted、Scaled、Failed等

7. **诊断工具（优先级低）**：
   - 提供kubectl插件或Web界面诊断面板
   - 检查平台Operator的健康状态
   - 查看平台组件的资源使用情况（CPU、内存）
   - 检查平台与K8s API Server的连接状态
   - 查看平台的配置和运行状态

### 用户需求11：安全性需求

#### 需求详情

**TLS支持**：
1. 自动为中间件实例生成CA证书和各节点证书、密钥
2. 根据中间件实例的配置，自动生成中间件实例所需的其他TLS证书
3. 自动或手动更新TLS证书
4. Web界面支持HTTPS

**数据安全**：
- 敏感信息（如数据库密码、访问密钥）存放到K8s Secret
- 备份数据支持加密（由用户配置决定）
- 配置文件中的敏感信息通过环境变量或Volume挂载注入，或使用配置模板

**访问安全**：
- Web界面不需要认证，安全访问的功能由更上一级的平台提供

**合规性**：
- 暂不需要满足特定的安全合规要求（如等保、SOC2等）
- 暂不需要安全审计报告

#### 技术洞察

1. **TLS证书管理**：
   - 使用cert-manager或自研证书管理组件
   - 实现CA证书的自动生成和轮换
   - 支持为中间件实例自动签发证书
   - 支持证书的自动续签和手动更新
   - 证书和私钥存储在Kubernetes Secret中

2. **Web界面HTTPS**：
   - 使用Ingress配置TLS
   - 支持自动证书管理（如Let's Encrypt）或用户自签名证书
   - 支持TLS 1.2和TLS 1.3

3. **敏感信息存储**：
   - 使用Kubernetes Secret存储密码、密钥等敏感信息
   - 支持Secret的加密存储（K8s at-rest encryption）
   - 通过环境变量或Volume挂载注入到Pod
   - 使用配置模板中的占位符，运行时替换敏感信息

4. **备份加密**：
   - 支持对称加密（AES-256）和非对称加密（RSA）
   - 加密密钥可以存储在Kubernetes Secret中
   - 支持使用KMS（Key Management Service）管理密钥
   - 用户可以通过配置选择是否加密备份数据

5. **配置文件敏感信息处理**：
   - 使用环境变量注入：通过Pod的env或envFrom字段
   - 使用Volume挂载：将Secret挂载为文件
   - 使用配置模板：在模板中使用占位符，运行时替换
   - 避免在ConfigMap中直接存储敏感信息

6. **无认证依赖**：
   - Web界面本身不实现认证，依赖更上一级的平台
   - 通过反向代理或API Gateway实现认证
   - 可以传递用户身份信息到Web界面

7. **网络安全**：
   - 使用NetworkPolicy实现Pod间的网络隔离
   - 支持Service Mesh集成（可选）
   - 支持Pod Security Policy（K8s 1.25之前）或Pod Security Standards（K8s 1.25+）

8. **镜像安全**：
   - 支持镜像签名验证（如Notary、cosign）
   - 支持镜像漏洞扫描（可选）
   - 使用最小权限原则运行容器

9. **审计日志**：
   - 记录所有敏感操作（创建、删除、修改配置等）
   - 包含操作时间、操作用户、操作对象、操作结果等信息
   - 可以输出到stdout或发送到审计日志系统

