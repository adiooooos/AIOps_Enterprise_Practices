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

在 bundle 目录中编辑 `inventory-growth`（基于 bundle `2.6-7` 模板）。AIOps DEMO 在官方模板基础上做如下调整：

| # | 调整项 | DEMO 设置 |
| --- | --- | --- |
| 1 | **所有主机组** | `automationgateway` / `automationcontroller` / `automationhub` / `automationeda` / `database` 均指向 `aap26.example.com` |
| 2 | **离线安装** | `bundle_install=true`，`bundle_dir` 指向当前目录下 `bundle/` |
| 3 | **组件密码** | Gateway / Controller / Hub / EDA 及 PostgreSQL 密码均为 `redhat`（演示用） |
| 4 | **Controller 内存** | `controller_percent_memory_capacity=0.5` |
| 5 | **Hub 种子集合** | `hub_seed_collections=false` |
| 6 | **Ansible MCP** | 在文件末尾启用 `[ansiblemcp]` 组，并追加第二个 `[all:vars]` 块配置 MCP 权限 |
| 7 | **Lightspeed** | 保持注释（DEMO 不部署） |

> **密码安全**：DEMO 统一使用 `redhat` 仅为演示方便；生产环境请为各组件与数据库设置强密码。完整可选变量见 [Inventory File Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars)。

配置完成后可用以下命令核对；**完整文件见文末 [附录](#附录inventory-growth-完整文件)**。

```bash
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-7-x86_64
more inventory-growth
```

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

---

## 附录：`inventory-growth` 完整文件

以下为 AIOps DEMO 配置完成后的 **`inventory-growth` 全文**（`ansible-automation-platform-containerized-setup-bundle-2.6-7-x86_64` 目录下）：

```ini
# This is the AAP installer inventory file intended for the Container growth deployment topology.
# This inventory file expects to be run from the host where AAP will be installed.
# Please consult the Ansible Automation Platform product documentation about this topology's tested hardware configuration.
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies
#
# Please consult the docs if you're unsure what to add
# For all optional variables please consult the included README.md
# or the Ansible Automation Platform documentation:
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation

# This section is for your AAP Gateway host(s)
# -----------------------------------------------------
[automationgateway]
aap26.example.com
# This section is for your AAP Controller host(s)
# -----------------------------------------------------
[automationcontroller]
aap26.example.com
# This section is for your AAP Automation Hub host(s)
# -----------------------------------------------------
[automationhub]
aap26.example.com
# This section is for your AAP EDA Controller host(s)
# -----------------------------------------------------
[automationeda]
aap26.example.com
# This section is for your AAP Lightspeed host(s)
# -----------------------------------------------------
# [ansiblelightspeed]
# aap.example.org

# This section is for your Ansible MCP Server host(s)
# -----------------------------------------------------
# [ansiblemcp]
# aap.example.org

# This section is for the AAP database
# -----------------------------------------------------
[database]
aap26.example.com
[all:vars]
# Ansible
ansible_connection=local
validate_certs=false

# Common variables
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#general-variables
# -----------------------------------------------------
postgresql_admin_username=postgres
postgresql_admin_password=redhat

bundle_install=true
# The bundle directory must include /bundle in the path
bundle_dir='{{ lookup("ansible.builtin.env", "PWD") }}/bundle'


redis_mode=standalone

# AAP Gateway
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#platform-gateway-variables
# -----------------------------------------------------
gateway_admin_password=redhat
gateway_pg_host=aap26.example.com
gateway_pg_password=redhat

# AAP Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#controller-variables
# -----------------------------------------------------
controller_admin_password=redhat
controller_pg_host=aap26.example.com
controller_pg_password=redhat
controller_percent_memory_capacity=0.5

# AAP Automation Hub
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#hub-variables
# -----------------------------------------------------
hub_admin_password=redhat
hub_pg_host=aap26.example.com
hub_pg_password=redhat
hub_seed_collections=false

# AAP EDA Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#event-driven-ansible-variables
# -----------------------------------------------------
eda_admin_password=redhat
eda_pg_host=aap26.example.com
eda_pg_password=redhat

# AAP Lightspeed
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#lightspeed-variables
# -----------------------------------------------------
# lightspeed_admin_password=<set your own>
# lightspeed_pg_host=aap.example.org
# lightspeed_pg_password=<set your own>

# In case chabot is enabled, default provider is "rhoai"
# lightspeed_chatbot_model_url=<set your own>
# lightspeed_chatbot_model_api_key=<set your own>
# lightspeed_chatbot_model_id=<set your own>

# In case "azure" provider
# lightspeed_chatbot_default_provider = "azure"

# In case "openai" provider
# lightspeed_chatbot_default_provider = "openai"

# lightspeed_mcp_controller_enabled=true
# lightspeed_mcp_lightspeed_enabled=true
# lightspeed_wca_model_api_key=<set your own>
# lightspeed_wca_model_id=<set your own>
[ansiblemcp]
aap26.example.com

# This section is for Ansible MCP server permissions
# --------------------------------------------------
[all:vars]
mcp_allow_write_operations=true
mcp_ignore_certificate_errors=true
#mcp_tls_cert= <path to tls certificate>
#mcp_tls_key= <path to tls key>
```

> **下一步**：[03-04 AAP 配置](03-04-AAP-Configuration-CN.md) — 激活许可、配置 UI 与 MCP Token
