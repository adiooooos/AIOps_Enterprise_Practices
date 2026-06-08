# AAP 部署先决条件

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 核心洞察

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **部署拓扑** | AIOps DEMO 采用 **AAP 2.6 Container Growth** 单节点拓扑——Gateway、Controller、Hub、EDA、Database 同机部署。 |
| ② | **两层规格** | **官方最低要求**满足安装器校验；**DEMO 推荐配置**（见 02-02）预留 EDA、MCP、并发 Job 余量。 |
| ③ | **先决 ≠ 准备** | 本章冻结「能不能装」；主机初始化、离线包、SSH 互信等操作见 [03-02 部署前准备](03-02-AAP-Preparation-CN.md)。 |

## 本章目标

在 AIOps Solution 环境下，确认 AAP Server **具备容器化安装的先决条件**——硬件、操作系统、权限、订阅、时间同步与网络基线达标后，再进入部署准备与安装。

| 项目 | 说明 |
| --- | --- |
| **适用版本** | RHEL 9.2+ · Ansible Automation Platform **2.6**（Containerized） |
| **部署拓扑** | [Container Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies) · 单 VM |
| **示例节点** | `aap26.example.com` · `10.210.65.24`（见 [02-02 资源规划](../02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md)） |

---

## Executive Summary: 先决条件一览

| 类别 | 要求 | 必须 |
| --- | --- | --- |
| **操作系统** | RHEL 9.2 及以上（DEMO 建议 9.6） | ✅ |
| **硬件** | 满足官方最低或 DEMO 推荐（见下表） | ✅ |
| **权限** | `sudo` / root；可降级至 AWX、PostgreSQL、EDA、Pulp 等服务用户 | ✅ |
| **时间同步** | 全节点 NTP / chronyd 已配置 | ✅ |
| **订阅** | RHEL 订阅 + AAP 订阅或 Manifest | ✅ |
| **网络** | FQDN 可解析；建议可访问 Internet（拉镜像 / 订阅） | 建议 ✅ |
| **容器运行时** | Podman（RHEL 9 默认）可用 | ✅ |

---

## 硬件与存储要求

### 官方最低要求（Container Growth · 单 VM）

> 来源：AAP 2.6 容器化安装文档 · Growth 拓扑测试配置

| 虚拟机数量 | 资源项 | 最低要求 | 同机容器角色 |
| --- | --- | --- | --- |
| **1** | **RAM** | 16 GB | automationgateway |
| | **CPU** | 4 vCPU | automationcontroller |
| | **本地磁盘** | 200 GB（**`/home` 至少 100 GB**） | automationhub |
| | **磁盘 IOPS** | 3000 | automationeda |
| | | | database |

### AIOps DEMO 推荐配置

> 在官方最低之上预留并发 Job、EDA Event Stream、后续 MCP 联调余量。详见 [02-02 资源规划](../02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md)。

| 资源项 | DEMO 推荐 | 说明 |
| --- | --- | --- |
| **vCPU** | 8 | 多容器同机 + 并发 Playbook |
| **内存** | 20 GB | Controller + EDA + Hub 峰值 |
| **磁盘** | 500 GB | 镜像、数据库、Project 同步、日志 |
| **磁盘 IOPS** | ≥ 3000 | 满足官方基线 |

---

## 操作系统与订阅

### 操作系统

| 要求 | 说明 |
| --- | --- |
| **版本** | Red Hat Enterprise Linux **9.2 及以上** |
| **DEMO 基线** | RHEL **9.6**（与全环境 02-02 对齐） |
| **架构** | x86_64 |
| **注册** | 使用 `subscription-manager` 注册并附加合适 Repo（03-02 执行） |

### AAP 许可

| 场景 | 要求 |
| --- | --- |
| **在线环境** | 有效 AAP 订阅；安装后通过订阅用户名/密码激活 |
| **离线 / 隔离环境** | 导入企业 **AAP Manifest** 文件激活（见 [03-04 配置](03-04-AAP-Configuration-CN.md)） |

---

## 权限与账户模型

容器化 AAP 安装器需要以下权限能力（安装阶段由 `admin` 或等效用户执行）：

| # | 先决条件 | 说明 |
| --- | --- | --- |
| 1 | **Root 访问** | 可通过 `sudo` 或权限提升获得 root |
| 2 | **权限降级** | 安装器可将进程权限从 root 降级到服务用户：**AWX**、**PostgreSQL**、**Event-Driven Ansible**、**Pulp** 等 |
| 3 | **部署用户** | 建议专用用户 `admin`（`NOPASSWD sudo` + `loginctl enable-linger`）——详见 03-02 |
| 4 | **SSH** | 安装节点对本机 FQDN / IP 的 SSH 互信（Ansible 本地连接） |

> **安全提示**：生产环境避免长期使用 `root` 直接运维；DEMO/PoC 可按 03-02 简化，上线前收紧 sudo 与 SSH 策略。

---

## 时间同步

| 要求 | 说明 |
| --- | --- |
| **服务** | 所有节点（含 AAP 及后续被管主机）配置 **chronyd** |
| **精度** | 节点间时间偏差建议 < 1s（EDA 告警时间戳、证书校验） |
| **验证** | `systemctl status chronyd` · `chronyc tracking` |

---

## 网络与主机名

| # | 先决条件 | 说明 |
| --- | --- | --- |
| 1 | **FQDN** | 主机名必须为 FQDN，如 `aap26.example.com`（非短名 `aap26`） |
| 2 | **正向 / 反向解析** | `/etc/hosts` 或内网 DNS 可解析；`ping $(hostname -f)` 成功 |
| 3 | **静态 IP** | 管理网固定 IP（示例 `10.210.65.24/24`） |
| 4 | **出站 Internet** | **建议**：订阅注册、容器镜像拉取、Collection 下载 |
| 5 | **入站端口** | 部署完成后 Web UI **443/tcp**（见 02-02 端口矩阵） |
| 6 | **集群互通** | 与 Prometheus、Chaos、n8n、Mattermost 等节点 SSH / HTTPS 可达 |

---

## 软件与运行时依赖

| 组件 | 要求 |
| --- | --- |
| **Podman** | RHEL 9 自带；容器化 AAP 运行时 |
| **ansible-core** | 安装器执行依赖（03-02 预装） |
| **工具链** | `wget` · `git` · `rsync` · `vim`（03-02 预装） |
| **安装包** | `ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64.tar.gz`（03-02 下载/解压） |

**官方文档：**

- [Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation)
- [Container Topologies](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies)

---

## 部署前检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | 硬件规格 | ≥ 官方最低；DEMO 建议 ≥ 8 vCPU / 20 GB / 500 GB |
| 2 | `/home` 空间 | ≥ 100 GB（容器镜像与数据目录） |
| 3 | 磁盘 IOPS | ≥ 3000 |
| 4 | RHEL 版本 | ≥ 9.2；已订阅注册 |
| 5 | AAP 许可 | 订阅或 Manifest 就绪 |
| 6 | FQDN | `hostname --fqdn` 返回完整域名 |
| 7 | chronyd | 服务 active & enabled |
| 8 | root / sudo | 部署用户可提权 |
| 9 | Internet / 离线包 | 在线可拉镜像，或离线 bundle 已就位 |
| 10 | 架构对齐 | IP / 主机名与 [02-02](../02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md) 一致 |

> **下一步**：[03-02 AAP 部署前准备](03-02-AAP-Preparation-CN.md) — 用户创建、IP/hosts、防火墙、SSH 互信、离线包解压
