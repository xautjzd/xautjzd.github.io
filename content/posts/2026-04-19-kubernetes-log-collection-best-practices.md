---
title: Kubernetes 集群应用日志采集最佳实践
date: "2026-04-19T20:00:00+08:00"
tags: ["kubernetes"]
---

## 1. Kubernetes 集群下应用日志采集面临的挑战

在传统的虚拟机或物理机环境中，应用日志通常以文件形式持久化在固定路径，日志采集 Agent 只需读取这些文件即可。但在 Kubernetes 环境下，日志管理面临一系列独特挑战。

### 1.1 容器的临时性

Pod 是 Kubernetes 中最小的调度单元，其生命周期具有高度不确定性：

- **弹性伸缩**：HPA 根据负载自动扩缩 Pod，Pod 随时可能被创建或销毁
- **滚动更新**：Deployment 更新时，旧版本 Pod 被逐个替换，历史日志随之消失
- **节点故障**：节点宕机时，Pod 被驱逐并在其他节点重建，原节点日志无法访问
- **资源回收**：节点磁盘压力触发 Eviction，Pod 被强制终止

若日志仅保存在容器内部，Pod 销毁意味着日志永久丢失，这对问题排查和合规审计都是不可接受的。

### 1.2 日志的分散性

一个中等规模的 K8s 集群可能有数百个节点、数千个 Pod，应用日志分散在每个节点的容器存储中：

- 日志物理位置随 Pod 调度动态变化，无法预知
- 跨节点查看日志需要逐台机器登录，效率极低
- 分布式调用链的日志需要跨节点聚合，才能完整还原请求路径

### 1.3 多租户隔离需求

企业级 K8s 平台通常服务多个业务团队，因此需要：

- 按 Namespace / Service 隔离不同团队的日志
- 支持细粒度的日志访问权限控制
- 按团队分摊和管控日志存储成本

### 1.4 传统方案的局限

| 传统方案 | 核心问题 | K8s 场景下的不足 |
|---------|---------|-----------------|
| SSH 登录节点查看 | 无法集中检索，效率低 | Pod 频繁漂移，难以定位日志位置 |
| 写入共享存储（NFS/EBS） | 需要应用改造，引入依赖 | 增加应用复杂度，不适合无状态服务 |
| 应用直推日志中心 | 与业务代码耦合 | 多语言/框架依赖重，难以统一运维 |

因此，需要一种**与业务解耦、集中化、可扩展**的日志采集方案。

## 2. 日志采集架构模式对比

Kubernetes 社区演进出了三种主流的日志采集架构模式，各有其适用场景。

### 2.1 DaemonSet 模式（节点级采集）

**架构原理**：在每个 K8s 节点上以 DaemonSet 形式部署一个日志采集 Agent，通过挂载节点上的容器日志目录（`/var/log/containers`），统一采集该节点上所有 Pod 的标准输出日志。

