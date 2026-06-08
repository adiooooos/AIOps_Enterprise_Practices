# AAP 部署前准备

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **准备 ≠ 安装** | 本章完成主机初始化与离线包就位；`inventory-growth` 编辑与 `setup.sh` 执行见 [03-03 AAP Server 部署](03-03-AAP-Server-Deploy-CN.md)。 |
| ② | **操作主体** | 容器化安装以 **`admin` 用户** 为部署主体；部分步骤（如 `useradd`、防火墙）需 **root** 执行后再切换至 `admin`。 |


## 本章目标

在确认 [03-01 先决条件](03-01-AAP-Prerequisites-CN.md) 达标后，完成 AAP Server 的 **标准化主机准备**——离线软件包、部署用户、网络与主机名、SSH 互信、订阅注册、时间同步、性能调优及安装包解压，为容器化安装做好准备。

| 项目 | 说明 |
| --- | --- |
| **适用版本** | RHEL 9.2+ · Ansible Automation Platform **2.6**（Containerized） |
| **部署拓扑** | [Container Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies) · 单 VM |
| **示例节点** | `aap26.example.com` · `10.210.65.24` |
| **部署用户** | `admin`（密码示例：`redhat`） |

---

## Executive Summary: 准备项一览

| # | 类别 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **离线软件包** | 下载 AAP 2.6 containerized bundle 或从网盘获取 | ✅ |
| 2 | **部署用户** | 创建 `admin`，配置 `sudo` 免密与 `linger` | ✅ |
| 3 | **网络** | 静态 IP、Gateway、DNS（建议 `nmtui`） | ✅ |
| 4 | **主机名** | FQDN + `/etc/hosts`（含 DEMO 全栈节点） | ✅ |
| 5 | **防火墙** | DEMO/PoC 可关闭；生产按端口策略放行 | 建议 |
| 6 | **SSH 互信** | root 与 admin 对本机及 `/etc/hosts` 各节点 | ✅ |
| 7 | **订阅注册** | `subscription-manager` 注册 RHEL 9 | ✅ |
| 8 | **时间同步** | 安装并启用 `chronyd` | ✅ |
| 9 | **性能调优** | `ulimit` `nofile` 提升至 65535 | 建议 |
| 10 | **依赖包** | `ansible-core`、`wget`、`git`、`rsync`、`vim` | ✅ |
| 11 | **解压安装包** | bundle 置于 `/home/admin` 并解压 | ✅ |

---

## 1. 离线软件包准备

### 方式 A：百度网盘（AIOps DEMO 预置包）

| 项目 | 内容 |
| --- | --- |
| **分享名称** | `01-AIOPS_sws` |
| **链接** | https://pan.baidu.com/s/1cmwuHd5ztjbw4bCg7WwlEQ?pwd=AE86 |
| **提取码** | `AE86` |

> 网盘包通常已包含与 DEMO 环境匹配的 AAP 2.6 离线 bundle，下载后拷贝至 AAP Server 即可。

### 方式 B：Red Hat 客户门户（在线下载）

| 项目 | 内容 |
| --- | --- |
| **产品** | Ansible Automation Platform 2.6 · Containerized Setup Bundle |
| **下载页** | https://access.redhat.com/downloads/content/480/ver=2.6/rhel---9/2.6/x86_64/product-software |
| **文件名示例** | `ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64.tar.gz` |

> 请选择与目标 RHEL 版本（x86_64）匹配的离线安装包。

---

## 2. 容器化 AAP Server 标准化配置

> **操作说明**：以下步骤在全新 RHEL 9 虚拟机上执行。标注 `[root]` 的步骤使用 root；标注 `[admin]` 的步骤切换至 `admin` 用户。完成全部准备后，后续安装均在 `admin` 下操作。

### 2.1 创建部署用户 `admin`

以 **root** 执行：

```bash
useradd admin
echo redhat | passwd admin --stdin
echo "admin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
loginctl enable-linger admin
```

| 配置项 | 说明 |
| --- | --- |
| **密码** | 示例为 `redhat`；生产环境请使用强密码 |
| **sudo** | `NOPASSWD: ALL` 便于无人值守安装；生产可收紧 |
| **linger** | 允许 `admin` 在用户未登录时运行 systemd 用户服务（容器化安装需要） |

### 2.2 静态 IP 配置

建议使用 **`nmtui`** 配置 AAP Server 静态 IP，并同时设置 **Gateway** 与 **DNS Server**。

**DEMO 示例**（`ens33` · `10.210.65.24/24`）：

```ini
# /etc/NetworkManager/system-connections/ens33.nmconnection（节选）
[ipv4]
address1=10.210.65.24/24,10.210.65.254
dns=10.192.206.245;10.200.0.245;
may-fail=false
method=manual
```

验证：

```bash
nmcli connection show ens33 | grep ipv4
ip -4 addr show ens33
```

### 2.3 主机名与 `/etc/hosts`

AAP 2.6 要求使用 **FQDN**。DEMO 建议主机名：

| 项目 | 值 |
| --- | --- |
| **FQDN** | `aap26.example.com` |
| **短名** | `aap26` |

