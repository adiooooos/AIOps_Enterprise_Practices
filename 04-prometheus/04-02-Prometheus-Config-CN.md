# Prometheus 标准化配置部署

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **标准化配置部署** | 本章**只做标准化配置部署**：仅创建标准化目录结构、专用用户与权限；不含 `prometheus.yml` / 规则文件部署。 |
| ② | **最佳实践** | 遵循 Prometheus 官方建议，在 `/opt/` 下分离 `data/`、`rules/`、`config/` 等目录。 |
| ③ | **后续章节** | 告警规则与主配置文件见 [04-03](04-03-Prometheus-Rules-CN.md) 及后续章节。 |

## 本章目标

在 [04-01](04-01-Prometheus-Package-CN.md) 二进制解压完成后，于 Prometheus 服务器（`10.210.65.174`）执行 **Step 5：创建标准化目录结构**（5.1～5.4）。

| 项目 | 说明 |
| --- | --- |
| **目标节点** | `prometheus.example.com` · `10.210.65.174` |
| **Prometheus 路径** | `/opt/prometheus` |
| **Alertmanager 路径** | `/opt/alertmanager` |
| **Node Exporter 路径** | `/opt/node_exporter` |
| **运行用户** | `prometheus`（系统用户） |

---

## Executive Summary: Step 5 一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 5.1 | **Prometheus 目录** | `mkdir -p data rules config consoles console_libraries targets` | ✅ |
| 5.2 | **Alertmanager 目录** | `mkdir -p data config templates` | ✅ |
| 5.3 | **创建专用用户** | `groupadd` / `useradd` → `prometheus` | ✅ |
| 5.4 | **设置权限** | `chown` + `chmod` 755/750/640 | ✅ |

---

## 1. 前置条件

| # | 条件 | 验证 |
| --- | --- | --- |
| 1 | [04-01](04-01-Prometheus-Package-CN.md) 完成 | `/opt/prometheus`、`/opt/alertmanager`、`/opt/node_exporter` 二进制已解压 |
| 2 | root 或 sudo 权限 | 可创建系统用户与修改 `/opt` 权限 |

---

## Step 5：创建标准化目录结构

> 遵循 Prometheus 最佳实践，创建标准化的目录结构。

### 5.1 创建 Prometheus 目录结构

```bash
# 创建 Prometheus 运行所需目录
cd /opt/prometheus
mkdir -p data rules config consoles console_libraries targets

# 验证目录结构
tree -L 1 /opt/prometheus
# 或者使用 ls -l
ls -l /opt/prometheus
```

**目录说明：**

| 目录 | 说明 |
| --- | --- |
| `data/` | Prometheus TSDB 时序数据库存储目录（**重要：需要足够的磁盘空间**） |
| `rules/` | 告警和记录规则文件目录 |
| `config/` | 配置文件版本备份目录（用于配置回滚） |
| `consoles/` | Web 控制台模板目录（可选） |
| `console_libraries/` | 控制台库文件目录（可选） |
| `targets/` | 基于文件的服务发现配置目录（用于动态目标发现） |

### 5.2 创建 Alertmanager 目录结构

```bash
# 创建 Alertmanager 运行所需目录
cd /opt/alertmanager
mkdir -p data config templates

# 验证目录结构
ls -l /opt/alertmanager
```

**目录说明：**

| 目录 | 说明 |
| --- | --- |
| `data/` | Alertmanager 状态和静默数据存储目录 |
| `config/` | 配置文件版本备份目录 |
| `templates/` | 告警通知模板目录（用于自定义告警格式） |

### 5.3 创建专用用户（推荐用于生产环境）

```bash
# 创建 prometheus 系统用户和组
groupadd --system prometheus
useradd -s /sbin/nologin --system -g prometheus prometheus

# 验证用户创建
id prometheus
# 预期输出：uid=xxx(prometheus) gid=xxx(prometheus) groups=xxx(prometheus)
# uid=981(prometheus) gid=980(prometheus) groups=980(prometheus)
```

### 5.4 设置目录权限和所有者

```bash
# 设置目录所有者为 prometheus 用户
chown -R prometheus:prometheus /opt/prometheus
chown -R prometheus:prometheus /opt/alertmanager
chown -R prometheus:prometheus /opt/node_exporter

# 设置基础目录权限
chmod 755 /opt/prometheus /opt/alertmanager /opt/node_exporter

# 设置数据目录权限（更严格的权限）
chmod 750 /opt/prometheus/data
chmod 750 /opt/alertmanager/data

# 设置配置文件权限（防止未授权修改）
chmod 640 /opt/prometheus/prometheus.yml 2>/dev/null || true
chmod 640 /opt/alertmanager/alertmanager.yml 2>/dev/null || true

# 验证权限
ls -ld /opt/prometheus /opt/alertmanager /opt/node_exporter
ls -ld /opt/prometheus/data /opt/alertmanager/data
```

**安全最佳实践说明：**

- 使用专用系统用户运行服务，遵循最小权限原则
- 数据目录权限 **750**（仅 prometheus 用户和组可访问）
- 配置文件权限 **640**（仅 prometheus 用户可读写，组可读）
- 可执行文件权限 **755**（所有用户可执行）

---

## 配置后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | Prometheus 目录 | `data/` · `rules/` · `config/` · `consoles/` · `console_libraries/` · `targets/` 已创建 |
| 2 | Alertmanager 目录 | `data/` · `config/` · `templates/` 已创建 |
| 3 | 用户 | `id prometheus` 输出正常（uid/gid 示例见 5.3） |
| 4 | 属主 | `/opt/prometheus`、`/opt/alertmanager`、`/opt/node_exporter` 属主为 `prometheus:prometheus` |
| 5 | 数据目录权限 | `/opt/prometheus/data`、`/opt/alertmanager/data` 为 `750` |
| 6 | 基础目录权限 | `/opt/prometheus`、`/opt/alertmanager`、`/opt/node_exporter` 为 `755` |

---

## 参考文档

- [04-01 Prometheus 软件包](04-01-Prometheus-Package-CN.md)
- [00-Prometheus 架构](../../04_prometheus.example.com/00-Prometheus架构.md)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)

> **下一步**：[04-03 Prometheus 规则文件部署](04-03-Prometheus-Rules-CN.md)