```
┌─────────────────────────────────────────────────────────┐
│                         Node                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Pod A     │  │   Pod B     │  │   Pod C     │      │
│  │  stdout/err │  │  stdout/err │  │  stdout/err │      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │
│         └────────────────┴────────────────┘             │
│                          │ /var/log/containers/         │
│                          ▼                              │
│       ┌──────────────────────────────────────┐          │
│       │        Fluent Bit DaemonSet          │          │
│       │  读取日志 → 注入 K8s 元数据 → 推送       │          │
│       └──────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

**优点**：

- 资源开销极低：50 节点集群仅需 50 个 Agent Pod
- 对应用完全无侵入：无需修改业务代码或镜像
- 运维简单：DaemonSet 统一升级配置，管理成本低

**缺点**：

- 仅支持 stdout/stderr，无法直接采集容器内的日志文件
- 若应用将日志写入文件，需要额外挂载 EmptyDir / HostPath 卷

**适用场景**：大多数通用场景，尤其适合遵循 12-Factor 原则（日志输出到 stdout）的微服务应用。

### 2.2 Sidecar 模式（Pod 级采集）

**架构原理**：在业务 Pod 中注入一个日志采集 Sidecar 容器，与应用容器通过共享卷（EmptyDir）交换日志，由 Sidecar 独立完成采集与推送。

```
┌──────────────────────────────────────────────────────┐
│                         Pod                          │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │    App Container    │  │   Sidecar (Fluent Bit) │ │
│  │                     │  │                        │ │
│  │  写入 /app/logs/*.log│  │  读取共享卷 → 推送日志    │ │
│  └──────────┬──────────┘  └────────────────────────┘ │
│             │                        ▲               │
│             └────── EmptyDir 共享卷 ──┘               │
└──────────────────────────────────────────────────────┘
```

**优点**：

- 隔离性强：每个 Pod 独立采集，单个 Sidecar 故障不影响其他 Pod
- 配置灵活：可按应用自定义解析规则和标签
- 支持文件日志：适合将日志写入文件的遗留应用

**缺点**：

- 资源开销大：1000 个 Pod 需要 1000 个 Sidecar，CPU/内存消耗显著
- 生命周期耦合：Sidecar 异常可能导致 Pod 启动或运行异常

**适用场景**：多租户强隔离场景、遗留应用（日志写入文件）、需要独立采集策略的核心业务系统。

### 2.3 应用直推模式

**架构原理**：应用代码直接集成日志 SDK（如 Logback Loki Appender、Fluent Logger），通过网络将日志直接发送到日志中心，无需采集代理。

**优点**：

- 实时性最高：无落盘延迟，日志产生即发送
- 结构化友好：可直接发送 JSON 格式，无需二次解析
- 精准控制：应用自主决定采样率、日志级别过滤

**缺点**：

- 与代码强耦合：需引入 SDK，多语言栈维护成本高
- 网络依赖：发送失败可能影响业务性能（需引入异步缓冲）
- 接入改造成本高：每个应用都需单独接入

**适用场景**：对日志实时性有极致要求的金融交易系统、已有统一日志 SDK 基础设施的大型平台。

### 2.4 三种模式横向对比

| 维度 | DaemonSet | Sidecar | 应用直推 |
|------|:---------:|:-------:|:-------:|
| 资源占用 | 低（节点级） | 高（Pod 级） | 中（应用级） |
| 应用侵入性 | 无 | 低 | 高 |
| 配置灵活性 | 中 | 高 | 高 |
| 运维复杂度 | 低 | 中 | 高 |
| 支持日志类型 | stdout/stderr | 文件 + stdout | 任意 |
| 生产推荐度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

> **结论**：对于大多数场景，**DaemonSet 模式是首选**，资源利用率最优且对业务零侵入。对于确有文件日志需求的场景，可考虑 Sidecar 模式作为补充。

## 3. 技术选型

日志采集方案由**采集层 → 传输层 → 存储层 → 展示层**四层组成。

### 3.1 采集层选型

| 工具 | 语言 | 内存占用 | 特点 |
|------|------|---------|------|
| **Fluent Bit** | C | ~1MB | 云原生首选，轻量高效，插件丰富 |
| Fluentd | Ruby | ~100MB | 功能全面，生态成熟，资源占用较高 |
| **Vector** | Rust | 极低 | 新兴高性能采集器，数据转换能力强 |
| Promtail | Go | 低 | 专为 Loki 设计，配置简洁 |
| Logstash | JVM | 高（JVM） | ELK 传统方案，功能强但重资源 |

**推荐**：**Fluent Bit**。C 语言实现，资源占用极低，原生支持 Loki、Elasticsearch、Kafka、S3 等多种输出目标，是当前云原生场景的事实标准。

### 3.2 存储层选型

| 方案 | 存储成本 | 查询能力 | 扩展性 | 适用场景 |
|------|:-------:|:-------:|:------:|---------|
| **Loki** | 低（对象存储） | LogQL 查询 | 高 | K8s 原生，成本敏感型首选 |
| Elasticsearch | 高（全索引） | 极强（全文检索） | 中 | 搜索密集型，预算充足 |
| ClickHouse | 中 | 强（列式分析） | 高 | 日志分析、OLAP 场景 |
| S3 + Athena | 极低 | 中（查询延迟高） | 极高 | 合规归档，低频访问 |

**推荐**：**Loki**。采用轻量级索引设计（只索引标签，不索引日志内容），存储成本仅为 Elasticsearch 的 1/5 ~ 1/10，且与 Grafana 深度集成，是 K8s 日志存储的首选方案。

### 3.3 展示层选型

- **Grafana**：与 Loki / Prometheus 深度集成，支持 LogQL 查询、告警规则、可视化仪表盘，是 K8s 可观测性的标准选择
- **Kibana**：Elasticsearch 原生配套，功能全面，但资源占用较大
- **自建平台**：基于 K8s API 和日志存储查询接口自研，适合有定制化需求的大型平台

## 4. 推荐方案：Fluent Bit + Loki + Grafana

基于上述分析，推荐采用 **Fluent Bit（采集）+ Loki（存储）+ Grafana（展示）** 的全栈方案。

### 4.1 整体架构

```
┌────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                          │
│                                                                    │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │
│  │    Node 1     │  │    Node 2     │  │    Node N     │           │
│  │ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │           │
│  │ │Fluent Bit │ │  │ │Fluent Bit │ │  │ │Fluent Bit │ │           │
│  │ │DaemonSet  │ │  │ │DaemonSet  │ │  │ │DaemonSet  │ │           │
│  │ └─────┬─────┘ │  │ └─────┬─────┘ │  │ └─────┬─────┘ │           │
│  └───────┼───────┘  └───────┼───────┘  └───────┼───────┘           │
│          └─────────────────►│◄─────────────────┘                   │
│                             │  HTTP Push                           │
│                             ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                         Loki Stack                           │  │
│  │                                                              │  │
│  │  ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐    │  │
│  │  │ Distributor │──►│   Ingester  │──►│   Object Store   │    │  │
│  │  │  接收日志流  │   │  构建索引     │   │  (S3/GCS/MinIO)  │    │  │
│  │  └─────────────┘   └─────────────┘   └──────────────────┘    │  │
│  │                                                              │  │
│  │  ┌─────────────┐   ┌──────────────────┐                      │  │
│  │  │   Querier   │◄──│  Query Frontend  │◄── Grafana / LogCLI  │  │
│  │  │   日志查询   │   │    查询分发        │                      │  │
│  │  └─────────────┘   └──────────────────┘                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### 4.2 方案优势

- **云原生血统**：Fluent Bit 和 Loki 均为 CNCF 项目，与 K8s 生态无缝融合
- **成本优势显著**：Loki 使用对象存储（S3/MinIO）作为后端，长期存储成本远低于 Elasticsearch
- **统一可观测体验**：与 Prometheus + Grafana 形成指标 + 日志统一视图，团队学习成本低
- **轻量高效**：Fluent Bit 单节点内存占用仅数 MB，对业务节点几乎无感知

## 5. 生产部署指南

### 5.1 部署 Loki（分布式模式）

对于 50 节点以上的生产集群，推荐部署**分布式 Loki**（Loki Distributed 模式）以获得更好的性能和可用性。

**helm-values.yaml**：

```yaml
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 2
  storage:
    type: s3
    s3:
      region: ap-southeast-1
      bucketnames: mycompany-loki-logs
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: index_
          period: 24h
  limits_config:
    retention_period: 720h     # 日志保留 30 天
    ingestion_rate_mb: 64
    ingestion_burst_size_mb: 128

distributor:
  replicas: 2

ingester:
  replicas: 3
  persistence:
    enabled: true
    size: 50Gi

querier:
  replicas: 2
  persistence:
    enabled: true
    size: 10Gi

queryFrontend:
  replicas: 2

gateway:
  enabled: true
  replicas: 2
```

**部署命令**：

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --create-namespace \
  -f helm-values.yaml
```

### 5.2 部署 Fluent Bit（DaemonSet）

**Step 1：创建 ConfigMap**

```yaml
# fluent-bit-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: monitoring
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush             5
        Daemon            Off
        Log_Level         info
        Parsers_File      parsers.conf
        HTTP_Server       On
        HTTP_Listen       0.0.0.0
        HTTP_Port         2020
        storage.type      filesystem          # 启用持久化缓冲，防止重启丢日志
        storage.path      /var/fluent-bit/state

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            containerd
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        DB                /var/fluent-bit/state/tail.db   # 记录读取位置，防止重复采集

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On
        Annotations         Off
        Labels              On

    [FILTER]
        Name                nest
        Match               kube.*
        Operation           lift
        Nested_under        kubernetes
        Add_prefix          k8s_

    [FILTER]
        Name                modify
        Match               kube.*
        Rename              k8s_pod_name        pod
        Rename              k8s_namespace_name  namespace
        Rename              k8s_container_name  container
        Rename              k8s_node_name       node
        Add                 cluster             prod-k8s-01
        Remove_regex        k8s_.*

    [OUTPUT]
        Name            loki
        Match           kube.*
        Host            loki-gateway.monitoring.svc
        Port            80
        Labels          job=fluentbit,cluster=$cluster,namespace=$namespace,pod=$pod,container=$container
        Line_Format     json
        Retry_Limit     5
        Batch_Wait      1s
        Batch_Size      1MB

  parsers.conf: |
    [PARSER]
        Name        containerd
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On
```

**Step 2：创建 RBAC 与 DaemonSet**

```yaml
# fluent-bit-daemonset.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
  - apiGroups: [""]
    resources: [pods, namespaces, nodes]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: monitoring
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: monitoring
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
        prometheus.io/path: "/api/v1/metrics/prometheus"
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists   # 确保在所有节点（包括 master）上运行
      containers:
        - name: fluent-bit
          image: cr.fluentbit.io/fluent/fluent-bit:3.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 2020
              name: http-metrics
          resources:
            limits:
              cpu: 500m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 64Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc/
            - name: fluent-bit-state
              mountPath: /var/fluent-bit/state
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config
        - name: fluent-bit-state
          hostPath:
            path: /var/fluent-bit/state
            type: DirectoryOrCreate
```

**Step 3：部署**

```bash
kubectl apply -f fluent-bit-config.yaml
kubectl apply -f fluent-bit-daemonset.yaml

# 验证部署状态
kubectl get daemonset fluent-bit -n monitoring
kubectl logs -n monitoring -l app=fluent-bit --tail=50
```

### 5.3 配置 Grafana 数据源

```yaml
# grafana-loki-datasource.yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: loki
  namespace: monitoring
spec:
  datasource:
    name: Loki
    type: loki
    url: http://loki-gateway.monitoring.svc:80
    access: proxy
    isDefault: false
    jsonData:
      maxLines: 1000
      derivedFields:
        - name: TraceID
          matcherRegex: '"trace_id":"(\w+)"'
          url: "${__value.raw}"
          datasourceUid: tempo   # 关联 Tempo 实现日志 → Trace 跳转
```

常用 LogQL 查询示例：

```logql
# 查询某命名空间下所有容器的错误日志
{namespace="production", container="api"} |= "ERROR"

# 过滤 JSON 日志中的特定字段
{namespace="production"} | json | level="error" | line_format "{{.message}}"

# 按 Pod 聚合日志量（近 5 分钟）
sum by (pod) (rate({namespace="production"}[5m]))
```

## 6. 高级实践

### 6.1 多行日志合并（Java 异常堆栈）

Java 异常堆栈跨多行，需要配置多行解析，否则每一行都会作为独立日志条目上报：

```ini
[INPUT]
    Name              tail
    Tag               java.*
    Path              /var/log/containers/*-java-*.log
    Multiline.Parser  java   # 使用内置 Java 多行解析器

# 如需自定义规则
[MULTILINE_PARSER]
    Name           java-custom
    Type           regex
    Flush_Timeout  5000
    # 以日期时间开头为新日志条目的起始行
    Rule           "start_state"  "/^\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}/"   "cont"
    Rule           "cont"         "/^\s+at.*/"                                  "cont"
    Rule           "cont"         "/^Caused by:.*/"                             "cont"
    Rule           "cont"         "/^\s+\.\.\. \d+ more/"                       "cont"
