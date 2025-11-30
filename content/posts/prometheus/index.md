---
title: "监控系统 Prometheus"
date: 2025-11-30
tags: ["Go", "Monitoring", "Prometheus", "Time Series Database"]
---

作为一名后端工程师，当业务上线后，持续监控服务的健康状态与性能指标是必不可少的环节。比如，监控 API 的请求延迟、错误率，数据库的连接数与查询性能，服务器的 CPU 和内存使用情况等，都是保障系统稳定运行的重要指标。在现代软件系统中，监控是保证可靠性、性能和可观测性的基础。通过监控，团队可以：

- 实时发现问题并快速定位故障来源；
- 监控服务的性能与资源使用以便于容量规划；
- 把技术指标与业务指标结合，评估功能对用户与业务的影响；
- 驱动自动化响应（告警、自动缩放、回滚等）。

从技术/服务角度常关注的指标包括请求延迟、错误率、吞吐量（QPS）、CPU/内存/磁盘/网络等；从业务角度关注 PV/UV、转化率、订单率等关键业务指标。一个健全的监控体系应当同时覆盖基础设施指标与业务指标，并支持告警、可视化与长期查询。

## 传统监控方式

要判断线上接口的状态，最简单的方式就是记录日志，比如http状态码、响应时间等，然后通过日志分析工具进行离线统计与分析。这就是传统的系统级日志监控的逻辑，比如 ELK（Elasticsearch + Logstash + Kibana）方案：
> 应用输出日志 → Logstash 解析 → ES 索引 → Kibana 可视化查询

- Logstash 负责收集与解析日志；
- Elasticsearch 负责存储与索引日志数据；
- Kibana 提供可视化与查询界面。

```mermaid
flowchart LR
    subgraph LogSource[日志源/业务系统]
        A1[应用服务日志]
        A2[容器 stdout/stderr]
        A3[数据库日志]
        A4[系统日志]
    end

    subgraph Agent[轻量级采集 Agent]
        B1[Filebeat/Metricbeat]
        B2[自定义日志上报程序]
    end

    subgraph LogstashLayer[Logstash]
        C1[Input: tcp/http/beats/file/...]
        C2[Filter: grok/json/mutate/date/geoip/...]
        C3[Output: Elasticsearch/Kafka/文件/...]
    end

    subgraph Storage[存储 & 可视化]
        D1[Elasticsearch]
        D2[Kibana]
        D3[长期归档系统（可选）]
    end

    %% 数据流
    A1 --> B1
    A2 --> B1
    A3 --> B1
    A4 --> B1

    B1 --> C1
    C1 --> C2
    C2 --> C3
    C3 --> D1
    D1 --> D2
    C3 --> D3

```

但这些方法存在：

- 实时性差（离线处理导致响应延迟）；
- 可扩展性与维护复杂度高（大量脚本与管道）；
- 难以直接计算延迟分位数或高效聚合跨实例指标。

Prometheus 及其生态通过高效的时序存储、拉取式采集、多维标签模型与强大的查询语言（PromQL）解决了这些问题。

## Prometheus

Prometheus 是一个开源监控系统与时间序列数据库（TSDB），主要特性：

- 多维数据模型（metric name + labels）；
- 强大的查询语言 PromQL；
- 本地高效 TSDB 与块存储设计；
- 默认采用拉取（pull）采集，并通过 PushGateway 支持短期作业的推送；
- 与 Alertmanager 集成处理告警路由、抑制与通知；
- 丰富生态：node_exporter、cadvisor、Grafana、Thanos、Cortex 等。

Prometheus 非常适合微服务與容器化环境，并常与 Grafana 配合用于可视化。

### Prometheus 系统架构概览

一个典型的 Prometheus 系统包括：

| 分类        | 组件                                        | 用途               |
| --------- | ----------------------------------------- | ---------------- |
| **指标采集层** | Exporter、Client SDK、Pushgateway           | 采集系统和应用指标        |
| **监控核心层** | Prometheus Server、Service Discovery       | 抓取、存储、查询指标       |
| **告警层**   | Alertmanager                              | 管理与发送告警          |
| **可视化层**  | Grafana                                   | 展示数据图表、Dashboard |
| **扩展层**   | Thanos / Cortex / Mimir / VictoriaMetrics | 海量数据、长时存储、HA     |

![prometheus系统架构图](./images/image.png)

首先，exporter 和 client SDK 负责将系统或应用的指标以 http 接口的形式暴露出来；Prometheus Server 定期拉取这些指标并存储在本地 TSDB 中；通过 PromQL 查询数据并生成图表或触发告警；Alertmanager 负责处理告警的路由与通知；Grafana 用于可视化展示监控数据；Thanos 或 Cortex 等组件用于扩展存储与查询能力。

> exporter：负责将系统或应用的指标以 Prometheus 格式暴露出来的组件，如 `node_exporter`（主机指标）、`blackbox_exporter`（网络探测）等。
> client SDK：应用程序中集成的库，用于定义和暴露自定义指标。
> Pushgateway：用于短期批处理作业推送指标的组件。

