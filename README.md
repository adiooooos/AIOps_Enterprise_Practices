# AIOps Practices Center — Deploy & Setup Step-by-Step Guide

> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  
> **Chinee verion**: [README-CN.md](README-CN.md)  
> **Status**: 202606 Updated  

Reproduce the **GCG AIOps DEMO Center** end to end: observability, event-driven automation (EDA), workflow orchestration (n8n), ChatOps (Mattermost), and AI Agent integration (MCP for AAP / SSH).

---

## At a Glance

| Item | Summary |
| --- | --- |
| **Minimum footprint** | **5 servers** — AAP · Triage/Mattermost · Chaos · n8n · Prometheus |
| **Platform** | RHEL 9.6 · Ansible Automation Platform 2.6 · Event-Driven Ansible · Podman |
| **Demo network** | Example segment `10.210.65.0/24` (replace with your environment) |
| **Languages** | Each chapter has **`-CN.md`** (Chinese) and **`-EN.md`** (English) where applicable |
| **Source material** | Built from the live `01_AIOPS_DEMO_CENTER部署及配置` environment |

---

## Architecture (Core 5 Nodes)

```
                    ┌─────────────────────────────────────┐
                    │  Admin / ChatOps (Mattermost :8065) │
                    └──────────────────┬──────────────────┘
                                       │
     ┌──────────────┐    Event/Webhook │    AI Agent Workflows
     │ Prometheus   │◀───metrics───────┼───▶ n8n (:8080)
     │ :9090/:9093  │───alerts────────▶│         │
     └──────┬───────┘                  │         ▼
            │                           │    MCP Client
            ▼                           │         │
     ┌──────────────┐                   │    ┌────┴────┐
     │ Chaos        │◀──SSH/AAP─────────┼───▶│ AAP     │
     │ (fault tgt)  │                   │    │ :443    │
     └──────────────┘                   │    │ EDA+MCP │
                                        │    └─────────┘
                    Triage / Mattermost (:8065) + MCP for SSH (:3000)
                              10.210.65.14
```

| Host (example) | IP | Role |
| --- | --- | --- |
| `AAP2.6.exmaple.com` | 10.210.65.24 | AAP · EDA · MCP for AAP |
| `Mattermost.example.com` | 10.210.65.14 | Triage · Mattermost · MCP for SSH |
| `Chaos.example.com` | 10.210.65.148 | Fault simulation target |
| `n8n.example.com` | 10.210.65.103 | n8n workflow engine |
| `prometheus.example.com` | 10.210.65.174 | Prometheus · Alertmanager · node_exporter |

Details: [02-01 Components](02-DEMO_architecture_Planning/02-01-Components-And-Environments-EN.md) · [02-02 Resources & IP](02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md)

---

## Four DEMO Use Cases

| # | Trigger | Scenario | Key stack |
| --- | --- | --- | --- |
| **UC1** | Chat | Intelligent health check | Mattermost → n8n → MCP for AAP → Job Templates |
| **UC2** | Chat + Schedule | Patch / vulnerability management | AAP · HITL · Policy Enforcement |
| **UC3** | Event (EDA) | Process Hung RCA & self-heal | Prometheus → EDA → Triage → n8n → AI Agent |
| **UC4** | Chat + Policy | Web config compliance block | Policy Enforcement rejects unsafe JT |

Overview: [01-04 Use Cases](01-solution-introduction/01-04-Use-Cases-EN.md)

---

## Recommended Path

### Phase A — Understand (read first)

| Step | Document | Purpose |
| --- | --- | --- |
| 0 | [00-Blueprint-CN.md](00-overview/00-Blueprint-CN.md) | Series scope & conventions |
| 1 | [01-01 … 01-05](01-solution-introduction/) | Why / What / How to Land / Use Cases / ROI |
| 2 | [02-01](02-DEMO_architecture_Planning/02-01-Components-And-Environments-EN.md) · [02-02](02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md) | Components, sizing, IP & ports |

### Phase B — Deploy (follow in order)

| Order | Chapter | Document | Deploy on |
| --- | --- | --- | --- |
| 1 | **03 AAP** | [03-01](03-aap/03-01-AAP-Prerequisites-EN.md) → [03-06](03-aap/03-06-Automation-Decisions-EN.md) | AAP node |
| 2 | **04 Prometheus** | [04-01](04-prometheus/04-01-Prometheus-Config-EN.md) | Prometheus node |
| 3 | **05 Chaos** | [05-01](05-chaos/05-01-Chaos-Process-Hung-EN.md) · [05-02](05-chaos/05-02-Chaos-Network-Error-EN.md) · [05-03](05-chaos/05-03-Chaos-Other-Outage-EN.md) | Chaos node |
| 4 | **07 Triage** | [07-01](07-triage-and-ChatOps/07-01-Triage-Diagnostics-EN.md) | Triage node |
| 5 | **07 Mattermost** | [07-02](07-triage-and-ChatOps/07-02-Mattermost-Deploy-EN.md) · [07-03](07-triage-and-ChatOps/07-03-Mattermost-Config-EN.md) | Triage node |
| 6 | **08 n8n** | [08-01](08-n8n/08-01-n8n-Deploy-EN.md) · [08-02](08-n8n/08-02-n8n-Workflow-Import-EN.md) | n8n node |
| 7 | **06 EDA** | [06-01](06-EDA-rulebooks_and_actions/06-01-EDA-Artifacts-EN.md) | AAP (Rulebooks / JTs / Playbooks) |
| 8 | **09 MCP SSH** | [09-01](09-mcp-ssh/09-01-MCP-SSH-Setup-EN.md) | Triage node |
| 9 | **10 DEMO** | [10-01](10-aiops-DEMO-and-screenshots/10-01-Agent-Workflow-E2E-CN.md) · [Workflow prompts](10-aiops-DEMO-and-screenshots/Workflow提示词.md) | End-to-end validation |

