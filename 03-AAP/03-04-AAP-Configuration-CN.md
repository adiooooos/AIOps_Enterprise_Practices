# AAP 配置（激活许可、部署 Starter Pack）

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **配置 ≠ 安装** | [03-03](03-03-AAP-Server-Deploy-CN.md) 完成平台安装；本章激活许可并导入 **Starter Pack V3** 内容，为 AIOps 可信执行打底。 |
| ② | **离线 IaC** | 使用 `AAP_Starter_Pack_v3.sh` 一键解压离线 ZIP、安装 Collection、按序执行 IaC Playbook，无需外网。 |
| ③ | **MCP 另章** | Ansible MCP → AI Agent 接线见 [03-05](03-05-AAP-MCP-For-Agent-CN.md)，可在 AI Agent 就绪后配置。 |

## 本章目标

在 AAP 2.6 容器化安装完成且 Gateway UI 可访问后，完成 **许可激活** 与 **Starter Pack V3 离线部署**——在 Controller 中创建 Organization、Projects、Inventories、Credentials、Job Templates 等对象，支撑后续 AIOps DEMO 自动化场景。

| 项目 | 说明 |
| --- | --- |
| **适用版本** | AAP **2.5 / 2.6**（Containerized，DEMO 为 2.6） |
| **示例节点** | `aap26.example.com` · `10.210.65.24` |
| **执行用户** | `admin`（与 03-02 / 03-03 容器化安装一致） |
| **Projects 路径** | `/home/admin/aap/controller/data/projects` |
| **目标组织** | `StarterPack` |

---

## Executive Summary: 配置步骤一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **激活许可** | 浏览器登录 Gateway UI，绑定订阅或 Manifest | ✅ |
| 2 | **准备离线包** | 4 个 ZIP + `AAP_Starter_Pack_v3.sh` 就位 | ✅ |
| 3 | **编辑脚本变量** | `ADMIN_USER`、`AAP_HOSTNAME`、API 凭据 | ✅ |
| 4 | **执行一键脚本** | `./AAP_Starter_Pack_v3.sh` | ✅ |
| 5 | **UI 验证** | 确认 Organization / Job Templates 已创建 | ✅ |

---

## 1. 激活 AAP 许可

### 1.1 登录 Gateway UI

| 项目 | DEMO 值 |
| --- | --- |
| **URL** | `https://aap26.example.com` |
| **用户名** | `admin` |
| **密码** | `inventory-growth` 中 `gateway_admin_password`（DEMO：`redhat`） |

在浏览器地址栏输入 `https://<AAP Server FQDN>`，使用上述凭据登录。

### 1.2 绑定订阅

| 环境 | 操作 |
| --- | --- |
| **在线** | 输入 Red Hat 订阅账号与凭据，激活产品 |
| **离线** | 上传公司 AAP **Manifest** 文件完成激活 |

激活成功后，UI 不再提示许可过期，Controller / Hub / EDA 功能可用。

---

## 2. 部署 Starter Pack V3

> **目的**：为 AIOps 中的可信执行做好准备——通过 IaC 在 AAP 中批量创建演示所需的自动化对象。

### 2.1 概述

Starter Pack V3 采用 **Infrastructure as Code（IaC）** 在离线 / 气隙环境中部署：

1. 将离线 ZIP 与 Collection tarball 置于 AAP Controller 主机
2. 以 `admin` 用户执行 `AAP_Starter_Pack_v3.sh`
3. 脚本自动解压内容、离线安装 Collection、按固定顺序执行 Playbook

**IaC 创建的 AAP 对象包括：**

| 类型 | 说明 |
| --- | --- |
| **Organization** | `StarterPack` |
| **Projects** | Starter Pack 场景项目 |
| **Inventories / Hosts / Groups** | 受管主机与分组 |
| **Credentials** | 连接凭据 |
| **Job Templates** | 含 Survey 的作业模板（可扩展至 32+ 个） |

也可在文件就位后**手动按相同顺序**执行 Playbook，便于排错。

### 2.2 架构与目录布局

**目标环境：**

| 项目 | DEMO 配置 |
| --- | --- |
| **AAP 版本** | 2.6 Container Growth（单节点） |
| **执行上下文** | AAP Controller 主机，Linux 用户 `admin` |
| **Projects 根目录** | `/home/admin/aap/controller/data/projects` |

