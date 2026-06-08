# AAP Server 部署

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **安装入口** | 在 [03-02](03-02-AAP-Preparation-CN.md) 解压的 bundle 目录中编辑 `inventory-growth`，以 **`admin`** 用户执行 `ansible-playbook` 完成 Container Growth 拓扑安装。 |
| ② | **离线 bundle** | DEMO 使用 `bundle_install=true` 与本地 `bundle/` 目录，无需安装时拉取外部镜像。 |
| ③ | **MCP 同机部署** | AIOps DEMO 启用 `[ansiblemcp]` 组，与 Gateway / Controller / Hub / EDA 同机，供后续 AI Agent 集成（见 [03-05](03-05-AAP-MCP-For-Agent-CN.md)）。 |

## 本章目标

在主机准备就绪且离线 bundle 已解压后，完成 AAP 2.6 **容器化 Growth 拓扑**的安装——配置 `inventory-growth`、执行安装 playbook、验证各组件容器就绪，再进入 [03-04 配置与激活](03-04-AAP-Configuration-CN.md)。

| 项目 | 说明 |
| --- | --- |
| **适用版本** | Ansible Automation Platform **2.6**（Containerized） |
| **部署拓扑** | [Container Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies) · 单 VM |
| **Inventory 文件** | `inventory-growth` |
| **示例节点** | `aap26.example.com` · `10.210.65.24` |
| **执行用户** | `admin` |
| **工作目录** | `/home/admin/ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64` |

---

## Executive Summary: 部署步骤一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **进入 bundle 目录** | `cd` 至解压后的安装器目录 | ✅ |
| 2 | **编辑 inventory** | 按 DEMO 节点填写 `inventory-growth` | ✅ |
| 3 | **确认离线 bundle** | `bundle_install=true`，`bundle_dir` 指向 `./bundle` | ✅ |
| 4 | **执行安装** | `ansible-playbook -i inventory-growth ansible.containerized_installer.install` | ✅ |
| 5 | **验证容器** | `podman ps` 检查 Gateway / Controller / Hub / EDA / MCP 等 | ✅ |
| 6 | **访问 UI** | 浏览器打开 `https://<FQDN>` | ✅ |

---

## 1. 前置确认