```

### 6.2 日志脱敏（手机号、身份证）

生产环境必须对日志中的敏感信息进行脱敏处理，避免 PII 数据泄露：

```ini
[FILTER]
    Name    lua
    Match   kube.*
    script  /fluent-bit/etc/sanitize.lua
    call    sanitize_log
```

`sanitize.lua`：

```lua
function sanitize_log(tag, timestamp, record)
    local log = record["log"]
    if log then
        -- 手机号脱敏：138****8888
        log = string.gsub(log, "([1][3-9]%d%d)%d%d%d%d(%d%d%d%d)", "%1****%2")
        -- 身份证号脱敏：110101********1234
        log = string.gsub(log, "(%d%d%d%d%d%d)%d%d%d%d%d%d%d%d(%d%d%d[%dXx])", "%1********%2")
        -- 银行卡号脱敏（保留前4后4位）
        log = string.gsub(log, "(%d%d%d%d)%d%d%d%d%d%d%d%d(%d%d%d%d)", "%1 **** **** %2")
        record["log"] = log
    end
    return 1, timestamp, record
end
```

### 6.3 通过 Pod 注解控制采集行为

```yaml
# 禁用指定 Pod 的日志采集
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug-app
spec:
  template:
    metadata:
      annotations:
        fluentbit.io/exclude: "true"