> **Note**: Chapter **06 EDA** depends on AAP, Prometheus alerting, n8n webhooks, and Triage directories — deploy it after those are in place.

---

## Document Index

### 00 — Overview

| File | Description |
| --- | --- |
| [00-Blueprint-CN.md](00-overview/00-Blueprint-CN.md) | Series blueprint & writing standards |
| [00-01-Guide-How-To-Use-CN.md](00-overview/00-01-Guide-How-To-Use-CN.md) | Project background & goals |

### 01 — Solution Introduction

| CN | EN | Topic |
| --- | --- | --- |
| [01-01-CN](01-solution-introduction/01-01-Why-AIOps-传统运维瓶颈%20%26%20AI%20核心价值-CN.md) | [01-01-EN](01-solution-introduction/01-01-Why-AIOps-Traditional%20Pain%20Points%20%26%20AI%20Value-EN.md) | Why AIOps |
| [01-02-CN](01-solution-introduction/01-02-What-Is-AIOps-核心能力定义与参考架构-CN.md) | [01-02-EN](01-solution-introduction/01-02-What-Is-AIOps-%20Core%20Capability%20Definition%20and%20Reference%20Architecture-EN.md) | What is AIOps · Architecture |
| [01-03-CN](01-solution-introduction/01-03-How-To-Land-CN.md) | [01-03-EN](01-solution-introduction/01-03-How-To-Land-EN.md) | How to Land |
| [01-04-CN](01-solution-introduction/01-04-Use-Cases-CN.md) | [01-04-EN](01-solution-introduction/01-04-Use-Cases-EN.md) | Four Use Cases |
| [01-05-CN](01-solution-introduction/01-05-Biz-Value-CN.md) | [01-05-EN](01-solution-introduction/01-05-Biz-Value-EN.md) | Business Value & ROI |

### 02 — Architecture & Planning

| CN | EN | Topic |
| --- | --- | --- |
| [02-01-CN](02-DEMO_architecture_Planning/02-01-Components-And-Environments-CN.md) | [02-01-EN](02-DEMO_architecture_Planning/02-01-Components-And-Environments-EN.md) | Components & environments |
| [02-02-CN](02-DEMO_architecture_Planning/02-02-Resources-Planning-CN.md) | [02-02-EN](02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md) | Resources · IP · ports |

### 03 — Ansible Automation Platform

| CN | EN | Topic |
| --- | --- | --- |
| [03-01-CN](03-aap/03-01-AAP-Prerequisites-CN.md) | [03-01-EN](03-aap/03-01-AAP-Prerequisites-EN.md) | Prerequisites |
| [03-02-CN](03-aap/03-02-AAP-Preparation-CN.md) | [03-02-EN](03-aap/03-02-AAP-Preparation-EN.md) | Pre-deployment prep |
| [03-03-CN](03-aap/03-03-AAP-Server-Deploy-CN.md) | [03-03-EN](03-aap/03-03-AAP-Server-Deploy-EN.md) | Server install |
| [03-04-CN](03-aap/03-04-AAP-Configuration-CN.md) | [03-04-EN](03-aap/03-04-AAP-Configuration-EN.md) | License · Starter Pack |
| [03-05-CN](03-aap/03-05-AAP-MCP-For-Agent-CN.md) | [03-05-EN](03-aap/03-05-AAP-MCP-For-Agent-EN.md) | MCP for AAP → AI Agent |
| [03-06-CN](03-aap/03-06-Automation-Decisions-CN.md) | [03-06-EN](03-aap/03-06-Automation-Decisions-EN.md) | EDA / Automation Decisions |

### 04 — Prometheus

| CN | EN | Topic |
| --- | --- | --- |
| [04-01-CN](04-prometheus/04-01-Prometheus-Config-CN.md) | [04-01-EN](04-prometheus/04-01-Prometheus-Config-EN.md) | Prometheus · Alertmanager · node_exporter |

### 05 — Chaos (Fault Simulation)

