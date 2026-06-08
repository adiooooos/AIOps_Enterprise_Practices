# Prometheus 规则与配置文件部署

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **规则文件部署** | 在 [04-02](04-02-Prometheus-Config-CN.md) 目录与用户就绪后，部署告警规则、`prometheus.yml` / `alertmanager.yml`，并明确 Node Exporter 启动参数。 |
| ② | **配置源** | 规则与主配置来自 [04_prometheus.example.com](../../04_prometheus.example.com/) 目录。 |
| ③ | **校验先行** | 使用 `promtool` / `amtool` 验证语法后再进入 [04-04 systemd](04-04-Prometheus-Systemd-CN.md) 启服。 |

## 本章目标

在 Prometheus 服务器（`10.210.65.174`）完成 **规则文件部署**：复制告警规则、部署 Prometheus / Alertmanager 主配置、校验配置，并记录 Node Exporter 推荐启动参数。

| 项目 | 说明 |
| --- | --- |
| **前置章节** | [04-02 Step 5](04-02-Prometheus-Config-CN.md) 目录与用户权限 |
| **配置源目录** | `04_prometheus.example.com/` |
| **规则目录** | `/opt/prometheus/rules/` |
| **Prometheus 配置** | `/opt/prometheus/prometheus.yml` |
| **Alertmanager 配置** | `/opt/alertmanager/alertmanager.yml` |

---

## Executive Summary: Step 6～8 一览

| Step | 内容 | 必须 |
| --- | --- | --- |
| **6.1** | 复制 4 个告警规则 YAML | ✅ |
| **6.2** | 部署 `prometheus.yml` | ✅ |
| **6.3** | `promtool check config` | ✅ |
| **6.4** | `promtool check rules` | ✅ |
| **6.5** | 单规则文件测试（可选） | 建议 |
| **7.1** | 部署 `alertmanager.yml` | ✅ |
| **7.2** | `amtool check-config` | ✅ |
| **7.3** | 路由测试（可选） | 建议 |
| **8.1～8.2** | Node Exporter 采集器与启动参数说明 | ✅ |

---

## 1. 前置条件

| # | 条件 | 验证 |
| --- | --- | --- |
| 1 | [04-02](04-02-Prometheus-Config-CN.md) 完成 | `/opt/prometheus/rules/` 等目录存在 |
| 2 | 配置包就位 | `04_prometheus.example.com/` 含规则与 yml |
| 3 | 工作目录 | 复制命令在配置源目录执行 |

```bash
cd /path/to/04_prometheus.example.com
```

---

## Step 6：部署 Prometheus 告警规则与主配置

### 6.1 复制告警规则文件

将本目录下的规则文件复制到 Prometheus 的 `rules` 目录：

```bash
# 假设当前在配置文件目录（04_prometheus.example.com/）
# 复制所有告警规则文件
cp 10-Prometheus规则文件/linux_filesystem_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_network_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_performance_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_service_alerts.yml /opt/prometheus/rules/

# 设置规则文件权限
chown prometheus:prometheus /opt/prometheus/rules/*.yml
chmod 774 /opt/prometheus/rules/*.yml

# 验证规则文件
ls -lh /opt/prometheus/rules/
```

**告警规则文件说明：**

| 文件 | 说明 |
| --- | --- |
| `linux_filesystem_alerts.yml` | 文件系统相关告警（磁盘使用率、inode 使用率等） |
| `linux_network_alerts.yml` | 网络相关告警（网络错误、丢包率等） |
| `linux_performance_alerts.yml` | 系统性能告警（CPU、内存、负载等） |
| `linux_service_alerts.yml` | 服务可用性告警（节点宕机、Exporter 不可用等） |

### 6.2 部署 Prometheus 主配置文件

```bash
# 备份原始配置文件
cp /opt/prometheus/prometheus.yml /opt/prometheus/config/prometheus.yml.original

# 复制优化后的配置文件
cp prometheus.yml /opt/prometheus/prometheus.yml

# 设置配置文件权限
chown prometheus:prometheus /opt/prometheus/prometheus.yml
chmod 770 /opt/prometheus/prometheus.yml
```

**Prometheus 配置文件关键配置说明：**

| 区块 | 要点 |
| --- | --- |
| **1. global** | `scrape_interval: 15s` · `evaluation_interval: 15s` · `scrape_timeout: 10s` · `external_labels`（联邦/远程存储） |
| **2. alerting** | Alertmanager `localhost:9093` · API `v2` |
| **3. rule_files** | 引用所有告警规则（绝对路径 `/opt/prometheus/rules/*.yml`） |
| **4. scrape_configs** | `prometheus` · `alertmanager` · `node_exporter_prometheus`（本机 `:9100`）· `node_exporter_chaos`（`10.210.65.148:9100`） |

### 6.3 验证 Prometheus 配置

```bash
cd /opt/prometheus
./promtool check config prometheus.yml
```

**预期输出：**

```
Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax
```

### 6.4 验证告警规则文件

```bash
cd /opt/prometheus
./promtool check rules rules/*.yml
```

**预期输出示例：**

