# AIOps DEMO Center 部署与配置分步指南

> **系列**：AIOps DEMO Center 部署与配置分步指南  
> **English**: [README.md](README.md)  
> **状态**：202606 Updated  

本指南用于**完整复现 GCG AIOps DEMO Center**：可观测（Prometheus）→ 事件驱动（EDA）→ 工作流编排（n8n）→ 人机交互（Mattermost / ChatOps）→ AI 智能体（MCP for AAP / SSH）。

---

## 一眼看懂

| 项目 | 说明 |
| --- | --- |
| **最小环境** | **5 台核心服务器** — AAP · Triage/Mattermost · Chaos · n8n · Prometheus |
| **技术栈** | RHEL 9.6 · AAP 2.6 · Event-Driven Ansible · Podman |
| **示例网段** | `10.210.65.0/24`（可按实际环境全局替换） |
| **文档语言** | 各章均提供 **`-CN.md`**（中文）与 **`-EN.md`**（英文） |
| **内容来源** | 基于已落地的 `01_AIOPS_DEMO_CENTER部署及配置` 实战环境 |

---

## 架构概览（核心 5 节点）

```
                    ┌─────────────────────────────────────┐
                    │  管理员 / ChatOps（Mattermost :8065）│
                    └──────────────────┬──────────────────┘
                                       │
     ┌──────────────┐    事件/Webhook  │    AI Agent 工作流
     │ Prometheus   │◀───指标─────────┼───▶ n8n (:8080)
     │ :9090/:9093  │───告警─────────▶│         │
     └──────┬───────┘                  │         ▼
            │                           │    MCP Client
            ▼                           │         │
     ┌──────────────┐                   │    ┌────┴────┐
     │ Chaos        │◀──SSH/AAP─────────┼───▶│ AAP     │
     │（故障靶机）   │                   │    │ :443    │
     └──────────────┘                   │    │ EDA+MCP │
                                        │    └─────────┘
                    Triage / Mattermost (:8065) + MCP for SSH (:3000)
                              10.210.65.14
```

| 主机（示例） | IP | 角色 |
| --- | --- | --- |
| `AAP2.6.exmaple.com` | 10.210.65.24 | AAP · EDA · MCP for AAP |
| `Mattermost.example.com` | 10.210.65.14 | 分诊台 · Mattermost · MCP for SSH |
| `Chaos.example.com` | 10.210.65.148 | 故障仿真靶机 |
| `n8n.example.com` | 10.210.65.103 | n8n 工作流引擎 |
| `prometheus.example.com` | 10.210.65.174 | Prometheus · Alertmanager · node_exporter |

详见：[02-01 组件与环境](02-DEMO_architecture_Planning/02-01-Components-And-Environments-CN.md) · [02-02 资源与 IP 规划](02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md)

---

## 四大 DEMO 场景

| # | 触发器 | 场景 | 关键组件 |
| --- | --- | --- | --- |
| **UC1** | Chat | 智能健康巡检 | Mattermost → n8n → MCP for AAP → Job Template |
| **UC2** | Chat + Schedule | 智能补丁 / 漏洞管理 | AAP · HITL · Policy Enforcement |
| **UC3** | Event (EDA) | Process Hung 根因分析与自愈 | Prometheus → EDA → 分诊台 → n8n → AI Agent |
| **UC4** | Chat + Policy | Web 配置合规拦截 | Policy Enforcement 拒绝不安全作业 |

场景详解：[01-04 Use Cases](01-solution-introduction/01-04-Use-Cases-CN.md)

---

## 推荐阅读与部署顺序

### 阶段 A — 先读懂（阅读）

| 步骤 | 文档 | 目的 |
| --- | --- | --- |
| 0 | [00-Blueprint-CN.md](00-overview/00-Blueprint-CN.md) | 系列范围与编写规范 |
| 1 | [01-01 … 01-05](01-solution-introduction/) | Why / What / How to Land / 场景 / 商业价值 |
| 2 | [02-01](02-DEMO_architecture_Planning/02-01-Components-And-Environments-CN.md) · [02-02](02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md) | 组件、规格、IP 与端口 |

### 阶段 B — 再动手（部署）