**部署后目录结构：**

| 路径 | 用途 |
| --- | --- |
| `Starter_Pack_V3/` | 解压后的 Starter Pack 场景、README；部分场景目录含 collections |
| `Iac_For_StaterPack_V3/` | IaC Playbook、`configs/*.yml`、本地 `./collections`、`ansible.cfg` |

**IaC 关键文件：**

| 文件 / 目录 | 说明 |
| --- | --- |
| `ansible.cfg` | 由脚本生成（`collections_paths = ./collections` 等） |
| `configure_aap.yml` | Organization + Projects（调度入口） |
| `configure_projects.yml` | Projects |
| `configure_inventories.yml` | Inventories → Hosts → Groups |
| `configure_credentials.yml` | Credentials |
| `configure_job_templates.yml` | Job Templates（含 Survey） |
| `configs/` | `auth.yml`、`organizations.yml`、`projects.yml`、`inventories.yml`、`hosts.yml`、`groups.yml`、`credentials.yml`、`job_templates.yml` |

**脚本生成的 `ansible.cfg` 示例：**

```ini
[defaults]
inventory = ./inventory
remote_user = root
collections_paths = ./collections
deprecation_warnings = False
host_key_checking = False

[privilege_escalation]
become = false
become_method = sudo
become_user = root
become_ask_pass = false
```

### 2.3 前置条件

#### 离线 ZIP 文件

将以下 4 个文件放入与 `AAP_Starter_Pack_v3.sh` **同级**的 `Offline_installation_packages/` 目录（脚本通过 `${SCRIPT_DIR}/Offline_installation_packages` 解析路径）：

| 文件 | 说明 |
| --- | --- |
| `01_AnsibleCollectionForStarterPack_IaC.zip` | IaC 项目 + Collection tarball（离线 `ansible-galaxy` 安装） |
| `02_APAC_AAP_Starter_Pack-main.zip` | Starter Pack 场景树 |
| `03_collections.zip` | 额外 Collection，解压至 `collections/` 供场景使用 |
| `04_IaC_offline_packages.zip` | IaC 离线附加包（可含示例 configs） |

#### 主机目录准备

```bash
# root 创建离线包目录并上传 ZIP
mkdir -p /tmp/Offline_installation_packages/
chmod 755 /tmp/Offline_installation_packages/*.zip

# 上传脚本并赋予执行权限
chmod 755 /tmp/AAP_Starter_Pack_v3.sh

# 确认 Projects 目录存在（不存在则脚本 precheck 失败）
ls /home/admin/aap/controller/data/projects
```

#### AAP API 连接变量

Playbook 需要 Controller API 参数。一键脚本通过 `-e` 传递；手动执行时维护 `configs/auth.yml`：

```yaml
---
aap_hostname: aap26.example.com
aap_username: admin
aap_password: "redhat"
aap_validate_certs: false   # 生产环境建议 true + 有效证书

# 生产推荐：OAuth Token 替代密码
# aap_token: "your-oauth-token"
```

> 生产环境请使用 OAuth Token + TLS 校验；勿将真实密钥提交至公开仓库。

#### 离线安装的 Collection（示例版本）

脚本按序安装至 `./collections`（以 bundle 实际版本为准）：

- `ansible-controller`、`ansible-hub`、`ansible-eda`、`ansible-platform`
- `ansible-posix`、`kubernetes-core`
- `infra-aap_configuration`、`infra-aap_configuration_extended`、`infra-aap_utilities`、`infra-ah_configuration`

安装后验证：

```bash
cd /home/admin/aap/controller/data/projects/Iac_For_StaterPack_V3
ansible-galaxy collection list
```

### 2.4 IaC 设计原则

| 原则 | 说明 |
| --- | --- |
| **数据与逻辑分离** | AAP 对象定义在 `configs/*.yml`；Playbook 加载并调用 `infra.aap_configuration` Role |
| **可组合执行** | `configure_aap.yml` 处理 Org + Projects；亦可单独运行各对象 Playbook 增量调试 |

### 2.5 配置文件与 Role 映射

