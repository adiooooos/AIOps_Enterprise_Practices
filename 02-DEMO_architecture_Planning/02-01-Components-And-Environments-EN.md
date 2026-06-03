# AIOps Components & Environment Planning

> **Status**: 20260603 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Insight

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Layered Deployment** | The DEMO environment splits into independent nodes along observability → event-driven → automation → human interaction → AI inference for easier upgrades and fault isolation. |
| ② | **Minimum Viable Set** | **5 core servers** (AAP / Triage / Chaos / n8n / Prometheus) cover all four Part I Use Cases; the AI stack (Hermes + vLLM + RAG) is an enhancement. |
| ③ | **Spec Baseline** | Resources below are **minimum recommended per node**; IP, ports, network, and aggregate tables are in [02-02 Resources Planning](02-02-Resources-Planning-EN.md). |

## Executive Summary: Components & Resources

**Executive summary**: AIOps DEMO components and per-node resource allocation (RHEL 9.6)

| Component | Nodes | CPU | RAM | Disk | Notes |
| --- | --- | --- | --- | --- | --- |
| **AAP** (incl. EDA, MCP) | 1 | 8 vCPU | 20 GB | 500 GB | Automation hub: Playbook / Job Template / EDA Rulebook / MCP Server |
| **Triage Server** (incl. Mattermost) | 1 | 4 vCPU | 8 GB | 500 GB | Triage desk: alert collaboration, ChatOps entry |
| **Chaos Server** | 1 | 4 vCPU | 8 GB | 200 GB | Fault injection target; triggers Prometheus alert chain |
| **n8n** | 1 | 4 vCPU | 8 GB | 500 GB | Workflow orchestration; links alerts to downstream actions |
| **Prometheus Server** | 1 | 4 vCPU | 8 GB | 200 GB | Prometheus + Alertmanager + node_exporter |
| **Hermes AI Agent** | 1 | 2 vCPU | 4 GB | 500 GB | **Optional** · Agent runtime |
| **vLLM + Chat LLM** | 1 | 16 vCPU | 64 GB | 240 GB | **160 GB GPU VRAM** · Qwen3-32B chat model |
| **Embedding + Reranking** | 1 | 16 vCPU | 64 GB | 240 GB | **160 GB GPU VRAM** · RAG retrieval & rerank (may share node with vLLM) |

> **Total**: **5** core platform nodes; **1–3** additional nodes when full AI is enabled (depends on Hermes / vLLM / RAG co-location).

---

## Core Platform Components

### ① AAP · Automation & Event-Driven Hub

| Attribute | Description |
| --- | --- |
| **Role** | Runs AAP Web UI; hosts Inventory, Project, Job Template; EDA receives Prometheus alerts and triggers Rulebook / Playbook; MCP Server for AI Agent calls |
| **Key Modules** | Ansible Automation Platform · Event-Driven Ansible (EDA) · MCP for AAP |
| **Resources** | 1 node · 8 vCPU + 20 GB RAM + 500 GB Disk |
| **OS** | RHEL 9.6 |
| **Example Hostname** | `AAP2.6.exmaple.com` |

### ② Triage Server · Triage & Collaboration Entry

| Attribute | Description |
| --- | --- |
| **Role** | Alert triage desk; Mattermost deployment as ChatOps unified entry |
| **Resources** | 1 node · 4 vCPU + 8 GB RAM + 500 GB Disk |
| **OS** | RHEL 9.6 |
| **Example Hostname** | `Mattermost.example.com` |

### ③ Chaos Server · Fault Injection Target

| Attribute | Description |
| --- | --- |
| **Role** | Simulates OS-level CPU / memory / disk I/O faults; Prometheus rules fire alerts driving EDA auto-remediation |
| **Resources** | 1 node · 4 vCPU + 8 GB RAM + 200 GB Disk |
| **OS** | RHEL 9.6 |
| **Example Hostname** | `Chaos.example.com` |

### ④ n8n · Workflow Orchestration

| Attribute | Description |
| --- | --- |
| **Role** | Runs n8n workflows; connects alert notifications, Webhooks, and downstream automation |
| **Resources** | 1 node · 4 vCPU + 8 GB RAM + 500 GB Disk |
| **OS** | RHEL 9.6 |
| **Example Hostname** | `n8n.example.com` |

### ⑤ Prometheus Server · Observability & Alert Source