| 顺序 | 章节 | 文档 | 部署节点 |
| --- | --- | --- | --- |
| 1 | **03 AAP** | [03-01](03-aap/03-01-AAP-Prerequisites-CN.md) → [03-06](03-aap/03-06-Automation-Decisions-CN.md) | AAP |
| 2 | **04 Prometheus** | [04-01](04-prometheus/04-01-Prometheus-Config-CN.md) | Prometheus |
| 3 | **05 Chaos** | [05-01](05-chaos/05-01-Chaos-Process-Hung-CN.md) · [05-02](05-chaos/05-02-Chaos-Network-Error-CN.md) · [05-03](05-chaos/05-03-Chaos-Other-Outage-CN.md) | Chaos |
| 4 | **07 分诊台** | [07-01](07-triage-and-ChatOps/07-01-Triage-Diagnostics-CN.md) | Triage |
| 5 | **07 Mattermost** | [07-02](07-triage-and-ChatOps/07-02-Mattermost-Deploy-CN.md) · [07-03](07-triage-and-ChatOps/07-03-Mattermost-Config-CN.md) | Triage |
| 6 | **08 n8n** | [08-01](08-n8n/08-01-n8n-Deploy-CN.md) · [08-02](08-n8n/08-02-n8n-Workflow-Import-CN.md) | n8n |
| 7 | **06 EDA** | [06-01](06-EDA-rulebooks_and_actions/06-01-EDA-Artifacts-CN.md) | AAP（Rulebook / JT / Playbook） |
| 8 | **09 MCP SSH** | [09-01](09-mcp-ssh/09-01-MCP-SSH-Setup-CN.md) | Triage |
| 9 | **10 DEMO** | [10-01](10-aiops-DEMO-and-screenshots/10-01-Agent-Workflow-E2E-CN.md) · [Workflow 提示词](10-aiops-DEMO-and-screenshots/Workflow提示词.md) | 端到端验证 |

> **说明**：**06 EDA** 依赖 AAP、Prometheus 告警链路、n8n Webhook 及分诊台目录，请在上述组件就绪后再导入制品。

---

## 文档索引

### 00 — 总览

| 文件 | 说明 |
| --- | --- |
| [00-Blueprint-CN.md](00-overview/00-Blueprint-CN.md) | 系列蓝图与编写规范 |
| [00-01-Guide-How-To-Use-CN.md](00-overview/00-01-Guide-How-To-Use-CN.md) | 项目背景与目标 |

### 01 — 方案介绍

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [01-01-CN](01-solution-introduction/01-01-Why-AIOps-传统运维瓶颈%20%26%20AI%20核心价值-CN.md) | [01-01-EN](01-solution-introduction/01-01-Why-AIOps-Traditional%20Pain%20Points%20%26%20AI%20Value-EN.md) | Why AIOps |
| [01-02-CN](01-solution-introduction/01-02-What-Is-AIOps-核心能力定义与参考架构-CN.md) | [01-02-EN](01-solution-introduction/01-02-What-Is-AIOps-%20Core%20Capability%20Definition%20and%20Reference%20Architecture-EN.md) | What is AIOps · 参考架构 |
| [01-03-CN](01-solution-introduction/01-03-How-To-Land-CN.md) | [01-03-EN](01-solution-introduction/01-03-How-To-Land-EN.md) | How to Land |
| [01-04-CN](01-solution-introduction/01-04-Use-Cases-CN.md) | [01-04-EN](01-solution-introduction/01-04-Use-Cases-EN.md) | 四大场景 |
| [01-05-CN](01-solution-introduction/01-05-Biz-Value-CN.md) | [01-05-EN](01-solution-introduction/01-05-Biz-Value-EN.md) | 商业价值与 ROI |

### 02 — 架构与规划

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [02-01-CN](02-DEMO_architecture_Planning/02-01-Components-And-Environments-CN.md) | [02-01-EN](02-DEMO_architecture_Planning/02-01-Components-And-Environments-EN.md) | 组件与环境 |
| [02-02-CN](02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md) | [02-02-EN](02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md) | 资源 · IP · 端口 |

### 03 — Ansible Automation Platform

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [03-01-CN](03-aap/03-01-AAP-Prerequisites-CN.md) | [03-01-EN](03-aap/03-01-AAP-Prerequisites-EN.md) | 部署先决条件 |
| [03-02-CN](03-aap/03-02-AAP-Preparation-CN.md) | [03-02-EN](03-aap/03-02-AAP-Preparation-EN.md) | 部署前准备 |
| [03-03-CN](03-aap/03-03-AAP-Server-Deploy-CN.md) | [03-03-EN](03-aap/03-03-AAP-Server-Deploy-EN.md) | 平台安装 |
| [03-04-CN](03-aap/03-04-AAP-Configuration-CN.md) | [03-04-EN](03-aap/03-04-AAP-Configuration-EN.md) | 许可 · Starter Pack |
| [03-05-CN](03-aap/03-05-AAP-MCP-For-Agent-CN.md) | [03-05-EN](03-aap/03-05-AAP-MCP-For-Agent-EN.md) | MCP for AAP → AI Agent |
| [03-06-CN](03-aap/03-06-Automation-Decisions-CN.md) | [03-06-EN](03-aap/03-06-Automation-Decisions-EN.md) | EDA / Automation Decisions |

### 04 — Prometheus

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [04-01-CN](04-prometheus/04-01-Prometheus-Config-CN.md) | [04-01-EN](04-prometheus/04-01-Prometheus-Config-EN.md) | Prometheus · Alertmanager · node_exporter |

