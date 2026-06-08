# Prometheus 标准化配置部署

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **目录先行** | 遵循 Prometheus 最佳实践，在 `/opt/` 下建立 `data/`、`rules/`、`targets/` 等运行目录，再部署配置文件。 |
| ② | **专用用户** | 使用系统用户 `prometheus` 运行三组件，数据目录 `750`、配置文件 `640`，最小权限。 |
| ③ | **规则在后章** | 告警规则 YAML 部署与 `promtool check rules` 见 [04-03](04-03-Prometheus-Rules-CN.md)；本章部署主配置并完成语法校验。 |

## 本章目标

在 [04-01](04-01-Prometheus-Package-CN.md) 二进制解压完成后，于 Prometheus 服务器（`10.210.65.174`）完成 **标准化目录结构**、**专用用户与权限**，并部署 `prometheus.yml` / `alertmanager.yml`。

| 项目 | 说明 |
| --- | --- |
| **目标节点** | `prometheus.example.com` · `10.210.65.174` |
| **配置源** | [04_prometheus.example.com](../../04_prometheus.example.com/) |
| **Prometheus 路径** | `/opt/prometheus` |
| **Alertmanager 路径** | `/opt/alertmanager` |
| **运行用户** | `prometheus`（系统用户） |

---

## Executive Summary: 配置步骤一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **Prometheus 目录** | `data` · `rules` · `config` · `targets` 等 | ✅ |
| 2 | **Alertmanager 目录** | `data` · `config` · `templates` | ✅ |
| 3 | **创建用户** | `groupadd` / `useradd` → `prometheus` | ✅ |
| 4 | **设置权限** | `chown` + `chmod` 755/750/640 | ✅ |
| 5 | **部署主配置** | 复制 `prometheus.yml` · `alertmanager.yml` | ✅ |
| 6 | **语法校验** | `promtool check config` · `amtool check-config` | ✅ |

---

## 1. 前置条件

| # | 条件 | 验证 |
| --- | --- | --- |
| 1 | [04-01](04-01-Prometheus-Package-CN.md) 完成 | `/opt/prometheus/prometheus` 等二进制存在 |
| 2 | root 或 sudo 权限 | 可创建用户与修改 `/opt` 权限 |
| 3 | 配置文件就绪 | 已从 DEMO 包或 [04_prometheus.example.com](../../04_prometheus.example.com/) 获取 yml |

---

## 2. 创建 Prometheus 目录结构

遵循 Prometheus 最佳实践，在二进制目录下创建运行所需子目录：

```bash
cd /opt/prometheus
mkdir -p data rules config consoles console_libraries targets

tree -L 1 /opt/prometheus
# 或
ls -l /opt/prometheus
```

| 目录 | 用途 |
| --- | --- |
| `data/` | TSDB 时序数据（**需足够磁盘空间**，见 02-02） |
| `rules/` | 告警与记录规则（[04-03](04-03-Prometheus-Rules-CN.md) 部署） |
| `config/` | 配置文件版本备份（回滚用） |
| `consoles/` | Web 控制台模板（可选） |
| `console_libraries/` | 控制台库（可选） |
| `targets/` | 基于文件的服务发现（可选扩展） |

---

## 3. 创建 Alertmanager 目录结构

```bash
cd /opt/alertmanager
mkdir -p data config templates

ls -l /opt/alertmanager
```

| 目录 | 用途 |
| --- | --- |
| `data/` | Alertmanager 状态与静默数据 |
| `config/` | 配置文件版本备份 |
| `templates/` | 告警通知模板（自定义格式） |

---

## 4. 创建专用系统用户

DEMO / 生产均建议使用专用用户运行服务：

```bash
groupadd --system prometheus
useradd -s /sbin/nologin --system -g prometheus prometheus

id prometheus
# 示例：uid=981(prometheus) gid=980(prometheus) groups=980(prometheus)
```

---

## 5. 设置目录权限与所有者