### Prometheus 数据模型与指标类型

**指标数据模型**主要由四部分组成：

```txt
例如：http_requests_total{job="api-server",instance="server1:8080",method="GET",code="200"} 1027
```

- **指标名称（Metric Name）**：标识具体的监控指标，如 `http_requests_total` 表示 HTTP 请求总数。
- **标签（Labels）**：键值对形式的附加信息，用于区分同一指标的不同维度，如 `method="GET"`、`code="200"` 等。
- **指标值（Value）**：具体的数值，如请求次数、延迟时间等。
- **时间戳（Timestamp）**：指标被 Prometheus Server 采集的时间点。

**Metric** 类型主要有四种：

| Metric 类型     | 特点                                  | 适用场景              | 优缺点                                          |
| ------------- | ----------------------------------- | ----------------- | -------------------------------------------- |
| **Counter**   | 单调递增，只能加不能减                         | 请求次数、错误次数、任务完成数   | ✔ 易懂稳定  ✘ 无法表示当前值                          |
| **Gauge**     | 可增可减，表示瞬时状态                         | 内存、CPU、队列长度、并发连接数 | ✔ 灵活  ✘ 容易被误用（短周期波动大）                      |
| **Histogram** | 将观测值按桶（bucket）分布统计，提供 `count`、`sum` | 请求延迟、请求大小等分布      | ✔ 可计算分位数（通过 PromQL）  ✔ 适合跨实例聚合  ✘ 需提前规划桶 |
| **Summary**   | 客户端侧计算分位数                           | 单机延迟分位数           | ✔ 本地实时分位数  ✘ 无法跨实例聚合，不推荐在分布式场景使用       |

上述所有指标计数结果均保存在服务的**内存**中，不会持久化到磁盘，Prometheus Server 定期拉取这些指标并存储在本地 TSDB 中。

/metrics接口暴露指标示例：

```text
# ===========================
# Counter 示例
# ===========================
# HELP http_requests_total 总请求数
# TYPE http_requests_total counter
http_requests_total{job="api-server",instance="server1:8080",method="GET",code="200"} 1027

# ===========================
# Gauge 示例
# ===========================
# HELP active_connections 当前活跃连接数
# TYPE active_connections gauge
active_connections{job="api-server",instance="server1:8080"} 134

# ===========================
# Histogram 示例
# ===========================
# HELP request_duration_seconds 请求延迟分布
# TYPE request_duration_seconds histogram
request_duration_seconds_bucket{job="api-server",instance="server1:8080",le="0.1"} 523
request_duration_seconds_bucket{job="api-server",instance="server1:8080",le="0.2"} 874
request_duration_seconds_bucket{job="api-server",instance="server1:8080",le="0.5"} 1192
request_duration_seconds_bucket{job="api-server",instance="server1:8080",le="1"} 1460
request_duration_seconds_bucket{job="api-server",instance="server1:8080",le="+Inf"} 1530
request_duration_seconds_sum{job="api-server",instance="server1:8080"} 247.39
request_duration_seconds_count{job="api-server",instance="server1:8080"} 1530

# ===========================
# Summary 示例
# ===========================
# HELP request_duration_summary_seconds 请求延迟分位数（客户端统计）
# TYPE request_duration_summary_seconds summary
request_duration_summary_seconds{job="api-server",instance="server1:8080",quantile="0.5"} 0.123
request_duration_summary_seconds{job="api-server",instance="server1:8080",quantile="0.9"} 0.245
request_duration_summary_seconds{job="api-server",instance="server1:8080",quantile="0.99"} 0.487
request_duration_summary_seconds_sum{job="api-server",instance="server1:8080"} 310.23
request_duration_summary_seconds_count{job="api-server",instance="server1:8080"} 1820
```

### 时序数据库 TSDB

Prometheus Server 从各个 Exporter 拉取指标数据后，存储在本地的时序数据库（TSDB）中。Prometheus 的 TSDB 设计目标是高效存储与查询时间序列数据，主要特点：

- **块存储（Block Storage）**：数据按时间块（block）存储，每个块包含若干时间序列数据，默认块大小为 2 小时。块内数据采用列式存储与压缩算法（如 Gorilla 压缩）以节省空间。
- **索引结构**：使用倒排索引（inverted index）加速标签查询，支持高效的标签过滤与聚合操作。
- **数据保留策略**：支持配置数据的保留时间（如 15 天），过期数据会被自动删除以节省存储空间。
- **查询优化**：通过内存缓存、预计算等手段提升查询性能。

### PromQL 查询语言

pv计算，QPS计算，分位数计算等。



### 可视化与告警 Grafana、Alertmanager


## Prometheus 实践指南

TODO： 简化流程并实践

下面把常见的安装、接入、PromQL 示例、告警、服务发现、安全与运维建议放在同一节，便于阅读与实践。