# 为 Pod 指定自定义日志解析器
---
metadata:
  annotations:
    fluentbit.io/parser: "nginx"
```

Fluent Bit 对应过滤配置：

```ini
[FILTER]
    Name                grep
    Match               kube.*
    Exclude             $kubernetes['annotations']['fluentbit.io/exclude'] true
```

### 6.4 高吞吐量性能调优

对于日志量大（>10MB/s per node）的场景，重点调整以下参数：

```ini
[SERVICE]
    storage.type            filesystem    # 持久化缓冲，防止 OOM 导致日志丢失
    storage.max_chunks_up   128           # 内存中最大 chunk 数

[INPUT]
    Buffer_Chunk_Size  2MB      # 单文件读取 chunk 大小
    Buffer_Max_Size    10MB     # 单条日志最大长度（防止超长日志截断）
    Refresh_Interval   2        # 文件监听刷新频率（秒）

[OUTPUT]
    Batch_Wait    500ms     # 批次等待时间，减少请求数
    Batch_Size    4MB       # 批次大小，提升吞吐量
    Retry_Limit   10        # 重试次数，保障可靠性
    Workers       2         # 并发输出 worker 数
```

### 6.5 日志分级存储策略

合理的存储策略可大幅降低长期存储成本：

| 存储层 | 保留时长 | 存储介质 | 适用场景 |
|--------|---------|---------|---------|
| 热存储 | 7 天 | SSD（Ingester WAL） | 实时查询、问题排查 |
| 温存储 | 30 天 | 对象存储（S3 Standard） | 近期审计、回溯分析 |
| 冷归档 | 180 天 | 对象存储（S3 Glacier） | 合规留存、低频访问 |

```yaml
# Loki 分级存储配置示例
loki:
  storage_config:
    aws:
      s3: s3://mybucket/loki
  compactor:
    retention_enabled: true
  limits_config:
    retention_period: 720h    # 热/温存储保留 30 天
