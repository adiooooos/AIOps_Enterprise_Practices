# AIOps 组件与环境规划

> **状态**：20260603 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 核心洞察

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **分层部署** | DEMO 环境按「可观测 → 事件驱动 → 自动化执行 → 人机交互 → AI 推理」分层拆分为独立节点，便于独立升级与故障隔离。 |
| ② | **最小可用集** | **5 台核心服务器**（AAP / Triage / Chaos / n8n / Prometheus）即可跑通 Part I 四大 Use Case；AI 栈（Hermes + vLLM + RAG）为增强项。 |
| ③ | **规格基准** | 以下资源为单节点 **最低建议配置**；IP、端口、网络与汇总清单见 [02-02 资源规划](02-02-Resources-Planning-CN.md)。 |

## Executive Summary: 组件与资源一览

**执行摘要**：AIOps DEMO 环境组件及单节点资源配置（RHEL 9.6）

| 组件 | 节点数 | CPU | 内存 | 磁盘 | 备注 |
| --- | --- | --- | --- | --- | --- |
| **AAP**（含 EDA、MCP） | 1 | 8 vCPU | 20 GB | 500 GB | 自动化中枢：Playbook / Job Template / EDA Rulebook / MCP Server |
| **Triage Server**（含 Mattermost） | 1 | 4 vCPU | 8 GB | 500 GB | 分诊台：告警协作、ChatOps 统一入口 |
| **Chaos Server** | 1 | 4 vCPU | 8 GB | 200 GB | 故障仿真靶机，触发 Prometheus 告警链路 |
| **n8n** | 1 | 4 vCPU | 8 GB | 500 GB | 工作流编排，衔接告警与下游动作 |
| **Prometheus Server** | 1 | 4 vCPU | 8 GB | 200 GB | Prometheus + Alertmanager + node_exporter |
| **Hermes AI Agent** | 1 | 2 vCPU | 4 GB | 500 GB | **可选** · 智能体运行时 |
| **vLLM + Chat LLM** | 1 | 16 vCPU | 64 GB | 240 GB | **GPU 160 GB 显存** · Qwen3-32B 对话模型 |
| **Embedding + Reranking** | 1 | 16 vCPU | 64 GB | 240 GB | **GPU 160 GB 显存** · RAG 向量检索与重排序（可与 vLLM 同机或分机） |

> **合计**：核心平台 **5 台**；启用完整 AI 能力时另增 **1～3 台**（视 Hermes / vLLM / RAG 是否合并部署而定）。

---

## 核心平台组件

### ① AAP · 自动化与事件驱动中枢

| 属性 | 说明 |
| --- | --- |
| **职责** | 运行 AAP Web UI；托管 Inventory、Project、Job Template；EDA 接收 Prometheus 告警并触发 Rulebook / Playbook；MCP Server 供 AI Agent 调用 |
| **关键模块** | Ansible Automation Platform · Event-Driven Ansible (EDA) · MCP for AAP |
| **资源配置** | 1 台 · 8 vCPU + 20 GB RAM + 500 GB Disk |
| **操作系统** | RHEL 9.6 |
| **逻辑主机名（示例）** | `AAP2.6.exmaple.com` |

### ② Triage Server · 分诊与协作入口

| 属性 | 说明 |
| --- | --- |
| **职责** | 告警分诊台；部署 Mattermost，作为 ChatOps 人机交互统一入口 |
| **资源配置** | 1 台 · 4 vCPU + 8 GB RAM + 500 GB Disk |
| **操作系统** | RHEL 9.6 |
| **逻辑主机名（示例）** | `Mattermost.example.com` |

### ③ Chaos Server · 故障仿真靶机

| 属性 | 说明 |
| --- | --- |
| **职责** | 模拟 CPU / 内存 / 磁盘 I/O 等 OS 级故障；故障经 Prometheus 规则产生告警，驱动 EDA 自动排障链路 |
| **资源配置** | 1 台 · 4 vCPU + 8 GB RAM + 200 GB Disk |
| **操作系统** | RHEL 9.6 |
| **逻辑主机名（示例）** | `Chaos.example.com` |

### ④ n8n · 工作流编排

| 属性 | 说明 |
| --- | --- |
| **职责** | 运行 n8n Workflow，衔接告警通知、Webhook 与下游自动化动作 |
| **资源配置** | 1 台 · 4 vCPU + 8 GB RAM + 500 GB Disk |
| **操作系统** | RHEL 9.6 |
| **逻辑主机名（示例）** | `n8n.example.com` |

### ⑤ Prometheus Server · 可观测与告警源