开始安装前，确认 [03-02 检查清单](03-02-AAP-Preparation-CN.md#部署前检查清单) 已全部通过：

```bash
su - admin
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64
ls inventory-growth bundle/
```

| 检查项 | 期望 |
| --- | --- |
| `inventory-growth` | 存在且可编辑 |
| `bundle/` | 含离线镜像与依赖 |
| `ansible-playbook` | `ansible --version` 可用 |
| FQDN | `hostname -f` → `aap26.example.com` |

---

## 2. 配置 `inventory-growth`

在 bundle 目录中编辑 `inventory-growth`。以下为 AIOps DEMO **Container Growth 单节点**完整示例（基于 bundle `2.6-7`）。

### 2.1 主机分组（全部指向同一节点）

```ini
# Container Growth — 单 VM 拓扑
# 文档：https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies

[automationgateway]
aap26.example.com

[automationcontroller]
aap26.example.com

[automationhub]
aap26.example.com

[automationeda]
aap26.example.com

[database]
aap26.example.com

# AIOps DEMO：启用 Ansible MCP Server（同机）
[ansiblemcp]
aap26.example.com

# 可选组件（DEMO 默认注释）
# [ansiblelightspeed]
# aap26.example.com
```

### 2.2 全局与数据库变量

```ini
[all:vars]
# Ansible 连接
ansible_connection=local
validate_certs=false

# PostgreSQL（database 组）
postgresql_admin_username=postgres
postgresql_admin_password=redhat

# 离线安装
bundle_install=true
bundle_dir='{{ lookup("ansible.builtin.env", "PWD") }}/bundle'

redis_mode=standalone
```

### 2.3 各组件密码与数据库主机

| 组件 | 关键变量 | DEMO 值 |
| --- | --- | --- |
| **Gateway** | `gateway_admin_password` | `redhat` |
| | `gateway_pg_host` / `gateway_pg_password` | `aap26.example.com` / `redhat` |
| **Controller** | `controller_admin_password` | `redhat` |
| | `controller_pg_host` / `controller_pg_password` | `aap26.example.com` / `redhat` |
| | `controller_percent_memory_capacity` | `0.5` |
| **Hub** | `hub_admin_password` | `redhat` |
| | `hub_pg_host` / `hub_pg_password` | `aap26.example.com` / `redhat` |
| | `hub_seed_collections` | `false` |
| **EDA** | `eda_admin_password` | `redhat` |
| | `eda_pg_host` / `eda_pg_password` | `aap26.example.com` / `redhat` |

对应 inventory 片段：

```ini
# AAP Gateway
gateway_admin_password=redhat
gateway_pg_host=aap26.example.com
gateway_pg_password=redhat

# AAP Controller
controller_admin_password=redhat
controller_pg_host=aap26.example.com
controller_pg_password=redhat
controller_percent_memory_capacity=0.5

# AAP Automation Hub
hub_admin_password=redhat
hub_pg_host=aap26.example.com
hub_pg_password=redhat
hub_seed_collections=false

# AAP EDA Controller
eda_admin_password=redhat
eda_pg_host=aap26.example.com
eda_pg_password=redhat
```

### 2.4 Ansible MCP Server 变量（AIOps DEMO）

将 MCP 相关变量置于同一 `[all:vars]` 块（或与上文合并）：

```ini
# Ansible MCP Server
mcp_allow_write_operations=true
mcp_ignore_certificate_errors=true
# mcp_tls_cert=<path to tls certificate>
# mcp_tls_key=<path to tls key>
```

| 变量 | DEMO 设置 | 说明 |
| --- | --- | --- |
| `mcp_allow_write_operations` | `true` | 允许 MCP 写操作（Job 触发等） |
| `mcp_ignore_certificate_errors` | `true` | DEMO 环境忽略 TLS 校验；生产建议配置正式证书 |

> **密码安全**：DEMO 统一使用 `redhat` 仅为演示方便；生产环境请为各组件与数据库设置强密码。完整可选变量见 [Inventory File Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars)。

---

## 3. 执行安装

以 **`admin`** 用户在 bundle 目录执行：

```bash
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64
ansible-playbook -i inventory-growth ansible.containerized_installer.install
```

| 项目 | 说明 |
| --- | --- |
| **预计耗时** | 约 30–60 分钟（视硬件与磁盘 IOPS 而定） |
| **执行身份** | 必须为 `admin`（03-02 中已配置 `linger` 与 `sudo`） |
| **失败排查** | 查看 playbook 输出；常见问题包括 `/home` 空间不足、订阅/repo 未就绪、bundle 路径错误 |

安装成功后，各组件以 **Podman 容器**形式运行，并由 systemd 用户服务管理（`admin` 用户下）。

---

## 4. 安装后验证

### 4.1 检查容器状态

```bash
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

期望可见与 Gateway、Controller、Hub、EDA、Database、Redis、MCP 等相关的运行中容器（具体名称因版本略有差异）。

### 4.2 检查 systemd 用户服务

```bash
systemctl --user list-units --type=service | grep -i automation
```

### 4.3 访问 Web UI

| 项目 | 值 |
| --- | --- |
| **URL** | `https://aap26.example.com` |
| **默认用户** | `admin` |
| **密码** | `inventory-growth` 中 `gateway_admin_password`（DEMO：`redhat`） |

```bash
curl -k -I https://aap26.example.com
```

浏览器访问应呈现 AAP Gateway 登录页。许可激活与 UI 细项配置见 [03-04 AAP 配置](03-04-AAP-Configuration-CN.md)。

### 4.4 MCP 端点（安装后预留）

Ansible MCP Server 默认监听 **3000** 端口。安装完成后可初步探测：

```bash
curl -s -o /dev/null -w "%{http_code}" http://aap26.example.com:3000/mcp/job_management
```

完整 MCP → AI Agent 接线见 [03-05 AAP MCP For Agent](03-05-AAP-MCP-For-Agent-CN.md)。

---

## 部署后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | Playbook | `ansible.containerized_installer.install` 无 failed 任务 |
| 2 | 容器 | `podman ps` 显示核心 AAP 容器 Running |
| 3 | Gateway UI | `https://<FQDN>` 可打开登录页 |
| 4 | 登录 | `admin` / 配置密码可进入 |
| 5 | MCP 组 | inventory 中 `[ansiblemcp]` 已包含目标主机 |
| 6 | MCP 端口 | `:3000` 端点可连通（待 Token 配置后完整验签） |
| 7 | 许可 | 待 [03-04](03-04-AAP-Configuration-CN.md) 激活订阅或 Manifest |

---

## 参考文档

- [03-02 AAP 部署前准备](03-02-AAP-Preparation-CN.md)
- [Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation)
- [Inventory File Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars)
- [Container Topologies — Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies)

> **下一步**：[03-04 AAP 配置](03-04-AAP-Configuration-CN.md) — 激活许可、配置 UI 与 MCP Token