| 配置文件 | 顶层 Key | Role |
| --- | --- | --- |
| `organizations.yml` | `aap_organizations` | `gateway_organizations` |
| `projects.yml` | `controller_projects` | `controller_projects` |
| `inventories.yml` / `hosts.yml` / `groups.yml` | — | `controller_inventories` / `controller_hosts` / `controller_host_groups` |
| `credentials.yml` | `controller_credentials` | — （敏感项建议 Ansible Vault 加密） |
| `job_templates.yml` | `controller_templates` | `controller_job_templates` |

> Survey 默认值须为 AAP 期望的字符串（如多选 `"true"` / `"false"`）。

### 2.6 一键部署（推荐）

#### Step 1：编辑脚本变量

```bash
su - admin
cd /tmp/
vi AAP_Starter_Pack_v3.sh
```

修改脚本顶部变量（DEMO 示例）：

```bash
ADMIN_USER="admin"                    # 必须与执行脚本的用户一致
AAP_HOSTNAME="aap26.example.com"      # AAP Controller FQDN
AAP_USER="admin"                      # AAP API 用户
AAP_PASSWORD="redhat"                 # 生产请用强密码或 Vault
```

#### Step 2：执行脚本

```bash
./AAP_Starter_Pack_v3.sh
```

**脚本执行摘要：**

| 阶段 | 动作 |
| --- | --- |
| **Precheck** | 当前用户 = `ADMIN_USER`；4 个 ZIP 存在；`PROJECTS_DIR` 存在 |
| **Starter Pack** | 创建 `Starter_Pack_V3`，解压 `02_*.zip`、`03_*.zip`，复制 collections 至场景目录 |
| **IaC** | 创建 `Iac_For_StaterPack_V3`，解压 `01_*.zip`、`04_*.zip`；若 `configs/auth.yml` 已存在则备份后恢复 |
| **Collections** | `ansible-galaxy collection install … -p ./collections` |
| **Playbooks** | 严格按下列顺序执行 |

**Playbook 执行顺序：**

```bash
ansible-playbook configure_aap.yml           -e "aap_hostname=... aap_username=... aap_password=..."
ansible-playbook configure_credentials.yml   -e "aap_hostname=... aap_username=... aap_password=..."
ansible-playbook configure_projects.yml      -e "aap_hostname=... aap_username=... aap_password=..."
ansible-playbook configure_inventories.yml   -e "aap_hostname=... aap_username=... aap_password=..."
ansible-playbook configure_job_templates.yml -e "aap_hostname=... aap_username=... aap_password=..."
```

**成功标志：**

```
==== AAP Starter Pack v3 offline one-click deployment completed.
     Please log in to AAP Web UI to verify that objects were created as expected. ====
```

> 完整运行日志较长（解压 + Ansible 任务输出），如需审计请归档日志。

---

## 3. UI 验证

登录 `https://aap26.example.com`，确认以下对象已创建：

| 检查项 | 期望 |
| --- | --- |
| **Organization** | `StarterPack` 存在 |
| **Projects** | Starter Pack 相关项目已同步 |
| **Inventories** | 演示用清单 / 主机 / 分组 |
| **Credentials** | 凭据已配置 |
| **Job Templates** | 模板列表可见，可手动 Launch 测试 |

---

## 配置后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | 许可 | Gateway UI 无许可过期提示 |
| 2 | 脚本 | `AAP_Starter_Pack_v3.sh` 执行完成，无 failed 任务 |
| 3 | 目录 | `Starter_Pack_V3/` 与 `Iac_For_StaterPack_V3/` 存在于 Projects 路径 |
| 4 | Collection | `ansible-galaxy collection list` 显示 infra.aap_configuration 等 |
| 5 | Organization | UI 中可见 `StarterPack` |
| 6 | Job Templates | UI 中可见 Starter Pack 作业模板 |

---

## 参考文档

- [03-03 AAP Server 部署](03-03-AAP-Server-Deploy-CN.md)
- [APAC AAP Starter Pack](https://github.com/APAC-Ansible-Tech-Sales/APAC_AAP_Starter_Pack)
- [IaC for Starter Pack](https://github.com/APAC-Ansible-Tech-Sales/IaC_for_Starter_Pack/tree/main)
- [infra.aap_configuration Collection](https://galaxy.ansible.com/ui/repo/published/infra/aap_configuration/)

> **下一步**：[03-05 AAP MCP For Agent](03-05-AAP-MCP-For-Agent-CN.md) — 创建 API Token，配置 MCP → AI Agent 连接
