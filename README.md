# 🏠 Homelab Observability Stack

> 可直接业务化的主机监控与日志告警基线方案，基于 Docker Compose 一键部署 **Metrics + Logs + Alerting + Visualization**。

---

## 📋 目录

- [项目概述](#-项目概述)
- [架构设计](#-架构设计)
- [技术栈](#-技术栈)
- [目录结构](#-目录结构)
- [快速开始](#-快速开始)
- [核心配置解析](#-核心配置解析)
- [告警体系](#-告警体系)
- [Dashboard 展示](#-dashboard-展示)

---

## 🎯 项目概述

本项目是一套**开箱即用的可观测性基线**，通过 Docker Compose 部署完整的监控告警链路：

| 能力 | 组件 | 说明 |
|---|---|---|
| **指标采集** | Prometheus + Node Exporter | 系统级 CPU / 内存 / 磁盘 / 网络指标 |
| **日志聚合** | Loki + Promtail | 轻量级日志收集，与 Grafana 深度集成 |
| **可视化** | Grafana | 预置 Node Exporter Full & Loki Dashboard，启动即加载 |
| **告警通知** | Alertmanager + 钉钉 Webhook | 分级告警（Warning / Critical），支持告警抑制 |

**设计要点：**
- 📦 **全容器化**：所有组件基于官方镜像，版本锁定
- 🔧 **配置即代码**：Prometheus / Loki / Alertmanager / Grafana 数据源与看板全部通过 YAML 声明式管理
- 🚨 **分级告警**：CPU、内存、磁盘设置 Warning / Critical 两级阈值，Critical 自动抑制 Warning
- 📱 **钉钉集成**：告警直达钉钉群，支持自定义 Markdown 模板
- 📊 **Provisioning 自动加载**：Dashboard 与数据源随 Grafana 启动自动导入，无需手动操作

---

## 🏗️ 架构设计
![架构设计](docs/images/Obserability.drawio.png)

**数据流说明：**
- **Metrics 链路**：Node Exporter 暴露 `:9100/metrics` → Prometheus 主动抓取 → Grafana 查询展示
- **Logs 链路**：Promtail 直接挂载宿主机 `/var/log` 采集日志 → 推送至 Loki → Grafana 查询展示
- **Alert 链路**：Prometheus 触发告警 → Alertmanager 路由/抑制 → DingTalk Webhook 推送

**网络设计：**
- 统一使用 `observe` Bridge 网络（子网 `10.90.0.0/24`），组件间通过服务名 DNS 解析通信
- 仅暴露必要端口：`9090` (Prometheus)、`3000` (Grafana)、`9093` (Alertmanager)、`3100` (Loki)、`8060` (Dingtalk)

---

## 🛠️ 技术栈

| 组件 | 版本 | 角色 |
|---|---|---|
| [Prometheus](https://prometheus.io/) | v2.51.2 | 时序数据库 & 告警引擎 |
| [Grafana](https://grafana.com/) | 10.4.2 | 可视化 & Dashboard |
| [Loki](https://grafana.com/oss/loki/) | 3.4.1 | 日志聚合系统 |
| [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/) | 3.4.1 | 日志采集 Agent |
| [Node Exporter](https://github.com/prometheus/node_exporter) | v1.7.0 | 主机指标暴露器 |
| [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) | v0.27.0 | 告警路由 & 去重 |
| [prometheus-webhook-dingtalk](https://github.com/timonwong/prometheus-webhook-dingtalk) | v2.1.0 | 钉钉告警适配器 |
| Docker Compose | 3.8 | 编排工具 |

---

## 📁 目录结构

```
.
├── docker-compose.yml          # 全栈编排定义
├── README.md                   # 项目文档（本文件）
│
├── prometheus.yml              # Prometheus 抓取配置 & 告警接入
├── alert_rules.yml             # Prometheus 告警规则（CPU/内存/磁盘/只读）
├── node-randon.yml             # 文件服务发现配置（预留）
│
├── alertmanager.yml            # Alertmanager 路由 & 抑制规则
├── dingtalk.yml                # 钉钉 Webhook 配置
├── dingtalk.tmpl               # 钉钉消息 Markdown 模板源文件
│
├── loki-config.yaml            # Loki 存储 & 索引配置
├── promtail-config.yaml        # Promtail 日志抓取配置
│
└── grafana/
    └── provisioning/
        ├── dashboards/
        │   ├── dashboard.yml              # Dashboard 自动加载声明
        │   ├── 1860_rev45.json            # Node Exporter Full (社区版)
        │   └── Loki-Dashboard-fixed.json  # Loki 日志看板
        └── datasources/
            └── datasource.yml             # 数据源自动加载声明
```

---

## 🚀 快速开始

### 1. 前置要求

- Docker Engine >= 20.10
- Docker Compose >= 2.0
- Linux 宿主机（CentOS / Ubuntu / Debian 均可）

### 2. 一键启动

```bash
# 克隆仓库
git clone https://github.com/wamudus/node-observability-baseline.git
cd node-observability-baseline

# 在dingtalk.yml中设置你的token
templates:
  - /etc/prometheus-webhook-dingtalk/dingtalk.tmpl
targets:
  example:
    url: "https://oapi.dingtalk.com/robot/send?access_token=< your token >"
    message:
      title: '{{ template "dingtalk.default.title" . }}'
      text: '{{ template "dingtalk.default.content" . }}'

# 启动全栈（首次会自动拉取镜像）
docker compose up -d

# 查看状态
docker compose ps
```

### 3. 访问服务

| 服务 | 地址 | 默认账号 |
|---|---|---|
| Grafana | http://localhost:3000 | admin / admin |
| Prometheus | http://localhost:9090 | - |
| Alertmanager | http://localhost:9093 | - |
| Loki | http://localhost:3100 | - |

### 4. Dashboard 与数据源

Grafana 通过 **Provisioning** 自动加载以下内容，无需手动导入：

- **数据源**：Prometheus (`http://prometheus:9090`) + Loki (`http://loki:3100`)
- **Node Exporter Full**：`grafana/provisioning/dashboards/1860_rev45.json`
- **Loki 日志看板**：`grafana/provisioning/dashboards/Loki-Dashboard-fixed.json`

---

## ⚙️ 核心配置解析

### Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
        labels:
          app: "node-exporter"

  - job_name: "loki"
    static_configs:
      - targets: ["loki:3100"]
```

**设计要点：**
- Node Exporter 通过 Docker 服务名 `node-exporter:9100` 内网通信，无需宿主机端口暴露
- 预留 `file_sd_configs` 注释，支持大规模场景下通过文件动态发现目标

### Loki 日志采集

```yaml
# promtail-config.yaml
scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/{messages,secure,cron,dnf.log}
```

- 采集系统级日志：`messages`（内核/系统）、`secure`（安全/SSH）、`cron`（定时任务）、`dnf.log`（包管理）
- 日志标签统一为 `job=varlogs`，与 Grafana Loki Dashboard 查询语句对齐
- **宿主机直接挂载**：Promtail 容器通过 `- /var/log:/var/log:ro` 直接读取宿主机日志目录，不经过 Node Exporter

---

## 🚨 告警体系

### 告警规则概览

| 告警名称 | 表达式 | 阈值 | 持续时间 | 级别 |
|---|---|---|---|---|
| CPUHighLoad | `1 - avg(rate(node_cpu_idle[5m]))` | > 85% | 5m | warning |
| MemoryLow | `1 - MemAvailable / MemTotal` | > 90% | 2m | critical |
| DiskFull | `1 - avail / size (mountpoint="/")` | > 85% | 1m | warning |
| DiskCritical | 同上 | > 90% | 1m | critical |
| DiskReadOnly | `node_filesystem_readonly == 1` | == 1 | 0m | critical |

### 告警抑制（Inhibition）

```yaml
inhibit_rules:
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: [alertname, dev, instance]
```

**作用**：当同一实例的同一告警触发 Critical 时，自动静默其 Warning 级别通知，避免告警风暴。

### 钉钉通知模板

支持 Markdown 格式，单条消息聚合多告警：

```markdown
**告警名称**: CPUHighLoad
**实例**: node-exporter:9100
**状态**: firing
**详情**: 主机 node-exporter:9100 CPU 非空闲占比超过 85%，持续 5 分钟。当前值: 87.3%
```

---

## 📊 Dashboard 展示

### 1. Node Exporter Full (Grafana ID: 1860)

> 直接采用社区最成熟的主机监控 Dashboard，通过 Provisioning 自动加载：

- **Quick CPU / Mem / Disk**：CPU 使用率、系统负载、内存、Swap、根分区占用（Gauge + Bar）
- **Basic CPU / Mem / Net / Disk**：时序趋势图，支持多维度下钻
- **Memory Meminfo / Vmstat**：内核级内存细节（Slab、PageTables、HugePages、OOM Killer）
- **Storage / Network / Systemd**：磁盘 I/O、网络包级分析、systemd 服务状态
- **Hardware**：温度传感器、风扇转速、电源状态

### 2. Loki 日志看板

> 借鉴社区 Dashboard 导入原理，针对 datasource 变量化进行微调，实现多环境切换：

| Panel | 类型 | 说明 |
|---|---|---|
| 总日志速率 | Time Series | 普通日志 vs 错误日志 5m 速率对比 |
| 各文件日志量 | Bar Gauge | 按 filename 聚合的日志速率 |
| 原始错误日志 | Logs | 实时 error 日志流，支持详情展开 |
| 24小时错误总数 | Stat | 近 24h 累计错误条数，带阈值变色 |

**优化点**：
- 摒弃 `__inputs` 导入映射，改为 `datasource` 查询变量，顶部下拉框自动发现所有 Loki 源
- `job` 标签变量化，从硬编码 `varlogs` 改为 `$job`，适配多环境日志源

---

---

## 🩹 踩坑记录

### 1. 钉钉告警模板配置（`dingtalk.yml`）

`prometheus-webhook-dingtalk` 的 `message` 字段是**对象**，不是字符串。正确写法：

```yaml
templates:
  - /etc/prometheus-webhook-dingtalk/dingtalk.tmpl

targets:
  example:
    url: "https://oapi.dingtalk.com/robot/send?access_token=..."
    message:
      title: '{{ template "dingtalk.default.title" . }}'
      text: '{{ template "dingtalk.default.content" . }}'
```

**常见错误**：
- `title` / `message` 写成 target 的顶级字段 → 报错 `field title not found`
- `message` 写成字符串 → 报错 `cannot unmarshal !!str into config.plain`
- 模板文件名和挂载路径不一致 → Docker 静默创建目录，模板加载失败

**模板引擎限制**：`prometheus-webhook-dingtalk` 使用原生 Go template，**不支持** Alertmanager 的扩展函数（如 `humanizeDuration`、`since`）。
### 2. Grafana 启动时 Loki 未就绪

`depends_on` 只保证启动顺序，不保证服务就绪。Grafana 启动时若 Loki 尚未监听 3100，数据源会被标记为 down，刷新 Dashboard 也无数据。

**解决**：给 Loki 添加 `healthcheck`，让 Grafana 等其 Ready 再启动；或启动后手动 `docker compose restart grafana`。

### 3. 容器时区不一致

默认容器使用 UTC，与宿主机（CST）差 8 小时，导致告警时间戳、日志时间轴错乱。

**解决**：所有服务统一挂载宿主机时区并设置 `TZ` 环境变量：

```yaml
environment:
  - TZ=Asia/Shanghai
volumes:
  - /etc/localtime:/etc/localtime:ro
```

---

## 📄 许可证

MIT License © 2024