```bash
chown -R prometheus:prometheus /opt/prometheus
chown -R prometheus:prometheus /opt/alertmanager
chown -R prometheus:prometheus /opt/node_exporter

chmod 755 /opt/prometheus /opt/alertmanager /opt/node_exporter
chmod 750 /opt/prometheus/data
chmod 750 /opt/alertmanager/data

chmod 640 /opt/prometheus/prometheus.yml 2>/dev/null || true
chmod 640 /opt/alertmanager/alertmanager.yml 2>/dev/null || true

ls -ld /opt/prometheus /opt/alertmanager /opt/node_exporter
ls -ld /opt/prometheus/data /opt/alertmanager/data
```

| 对象 | 权限 | 说明 |
| --- | --- | --- |
| 基础目录 | `755` | 可执行文件对所有用户可执行 |
| 数据目录 | `750` | 仅 `prometheus` 用户/组可访问 |
| 配置文件 | `640` | 仅属主可写，组可读 |

---

## 6. 部署 Prometheus 主配置

```bash
# 在配置源目录执行（示例路径）
cd /path/to/04_prometheus.example.com

cp /opt/prometheus/prometheus.yml /opt/prometheus/config/prometheus.yml.original
cp prometheus.yml /opt/prometheus/prometheus.yml

chown prometheus:prometheus /opt/prometheus/prometheus.yml
chmod 640 /opt/prometheus/prometheus.yml
```

### 6.1 关键配置说明

| 区块 | DEMO 要点 |
| --- | --- |
| **global** | `scrape_interval` / `evaluation_interval` 15s；`external_labels` 含 `cluster: aiops-demo` |
| **alerting** | Alertmanager `localhost:9093`，API `v2` |
| **rule_files** | 引用 `/opt/prometheus/rules/*.yml`（规则文件在 [04-03](04-03-Prometheus-Rules-CN.md) 部署） |
| **scrape_configs** | `prometheus` · `alertmanager` · `node_exporter_prometheus`（`:9100`）· `node_exporter_chaos`（`10.210.65.148:9100`） |

### 6.2 验证 Prometheus 配置语法

> 若 `rules/` 下尚无规则文件，`promtool check config` 可能报警告；完整校验建议在 [04-03](04-03-Prometheus-Rules-CN.md) 复制规则后重跑。

```bash
cd /opt/prometheus
./promtool check config prometheus.yml
```

期望：

```
Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax
```

---

## 7. 部署 Alertmanager 配置

```bash
cd /path/to/04_prometheus.example.com

cp /opt/alertmanager/alertmanager.yml /opt/alertmanager/config/alertmanager.yml.original
cp alertmanager.yml /opt/alertmanager/alertmanager.yml

chown prometheus:prometheus /opt/alertmanager/alertmanager.yml
chmod 640 /opt/alertmanager/alertmanager.yml
```

### 7.1 关键配置说明

| 区块 | DEMO 要点 |
| --- | --- |
| **global** | `resolve_timeout: 5m` |
| **route** | 按 `severity` 路由；Critical/Warning → `EDA` receiver |
| **receivers** | `EDA` Webhook 指向 AAP EDA Event Stream（示例 `http://10.210.65.24:5000/alerts`，以实际为准） |
| **inhibit_rules** | Critical 抑制同实例 Warning；InstanceDown 抑制其他告警 |

### 7.2 验证 Alertmanager 配置

```bash
cd /opt/alertmanager
./amtool check-config alertmanager.yml
```

期望：`SUCCESS` 或等效通过信息。

---

## 配置后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | Prometheus 目录 | `data/` · `rules/` · `config/` 等已创建 |
| 2 | Alertmanager 目录 | `data/` · `config/` · `templates/` 已创建 |
| 3 | 用户 | `id prometheus` 正常 |
| 4 | 权限 | 数据目录 `750`，属主 `prometheus:prometheus` |
| 5 | prometheus.yml | 已部署；`promtool check config` 通过 |
| 6 | alertmanager.yml | 已部署；`amtool check-config` 通过 |

---

## 参考文档

- [04-01 Prometheus 软件包](04-01-Prometheus-Package-CN.md)
- [00-Prometheus 架构](../../04_prometheus.example.com/00-Prometheus架构.md)
- [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)

> **下一步**：[04-03 Prometheus 规则文件部署](04-03-Prometheus-Rules-CN.md) — 复制规则 YAML 并执行 `promtool check rules`
