# Prometheus 软件包下载及解压

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **二进制部署** | AIOps DEMO 采用官方 **tar.gz 二进制**安装 Prometheus / Alertmanager / Node Exporter，解压至 `/opt/` 标准化目录。 |
| ② | **监控栈同机** | Prometheus Server 同机部署 **Prometheus + Alertmanager + Node Exporter**；被管节点（如 Chaos）单独部署 Node Exporter。 |
| ③ | **配置在后章** | 本章仅完成下载与解压；`prometheus.yml`、告警规则、systemd 见 [04-02](04-02-Prometheus-Config-CN.md)～[04-05](04-05-Prometheus-Verify-CN.md)。 |

## 本章目标

在 Prometheus 监控节点上，下载并解压 Prometheus 生态组件至标准目录，验证二进制与目录结构就绪，为后续配置与 systemd 服务做准备。

| 项目 | 说明 |
| --- | --- |
| **架构参考** | [00-Prometheus 架构](../../04_prometheus.example.com/00-Prometheus架构.md) · [Prometheus 官方概述](https://prometheus.io/docs/introduction/overview/) |
| **Prometheus 节点** | `prometheus.example.com` · `10.210.65.174` · RHEL 9.5 |
| **被管示例（Chaos）** | `Chaos.example.com` · `10.210.65.148` · RHEL 9.4 · 仅 Node Exporter |
| **安装路径** | `/opt/prometheus` · `/opt/alertmanager` · `/opt/node_exporter` |

### 软件版本（DEMO 冻结）

| 组件 | 版本 | 安装路径 |
| --- | --- | --- |
| **Prometheus** | 3.5.1 | `/opt/prometheus` |
| **Alertmanager** | 0.31.1 | `/opt/alertmanager` |
| **Node Exporter** | 1.10.2 | `/opt/node_exporter` |

---

## Executive Summary: 部署步骤一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **下载软件包** | 从 GitHub Releases 获取 3 个 tar.gz | ✅ |
| 2 | **校验完整性** | `sha256sum`（可选但推荐） | 建议 |
| 3 | **解压至 /opt** | `--strip-components=1` 标准化目录 | ✅ |
| 4 | **验证目录结构** | 确认可执行文件与默认 yml 存在 | ✅ |

---

## 1. 架构与环境说明

AIOps DEMO 监控拓扑（详见 [02-02 资源规划](../02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md)）：

| 角色 | 主机名 | IP | 操作系统 | 部署组件 |
| --- | --- | --- | --- | --- |
| **Prometheus 服务器** | `prometheus.example.com` | `10.210.65.174` | RHEL 9.5 | Prometheus · Alertmanager · Node Exporter |
| **被管服务器（Chaos）** | `Chaos.example.com` | `10.210.65.148` | RHEL 9.4 | Node Exporter |

> Prometheus 与 Alertmanager 的配置文件内容见后续章节 [04-02 标准化配置](04-02-Prometheus-Config-CN.md) 及 [04-03 告警规则](04-03-Prometheus-Rules-CN.md)。

---

## 2. 软件包下载

在 **Prometheus 服务器**（`10.210.65.174`）上以 root 或具备 sudo 的用户执行：

```bash
mkdir -p /tmp/prometheus_install && cd /tmp/prometheus_install

# Prometheus 3.5.1
wget https://github.com/prometheus/prometheus/releases/download/v3.5.1/prometheus-3.5.1.linux-amd64.tar.gz

# Alertmanager 0.31.1
wget https://github.com/prometheus/alertmanager/releases/download/v0.31.1/alertmanager-0.31.1.linux-amd64.tar.gz

# Node Exporter 1.10.2
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```

| 组件 | 下载 URL |
| --- | --- |
| Prometheus | https://github.com/prometheus/prometheus/releases/download/v3.5.1/prometheus-3.5.1.linux-amd64.tar.gz |
| Alertmanager | https://github.com/prometheus/alertmanager/releases/download/v0.31.1/alertmanager-0.31.1.linux-amd64.tar.gz |
| Node Exporter | https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz |

> 离线环境可将上述 tar.gz 预先拷贝至 `/tmp/prometheus_install/` 后跳过 `wget`。

---

## 3. 校验下载文件（可选但推荐）

```bash
cd /tmp/prometheus_install
sha256sum *.tar.gz
```

与 [GitHub Releases](https://github.com/prometheus/prometheus/releases) 页面公布的 SHA256 核对。校验失败应重新下载，避免损坏包导致后续异常。

---

## 4. 解压至标准化目录

```bash
mkdir -p /opt/prometheus /opt/alertmanager /opt/node_exporter

tar -xzf /tmp/prometheus_install/prometheus-3.5.1.linux-amd64.tar.gz \
  -C /opt/prometheus --strip-components=1

tar -xzf /tmp/prometheus_install/alertmanager-0.31.1.linux-amd64.tar.gz \
  -C /opt/alertmanager --strip-components=1

tar -xzf /tmp/prometheus_install/node_exporter-1.10.2.linux-amd64.tar.gz \
  -C /opt/node_exporter --strip-components=1
```

快速列出目录：

```bash
ls -l /opt/prometheus
ls -l /opt/alertmanager
ls -l /opt/node_exporter
```

---

## 5. 验证安装目录结构

```bash
cd /opt && pwd

# Alertmanager
ls -lh /opt/alertmanager
# 期望：
#   alertmanager      (可执行文件)
#   alertmanager.yml  (默认配置，后续在 04-02 替换)
#   amtool            (管理工具)

# Node Exporter
ls -lh /opt/node_exporter
# 期望：
#   node_exporter     (可执行文件)

# Prometheus
ls -lh /opt/prometheus
# 期望：
#   prometheus        (主程序)
#   promtool          (配置验证工具)
#   prometheus.yml    (默认配置，后续在 04-02 替换)
```

可选：验证二进制可执行：

```bash
/opt/prometheus/prometheus --version
/opt/alertmanager/alertmanager --version
/opt/node_exporter/node_exporter --version
```

---

## 部署前检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | 下载目录 | `/tmp/prometheus_install/` 含 3 个 tar.gz |
| 2 | Prometheus | `/opt/prometheus/prometheus` 存在且可执行 |
| 3 | Alertmanager | `/opt/alertmanager/alertmanager` 存在 |
| 4 | Node Exporter | `/opt/node_exporter/node_exporter` 存在 |
| 5 | 默认配置 | `prometheus.yml` / `alertmanager.yml` 存在（待 04-02 替换） |
| 6 | 磁盘空间 | `/opt` 所在分区满足时序数据需求（见 02-02） |

---

## 参考文档

- [00-Prometheus 架构](../../04_prometheus.example.com/00-Prometheus架构.md)
- [02-02 资源规划](../02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md)
- [Prometheus Installation](https://prometheus.io/docs/prometheus/latest/installation/)

> **下一步**：[04-02 Prometheus 标准化配置](04-02-Prometheus-Config-CN.md) — 替换 `prometheus.yml` / `alertmanager.yml` 并配置 scrape 目标