```
Checking rules/linux_filesystem_alerts.yml
 SUCCESS: 7 rules found

Checking rules/linux_network_alerts.yml
 SUCCESS: 7 rules found

Checking rules/linux_performance_alerts.yml
 SUCCESS: 8 rules found

Checking rules/linux_service_alerts.yml
 SUCCESS: 5 rules found
```

### 6.5 测试单个规则文件（可选）

```bash
./promtool check rules rules/linux_performance_alerts.yml
```

---

## Step 7：部署 Alertmanager 配置

### 7.1 部署 Alertmanager 配置文件

```bash
# 备份原始配置文件
cp /opt/alertmanager/alertmanager.yml /opt/alertmanager/config/alertmanager.yml.original

# 复制优化后的配置文件
cp alertmanager.yml /opt/alertmanager/alertmanager.yml

# 设置配置文件权限
chown prometheus:prometheus /opt/alertmanager/alertmanager.yml
chmod 770 /opt/alertmanager/alertmanager.yml
```

**Alertmanager 配置文件关键配置说明：**

| 区块 | 要点 |
| --- | --- |
| **1. global** | `resolve_timeout: 5m` |
| **2. route** | `group_by: ['alertname', 'cluster', 'service', 'instance']` · `group_wait: 10s` · `group_interval: 10s` · `repeat_interval: 12h` |
| **route 子路由** | Critical：5s 等待，3h 重复 · Warning：10s / 12h · Info/Unknown：24h |
| **3. receivers** | `default`（低优先级）· `EDA` Webhook → `http://10.210.65.24:5000/alerts`，`send_resolved: true` |
| **4. inhibit_rules** | Critical 抑制同实例 Warning · InstanceDown 抑制该实例其他告警 |

### 7.2 验证 Alertmanager 配置

```bash
cd /opt/alertmanager
./amtool check-config alertmanager.yml
```

**预期输出：**

```
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 2 inhibit rules
 - 2 receivers
 - 0 templates
```

### 7.3 测试 Alertmanager 配置（可选）

```bash
./amtool config routes --config.file=alertmanager.yml

./amtool config routes test \
  --config.file=alertmanager.yml \
  --tree \
  alertname=HighCPUUsage severity=critical
```

**预期输出：**

```
Routing tree:
└── default-route  receiver: default
    └── {severity="critical"}  receiver: EDA
```

---

## Step 8：配置 Node Exporter

Node Exporter 通过启动参数进行配置，**无需配置文件**。

### 8.1 Node Exporter 采集器说明

**默认启用的采集器（推荐保留）：**

| 采集器 | 说明 |
| --- | --- |
| `cpu` | CPU 统计信息 |
| `diskstats` | 磁盘 I/O 统计 |
| `filesystem` | 文件系统使用情况 |
| `loadavg` | 系统负载 |
| `meminfo` | 内存使用情况 |
| `netdev` | 网络设备统计 |
| `netstat` | 网络连接统计 |
| `stat` | 系统统计信息 |
| `time` | 系统时间 |
| `uname` | 系统信息 |

**常用可选采集器：**

| 采集器 | 启用方式 |
| --- | --- |
| `systemd` | `--collector.systemd` |
| `processes` | `--collector.processes` |
| `tcpstat` | `--collector.tcpstat` |

**推荐的过滤配置：**

- 排除虚拟文件系统挂载点（避免采集 `/proc`、`/sys` 等）
- 排除 Docker/容器网络接口（减少无用指标）

### 8.2 Node Exporter 启动参数

Node Exporter 将通过 **systemd 服务**启动，参数配置在服务文件中（见 [04-04](04-04-Prometheus-Systemd-CN.md)）：

```bash
--web.listen-address=:9100
--collector.filesystem.mount-points-exclude="^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)"
--collector.netclass.ignored-devices="^(veth.*|br-.*|docker.*)$"
--collector.netdev.device-exclude="^(veth.*|br-.*|docker.*)$"
```

| 参数 | 说明 |
| --- | --- |
| `--web.listen-address=:9100` | 监听端口 9100 |
| `--collector.filesystem.mount-points-exclude` | 排除虚拟文件系统挂载点 |
| `--collector.netclass.ignored-devices` | 排除虚拟网络设备（容器网桥） |
| `--collector.netdev.device-exclude` | 排除虚拟网络接口 |

---

## 配置后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | 规则文件 | `/opt/prometheus/rules/` 含 4 个 yml |
| 2 | prometheus.yml | 已部署；`promtool check config` SUCCESS |
| 3 | 规则语法 | `promtool check rules` 全部 SUCCESS |
| 4 | alertmanager.yml | 已部署；`amtool check-config` SUCCESS |
| 5 | EDA 路由 | Critical 告警路由至 `EDA` receiver（7.3 可选验证） |
| 6 | Node Exporter | 启动参数已记录，待 04-04 写入 systemd |

---

## 参考文档

- [04-02 Prometheus 标准化配置](04-02-Prometheus-Config-CN.md)
- [04_prometheus.example.com 配置包](../../04_prometheus.example.com/)
- [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Node Exporter Collectors](https://github.com/prometheus/node_exporter#collectors)

> **下一步**：[04-04 Prometheus systemd 服务配置](04-04-Prometheus-Systemd-CN.md) — Step 9 创建并启用 systemd 单元