| Attribute | Description |
| --- | --- |
| **Role** | Metric collection, rule evaluation, alert routing; Prometheus, Alertmanager, node_exporter |
| **Resources** | 1 node · 4 vCPU + 8 GB RAM + 200 GB Disk |
| **OS** | RHEL 9.6 |
| **Example Hostname** | `prometheus.example.com` |

---

## AI Agent Components (Optional)

> Without the AI stack, AAP + EDA + ChatOps still demonstrates automation and event-driven capabilities; Hermes + vLLM + RAG unlock full Part I Agentic scenarios.

### ⑥ Hermes AI Agent

| Attribute | Description |
| --- | --- |
| **Role** | Agent runtime; connects MCP, Skills, LLM for conversational operations |
| **Resources** | 1 node · 2 vCPU + 4 GB RAM + 500 GB Disk |
| **OS** | RHEL 9.6 |
| **Deployment** | **Optional**; small DEMO may co-locate with Triage or n8n; production should use a dedicated node |

### ⑦ vLLM + Chat LLM

| Attribute | Description |
| --- | --- |
| **Role** | LLM inference service for AIOps agent dialogue and reasoning |
| **Resources** | 1 node · 16 vCPU + 64 GB RAM + 240 GB Disk + **160 GB GPU VRAM** |
| **Recommended Model** | Qwen3-32B (Chat Model) |
| **OS** | RHEL 9.6 |

### ⑧ Embedding + Reranking (RAG)

| Attribute | Description |
| --- | --- |
| **Role** | Vector embedding and reranking for RAG knowledge retrieval and RCA |
| **Resources** | 1 node · 16 vCPU + 64 GB RAM + 240 GB Disk + **160 GB GPU VRAM** |
| **Embedding Options** | qwen-32B / bge-m3 / gte-Qwen2-7B-instruct / Qwen3-Embedding-8B / embeddinggemma |
| **Reranking Options** | bge-reranker-v2-minicpm-layerwise / Qwen3-Reranker-8B / jina-reranker-v2-base-multilingual |
| **Deployment** | May merge with vLLM on one GPU server; separate nodes for high-concurrency DEMO |

---

## Logical Topology & Data Flow

**Typical alert → self-healing chain** (performance alert example):

```text
Chaos Server (inject fault)
    ↓ metric anomaly
Prometheus Server (rule eval → Alert)
    ↓ Event Stream / Webhook
AAP EDA (Rulebook match → trigger Action)
    ↓ Job Template
AAP Playbook (collect / diagnose / remediate)
    ↓ notify
Triage / Mattermost (ChatOps human confirm)
    ↓ optional
Hermes AI Agent + vLLM (LLM-assisted RCA / RAG retrieval)
```

| Phase | Components | Description |
| --- | --- | --- |
| **Sense** | Chaos → Prometheus | Fault injection produces observable signals |
| **Decide** | AAP EDA + n8n | Rule matching and workflow orchestration |
| **Execute** | AAP Playbook / MCP | Automated collection, remediation, CMDB write-back |
| **Interact** | Mattermost + Hermes | ChatOps dialogue and AI-assisted diagnosis |

---

## Reference Deployment Instance

> Example logical hostnames and IPs from the existing DEMO Center; authoritative IP / port planning is in [02-02 Resources Planning](02-02-Resources-Planning-EN.md).

| Logical Hostname | IP Address | Role |
| --- | --- | --- |
| `AAP2.6.exmaple.com` | 10.210.65.24 | AAP + EDA + MCP |
| `Mattermost.example.com` | 10.210.65.14 | Triage / Mattermost |
| `Chaos.example.com` | 10.210.65.148 | Fault injection |
| `n8n.example.com` | 10.210.65.103 | n8n workflows |
| `prometheus.example.com` | 10.210.65.174 | Prometheus monitoring stack |

---

## Use Case Component Mapping

| Use Case (Part I) | Primary Dependencies |
| --- | --- |
| Health Check | AAP · MCP · Hermes (optional) · Mattermost |
| Vuln Scan & Patching | AAP · Satellite (external) · Mattermost |
| Incident RCA & Self-Healing | Prometheus · EDA · AAP · n8n · Hermes · vLLM · RAG |
| Capacity Management | AAP · MCP · Mattermost |

> **Next**: IP / ports / firewall / aggregate resource table → [02-02 Resources Planning](02-02-Resources-Planning-EN.md)