| 属性 | 说明 |
| --- | --- |
| **职责** | 指标采集、规则评估、告警路由；安装 Prometheus、Alertmanager、node_exporter |
| **资源配置** | 1 台 · 4 vCPU + 8 GB RAM + 200 GB Disk |
| **操作系统** | RHEL 9.6 |
| **逻辑主机名（示例）** | `prometheus.example.com` |

---

## AI 智能体组件（可选）

> 未部署 AI 栈时，AAP + EDA + ChatOps 仍可演示自动化与事件驱动能力；启用 Hermes + vLLM + RAG 后可完整演示 Part I 中的 Agentic 场景。

### ⑥ Hermes AI Agent

| 属性 | 说明 |
| --- | --- |
| **职责** | 智能体运行时；对接 MCP、Skills、LLM，承载对话式运维能力 |
| **资源配置** | 1 台 · 2 vCPU + 4 GB RAM + 500 GB Disk |
| **操作系统** | RHEL 9.6 |
| **部署说明** | **可选**；小规模 DEMO 可与 Triage 或 n8n 同机，生产建议独立节点 |

### ⑦ vLLM + Chat LLM

| 属性 | 说明 |
| --- | --- |
| **职责** | 大模型推理服务；为 AIOps 智能体提供对话与推理能力 |
| **资源配置** | 1 台 · 16 vCPU + 64 GB RAM + 240 GB Disk + **160 GB GPU 显存** |
| **推荐模型** | Qwen3-32B（Chat Model） |
| **操作系统** | RHEL 9.6 |

### ⑧ Embedding + Reranking（RAG）

| 属性 | 说明 |
| --- | --- |
| **职责** | 向量嵌入与重排序，支撑 RAG 知识检索与根因分析 |
| **资源配置** | 1 台 · 16 vCPU + 64 GB RAM + 240 GB Disk + **160 GB GPU 显存** |
| **Embedding 可选模型** | qwen-32B / bge-m3 / gte-Qwen2-7B-instruct / Qwen3-Embedding-8B / embeddinggemma |
| **Reranking 可选模型** | bge-reranker-v2-minicpm-layerwise / Qwen3-Reranker-8B / jina-reranker-v2-base-multilingual |
| **部署说明** | 可与 vLLM 合并至同一 GPU 服务器以节省节点；高并发 DEMO 建议分机 |

---

## 逻辑拓扑与数据流

**典型告警 → 自愈链路**（以性能告警为例）：

```text
Chaos Server（注入故障）
    ↓ 指标异常
Prometheus Server（规则评估 → Alert）
    ↓ Event Stream / Webhook
AAP EDA（Rulebook 匹配 → 触发 Action）
    ↓ Job Template
AAP Playbook（采集 / 诊断 / 修复）
    ↓ 通知
Triage / Mattermost（ChatOps 人机确认）
    ↓ 可选
Hermes AI Agent + vLLM（LLM 辅助 RCA / RAG 知识检索）
```

| 阶段 | 参与组件 | 说明 |
| --- | --- | --- |
| **感知** | Chaos → Prometheus | 故障仿真产生可观测信号 |
| **决策** | AAP EDA + n8n | 规则匹配与工作流编排 |
| **执行** | AAP Playbook / MCP | 自动化采集、修复与 CMDB 回写 |
| **交互** | Mattermost + Hermes | ChatOps 对话与 AI 辅助诊断 |

---

## 参考部署实例

> 以下为现有 DEMO Center 逻辑主机名与 IP，供对照；正式规划以 [02-02 资源规划](02-02-Resources-Planning-CN.md) 为准。

| 逻辑主机名 | IP 地址 | 角色 |
| --- | --- | --- |
| `AAP2.6.exmaple.com` | 10.210.65.24 | AAP + EDA + MCP |
| `Mattermost.example.com` | 10.210.65.14 | Triage / Mattermost |
| `Chaos.example.com` | 10.210.65.148 | 故障仿真 |
| `n8n.example.com` | 10.210.65.103 | n8n 工作流 |
| `prometheus.example.com` | 10.210.65.174 | Prometheus 监控栈 |

---

## 与 Use Case 的组件映射

| Use Case（Part I） | 主要依赖组件 |
| --- | --- |
| 健康巡检 | AAP · MCP · Hermes（可选）· Mattermost |
| 漏扫与补丁 | AAP · Satellite（外联）· Mattermost |
| 故障 RCA & 辅助自愈 | Prometheus · EDA · AAP · n8n · Hermes · vLLM · RAG |
| 容量管理 | AAP · MCP · Mattermost |

> **下一步**：完成 IP / 端口 / 防火墙 / 汇总资源表 → [02-02 资源规划](02-02-Resources-Planning-CN.md)