```

## 7. 监控与排障

### 7.1 日志丢失排查指南

| 现象 | 可能原因 | 排查与解决方法 |
|------|---------|--------------|
| 特定 Pod 无日志 | 该 Pod 注解配置了 exclude | 检查 Pod annotations，确认未被过滤 |
| 节点上所有 Pod 缺失日志 | 节点磁盘满，Fluent Bit 无法读取日志 | `df -h` 检查节点磁盘，配置 logrotate 或扩容 |
| 特定时间段日志缺失 | Fluent Bit 重启，内存缓冲丢失 | 检查是否启用 `storage.type filesystem` 持久化缓冲 |
| 日志延迟过高（>30s） | Loki Ingester 过载或网络问题 | 检查 Fluent Bit 输出队列深度和 Loki 摄入速率 |
| 多行日志被拆分 | 未配置多行解析器 | 为对应容器配置 Multiline Parser |

### 7.2 关键 Prometheus 监控指标

```promql
# Fluent Bit：输出重试率（过高说明 Loki 端有压力）
rate(fluentbit_output_retried_records_total[5m])

# Fluent Bit：各输出目标的错误计数
fluentbit_output_errors_total

# Loki：日志摄入速率（lines/s）
sum(rate(loki_distributor_lines_received_total[5m]))