### 安装与快速上手

常见部署方式：

- 本地二进制（学习/单机）：下载官方 release，运行 `./prometheus --config.file=prometheus.yml`；
- Docker：

```bash
docker run -p 9090:9090 \
    -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus:latest
```

- Kubernetes：使用 Helm Chart（如 `kube-prometheus-stack`）或 Operator 部署完整监控栈。


示例 `prometheus.yml`（最简单的 scrape 配置）：

```yaml
global:
    scrape_interval: 15s

scrape_configs:
    - job_name: 'prometheus'
        static_configs:
            - targets: ['localhost:9090']

    - job_name: 'node'
        static_configs:
            - targets: ['host1:9100', 'host2:9100']
```


### 应用接入示例（Go）

最小 Go 示例（使用 `client_golang`）暴露指标：

```go
package main

import (
        "net/http"
        "time"
        "github.com/prometheus/client_golang/prometheus"
        "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
        reqs = prometheus.NewCounterVec(prometheus.CounterOpts{Name: "http_requests_total"}, []string{"path", "method", "code"})
        latency = prometheus.NewHistogram(prometheus.HistogramOpts{Name: "http_request_duration_seconds", Buckets: prometheus.DefBuckets})
)

func init() { prometheus.MustRegister(reqs, latency) }

func main() {
        http.Handle("/metrics", promhttp.Handler())
        // your handlers...
        http.ListenAndServe(":8080", nil)
}
```

启动后，Prometheus 抓取 `http://<host>:8080/metrics` 即可采集指标。


### 服务发现与 relabeling

在动态环境中（例如 Kubernetes），Prometheus 支持多种服务发现方式（`kubernetes_sd_configs`, `consul_sd_configs`, `file_sd_configs` 等），并通过 `relabel_configs` 在抓取前对目标或 labels 做过滤、替换或删除：

示例（Kubernetes）：

```yaml
scrape_configs:
    - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
            - role: pod
        relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                action: keep
                regex: true
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
                target_label: __metrics_path__
```


### PromQL 常用示例

- 计算 5 分钟内每秒请求速率（QPS）：

```promql
sum(rate(http_requests_total[5m]))
```

- 计算 95 百分位响应时间（基于 histogram）：

```promql
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

- 某实例 CPU 使用率（node_exporter）：

```promql
(1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100
```

使用 recording rules 把复杂或开销大的查询提前计算为新的 time series，可以显著提升查询性能。


### 告警规则与 Alertmanager

示例告警规则：

```yaml
groups:
- name: example
    rules:
    - alert: HighRequestLatency
        expr: histogram_quantile(0.9, sum by (le)(rate(http_request_duration_seconds_bucket[5m]))) > 0.5
        for: 2m
        labels:
            severity: page
        annotations:
            summary: "高请求延迟 (实例 {{ $labels.instance }})"
```

Prometheus 将触发的告警发送到 Alertmanager，Alertmanager 负责抑制、分组、路由及通知（邮件/Slack/钉钉等）。


### PushGateway（短期作业）

对于短期批处理作业，PushGateway 可用于推送指标：

```bash
echo "my_job_metric 42" | curl --data-binary @- http://pushgateway:9091/metrics/job/my_job
```

注意：PushGateway 保存的是最新值，不适合作为所有长期指标的替代方案。


### 安全与访问控制

- 指标端点应放在受限网络或通过反向代理（带认证与 TLS）暴露；
- `prometheus.yml` 支持 `basic_auth`、`tls_config`：

```yaml
scrape_configs:
    - job_name: secure_app
        static_configs:
            - targets: ['10.0.0.5:9100']
        basic_auth:
            username: prometheus
            password: s3cr3t
        tls_config:
            ca_file: /etc/prometheus/ca.crt
```


### 长期存储与扩展

当需要长期保存指标或跨集群聚合查询时，可采用 Thanos 或 Cortex：

- Thanos：通过 sidecar 上传本地块到对象存储并提供全局查询层；
- Cortex：采用可扩展后端进行写入与查询，适合多租户场景。


### 可视化与调优建议

- 使用 Grafana 导入现有 Dashboard 或自定义面板；常用 PromQL 如 `sum(rate(http_requests_total[5m])) by (job)`。
- 排错流程：先访问目标 `/metrics` 检查输出 -> Prometheus UI `Status -> Targets` 查看抓取错误 -> 使用 `promtool check config` 验证配置。
- 控制 label 基数，使用 recording rules、合理安排 `scrape_interval` 来减轻查询与抓取负载。


## 更细化技术细节

### 几种不同的计数器的区别

### 时序数据库的存储与查询

### 常用的promQL查询


## 参考与延申阅读

- 官方文档：https://prometheus.io/docs/
- PromQL 教程：https://prometheus.io/docs/prometheus/latest/querying/basics/
- Thanos / Cortex：长期存储与扩展方案