| CN | EN | Topic |
| --- | --- | --- |
| [05-01-CN](05-chaos/05-01-Chaos-Process-Hung-CN.md) | [05-01-EN](05-chaos/05-01-Chaos-Process-Hung-EN.md) | UC3 · Process Hung (CPU stress) |
| [05-02-CN](05-chaos/05-02-Chaos-Network-Error-CN.md) | [05-02-EN](05-chaos/05-02-Chaos-Network-Error-EN.md) | UC2/3 · Network interface down |
| [05-03-CN](05-chaos/05-03-Chaos-Other-Outage-CN.md) | [05-03-EN](05-chaos/05-03-Chaos-Other-Outage-EN.md) | UC3 · xsos chaos (CPU / disk) |

### 06 — EDA Rulebooks & Actions

| CN | EN | Topic |
| --- | --- | --- |
| [06-01-CN](06-EDA-rulebooks_and_actions/06-01-EDA-Artifacts-CN.md) | [06-01-EN](06-EDA-rulebooks_and_actions/06-01-EDA-Artifacts-EN.md) | Rulebooks · Job Templates · Playbooks (UC1–3) |

### 07 — Triage & ChatOps

| CN | EN | Topic |
| --- | --- | --- |
| [07-01-CN](07-triage-and-ChatOps/07-01-Triage-Diagnostics-CN.md) | [07-01-EN](07-triage-and-ChatOps/07-01-Triage-Diagnostics-EN.md) | Triage directories · diagnostic samples |
| [07-02-CN](07-triage-and-ChatOps/07-02-Mattermost-Deploy-CN.md) | [07-02-EN](07-triage-and-ChatOps/07-02-Mattermost-Deploy-EN.md) | Mattermost install (Podman) |
| [07-03-CN](07-triage-and-ChatOps/07-03-Mattermost-Config-CN.md) | [07-03-EN](07-triage-and-ChatOps/07-03-Mattermost-Config-EN.md) | Webhooks · Bot · channels (screenshots) |

### 08 — n8n

| CN | EN | Topic |
| --- | --- | --- |
| [08-01-CN](08-n8n/08-01-n8n-Deploy-CN.md) | [08-01-EN](08-n8n/08-01-n8n-Deploy-EN.md) | n8n deploy (online / offline) |
| [08-02-CN](08-n8n/08-02-n8n-Workflow-Import-CN.md) | [08-02-EN](08-n8n/08-02-n8n-Workflow-Import-EN.md) | Import pre-built workflows |

### 09 — MCP for SSH

| CN | EN | Topic |
| --- | --- | --- |
| [09-01-CN](09-mcp-ssh/09-01-MCP-SSH-Setup-CN.md) | [09-01-EN](09-mcp-ssh/09-01-MCP-SSH-Setup-EN.md) | ssh-mcp container · pubkey · N8N integration |

### 10 — DEMO & Screenshots

| File | Description |
| --- | --- |
| [10-01-Agent-Workflow-E2E-CN.md](10-aiops-DEMO-and-screenshots/10-01-Agent-Workflow-E2E-CN.md) | End-to-end Agent workflow (WIP) |
| [Workflow提示词.md](10-aiops-DEMO-and-screenshots/Workflow提示词.md) | n8n AI Agent system prompt template |

---

## Key Endpoints (DEMO)

| Service | URL (example) | Chapter |
| --- | --- | --- |
| AAP Web UI | `https://10.210.65.24` | 03 |
| Prometheus | `http://10.210.65.174:9090` | 04 |
| Alertmanager | `http://10.210.65.174:9093` | 04 |
| Mattermost | `http://10.210.65.14:8065` | 07 |
| n8n | `http://10.210.65.103:8080` | 08 |
| MCP for AAP | `http://10.210.65.24:3000/mcp/<toolset>` | 03-05 |
| MCP for SSH | `http://10.210.65.14:3000/mcp` | 09 |

---

## How to Use This Guide

1. **Pick your language** — read `-CN.md` or `-EN.md`; commands and configs are identical in both.
2. **Freeze your IP plan** — copy the table in [02-02](02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md) and replace `10.210.65.x` globally if needed.
3. **Deploy in Phase B order** — do not skip prerequisites (SSH keys, firewalld, offline bundles).
4. **Validate per chapter** — each guide includes expected output; use UC1 Chat health check as the final smoke test.
5. **Slides & ROI** — solution narrative lives in `10_AIOPS_Solution_Slides/`; cross-reference [01-04](01-solution-introduction/01-04-Use-Cases-EN.md) and [01-05](01-solution-introduction/01-05-Biz-Value-EN.md).

---

## Related Resources

| Resource | Location |
| --- | --- |
| Live config workspace | `01_AIOPS_DEMO_CENTER部署及配置/` |
| Solution slides | `10_AIOPS_Solution_Slides/` |
| Architecture overview | `01_AAP2.6.exmaple.com/00-AIOPS架构.md` |
| GitHub (target) | [AIOps_Enterprise_Practices](https://github.com/adiooooos/AIOps_Enterprise_Practices) |

---

*Red Hat AIOps Solution — EDA + Agentic AI for intelligent operations.*