```bash
sudo hostnamectl set-hostname aap26.example.com
sudo vi /etc/hosts   # 填入 AAP 及 DEMO 全栈节点
```

**DEMO `/etc/hosts` 示例**（与 02-02 资源规划对齐）：

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.210.65.24  aap26.example.com  aap26
10.210.65.17  test-rhel-7.9-1
10.210.65.103 n8n.example.com    n8n
10.210.65.45  win2019
10.210.65.174 prometheus.example.com prometheus
10.210.65.148 Chaos.example.com  Chaos
10.210.65.14  mattermost.example.com mattermost
```

验证 FQDN 解析：

```bash
ping -c1 localhost
ping -c1 $(hostname -f)
hostname --fqdn
# 期望输出：aap26.example.com
```

### 2.4 防火墙

| 环境 | 建议 |
| --- | --- |
| **DEMO / PoC** | 可关闭防火墙以简化排障 |
| **生产** | 按 AAP 2.6 文档放行 Gateway、Controller、Hub、EDA 所需端口 |

DEMO 关闭示例（root）：

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 2.5 SSH 互信

安装器与后续运维需要 AAP Server 对本机及受管节点 **免密 SSH**。需分别为 **root** 与 **admin** 配置。

**本机互信（root）**：

```bash
ssh-keygen          # 一路回车，使用默认路径
ssh-copy-id aap26.example.com
ssh-copy-id aap26
ssh-copy-id 10.210.65.24
```

**本机互信（admin）**：

```bash
su - admin
ssh-keygen
ssh-copy-id aap26.example.com
ssh-copy-id aap26
ssh-copy-id 10.210.65.24
```

**扩展至 `/etc/hosts` 中其余节点**：

对 `test-rhel-7.9-1`、`n8n.example.com`、`prometheus.example.com` 等每一台目标主机，分别以 root 和 admin 执行 `ssh-copy-id <hostname-or-ip>`，确保安装与 playbook 可直达。

验证：

```bash
ssh aap26.example.com hostname
ssh prometheus.example.com hostname   # 示例
```

### 2.6 RHEL 订阅注册

以具备订阅权限的账号注册 RHEL 9 主机（root 或 admin + sudo）：

```bash
sudo subscription-manager register --username <RHSM_USER> --password <RHSM_PASS>
sudo subscription-manager attach --auto
sudo subscription-manager repos --enable ansible-automation-platform-2.6-for-rhel-9-x86_64-rpms
sudo dnf repolist
```

> 离线环境可使用 Manifest 或预配置的本地 repo；AAP 许可激活在 [03-04 配置](03-04-AAP-Configuration-CN.md) 章节完成。

### 2.7 系统时钟（chronyd）

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
systemctl status chronyd
```

### 2.8 性能调优（`ulimit` nofile）

大规模环境下建议提升打开文件数上限：

```bash
sudo tee /etc/security/limits.d/40-nofile.conf <<'EOF'
* soft nofile 65535
* hard nofile 65535
EOF

cat /etc/security/limits.d/40-nofile.conf
```

> 修改后建议重新登录会话使 limits 生效。

### 2.9 安装必要工具包

```bash
sudo dnf install -y ansible-core wget git-core rsync vim
ansible --version
```

| 包 | 用途 |
| --- | --- |
| **ansible-core** | 容器化安装器执行依赖 |
| **wget / git / rsync / vim** | 下载、同步、编辑 inventory 与配置 |

### 2.10 拷贝并解压离线安装包

将 bundle 拷贝至 **`/home/admin`**（示例使用 `admin` 家目录）：

```bash
# 以 admin 用户执行（或通过 scp/rsync 从跳板机传入）
cd /home/admin
tar xzf ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64.tar.gz
cd ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64
ls -la
```

解压后目录应包含 `inventory-growth`、`setup.sh`、`README.md` 等安装器文件。

---

## 部署前检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | 部署用户 | `admin` 存在，可 `sudo` 免密，`loginctl show-user admin` 显示 `Linger=yes` |
| 2 | 静态 IP | 与 02-02 规划一致；Gateway / DNS 可达 |
| 3 | FQDN | `hostname --fqdn` → `aap26.example.com` |
| 4 | `/etc/hosts` | AAP 及 DEMO 关联节点均已录入 |
| 5 | SSH 互信 | root 与 admin 对本机及目标节点 `ssh <host> hostname` 无密码提示 |
| 6 | RHEL 订阅 | `subscription-manager status` 为已注册 |
| 7 | chronyd | `systemctl is-active chronyd` → `active` |
| 8 | nofile | `/etc/security/limits.d/40-nofile.conf` 含 65535 |
| 9 | ansible-core | `ansible --version` 正常 |
| 10 | 离线包 | bundle 已解压于 `/home/admin/...`，可见 `inventory-growth` |

---

## 参考文档

- [03-01 AAP 部署先决条件](03-01-AAP-Prerequisites-CN.md)
- [Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation)
- [02-02 资源规划](../02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md)

> **下一步**：[03-03 AAP Server 部署](03-03-AAP-Server-Deploy-CN.md) — 编辑 `inventory-growth` 并执行 `setup.sh`