# Loki：摄入延迟 P99
histogram_quantile(0.99,
  sum(rate(loki_distributor_ingester_appends_latency_seconds_bucket[5m])) by (le)
)

# Loki：查询延迟 P99
histogram_quantile(0.99,
  sum(rate(loki_querier_request_duration_seconds_bucket[5m])) by (le)
)
```

### 7.3 Loki 查询性能优化

```logql
# ✅ 好的查询：使用精确标签缩小搜索范围
{namespace="production", container="order-service"} |= "ERROR" | json

# ❌ 差的查询：标签选择器过宽，匹配流过多
{job="fluentbit"} |= "ERROR"

# ✅ 避免高基数标签（如 user_id、trace_id 不应作为 Loki 标签）
# 应通过日志内容过滤，而非标签过滤
{namespace="production"} | json | trace_id="abc123"
```

**黄金原则**：

1. Loki 标签应保持**低基数**（< 1000 个唯一值），高基数字段放在日志内容中用 `| json` 过滤
2. 查询时间范围尽量精确，避免全量扫描
3. 生产环境查询加上 `limit` 限制返回条数

## 8. 总结

### 架构选型建议

- **默认选 DaemonSet 模式**：适用于 90% 的场景，资源利用率最优，对业务零侵入
- **文件日志用 Sidecar 补充**：仅在遗留应用或强隔离需求时引入，避免资源浪费
- **Fluent Bit + Loki + Grafana**：在成本、性能、运维复杂度之间达到最佳平衡，是中小规模集群的推荐组合

### 实施关键点

| 分类 | 要点 |
|------|------|
| 可靠性 | 启用 `storage.type filesystem` 持久化缓冲，配合 `tail DB` 记录读取位点 |
| 安全性 | 配置日志脱敏 Lua 脚本，防止 PII 数据写入日志中心 |
| 成本控制 | 制定分级存储策略（7 天热 + 30 天温 + 180 天冷归档） |
| 标签规范 | 统一标签命名（cluster / namespace / pod / container），避免高基数标签 |
| 可观测性 | 部署 Fluent Bit 监控指标，配置采集延迟和重试率告警 |

### 演进路径

对于更大规模的集群（500+ 节点），可考虑以下演进方向：

- 引入 **Kafka** 作为日志传输缓冲层，解耦采集与存储，应对流量峰值
- 使用 **Vector** 替代 Fluent Bit，利用其更强的数据路由和转换能力
- 结合 **OpenTelemetry Collector** 统一处理 Logs / Metrics / Traces，构建全面的可观测性平台

## 参考资料

- [Fluent Bit 官方文档](https://docs.fluentbit.io)
- [Loki 官方文档](https://grafana.com/docs/loki)
- [Kubernetes 官方日志架构文档](https://kubernetes.io/docs/concept/cluster-administration/logging/)