### 05 — Chaos（故障仿真）

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [05-01-CN](05-chaos/05-01-Chaos-Process-Hung-CN.md) | [05-01-EN](05-chaos/05-01-Chaos-Process-Hung-EN.md) | UC3 · 进程挂起（CPU 压测） |
| [05-02-CN](05-chaos/05-02-Chaos-Network-Error-CN.md) | [05-02-EN](05-chaos/05-02-Chaos-Network-Error-EN.md) | 网络接口 DOWN |
| [05-03-CN](05-chaos/05-03-Chaos-Other-Outage-CN.md) | [05-03-EN](05-chaos/05-03-Chaos-Other-Outage-EN.md) | xsos 混沌（CPU / 磁盘） |

### 06 — EDA Rulebook 与 Action

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [06-01-CN](06-EDA-rulebooks_and_actions/06-01-EDA-Artifacts-CN.md) | [06-01-EN](06-EDA-rulebooks_and_actions/06-01-EDA-Artifacts-EN.md) | Rulebook · JT · Playbook（UC1–3） |

### 07 — 分诊台与 ChatOps

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [07-01-CN](07-triage-and-ChatOps/07-01-Triage-Diagnostics-CN.md) | [07-01-EN](07-triage-and-ChatOps/07-01-Triage-Diagnostics-EN.md) | 分诊目录 · 诊断样例 |
| [07-02-CN](07-triage-and-ChatOps/07-02-Mattermost-Deploy-CN.md) | [07-02-EN](07-triage-and-ChatOps/07-02-Mattermost-Deploy-EN.md) | Mattermost 安装 |
| [07-03-CN](07-triage-and-ChatOps/07-03-Mattermost-Config-CN.md) | [07-03-EN](07-triage-and-ChatOps/07-03-Mattermost-Config-EN.md) | Webhook · Bot · 频道（截图位） |

### 08 — n8n

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [08-01-CN](08-n8n/08-01-n8n-Deploy-CN.md) | [08-01-EN](08-n8n/08-01-n8n-Deploy-EN.md) | n8n 部署（在线 / 离线） |
| [08-02-CN](08-n8n/08-02-n8n-Workflow-Import-CN.md) | [08-02-EN](08-n8n/08-02-n8n-Workflow-Import-EN.md) | 导入预置 Workflow |

### 09 — MCP for SSH

| 中文 | 英文 | 主题 |
| --- | --- | --- |
| [09-01-CN](09-mcp-ssh/09-01-MCP-SSH-Setup-CN.md) | [09-01-EN](09-mcp-ssh/09-01-MCP-SSH-Setup-EN.md) | ssh-mcp 容器 · 公钥分发 · N8N 对接 |

### 10 — DEMO 与截图

| 文件 | 说明 |
| --- | --- |
| [10-01-Agent-Workflow-E2E-CN.md](10-aiops-DEMO-and-screenshots/10-01-Agent-Workflow-E2E-CN.md) | Agent 工作流端到端（待完善） |
| [Workflow提示词.md](10-aiops-DEMO-and-screenshots/Workflow提示词.md) | n8n AI Agent System Prompt 模板 |

---

## 关键访问地址（DEMO 示例）

| 服务 | 地址（示例） | 对应章节 |
| --- | --- | --- |
| AAP Web UI | `https://10.210.65.24` | 03 |
| Prometheus | `http://10.210.65.174:9090` | 04 |
| Alertmanager | `http://10.210.65.174:9093` | 04 |
| Mattermost | `http://10.210.65.14:8065` | 07 |
| n8n | `http://10.210.65.103:8080` | 08 |
| MCP for AAP | `http://10.210.65.24:3000/mcp/<toolset>` | 03-05 |
| MCP for SSH | `http://10.210.65.14:3000/mcp` | 09 |

---

## 使用说明

1. **选择语言** — 阅读 `-CN.md` 或 `-EN.md`；命令与配置内容一致。
2. **冻结 IP 规划** — 复制 [02-02](02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md) 中的地址表，按需全局替换 `10.210.65.x`。
3. **按阶段 B 顺序部署** — 勿跳过前置步骤（SSH 互信、firewalld、离线包等）。
4. **逐章验证** — 各章含预期输出；最终以 UC1 聊天巡检作为冒烟测试。
5. **方案叙事与 ROI** — 演讲材料见 `10_AIOPS_Solution_Slides/`；可对照 [01-04](01-solution-introduction/01-04-Use-Cases-CN.md) 与 [01-05](01-solution-introduction/01-05-Biz-Value-CN.md)。

---

## 相关资源

| 资源 | 路径 |
| --- | --- |
| 实战配置工作区 | `01_AIOPS_DEMO_CENTER部署及配置/` |
| 方案 Slides | `10_AIOPS_Solution_Slides/` |
| 架构总览 | `01_AAP2.6.exmaple.com/00-AIOPS架构.md` |
| GitHub（目标仓库） | [AIOps_Enterprise_Practices](https://github.com/adiooooos/AIOps_Enterprise_Practices) |

---

*Red Hat AIOps Solution — 通过 EDA + Agentic AI 实现 AI 时代的智能运维。*
