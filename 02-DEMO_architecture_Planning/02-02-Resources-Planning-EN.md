# Resources Planning

> **Status**: 20260603 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Insight

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Planning Layers** | [02-01](02-01-Components-And-Environments-EN.md) defines *what to deploy*; this chapter freezes *how much per node, which IP, which ports* — the baseline table required before deployment. |
| ② | **Two Profiles** | **Profile A (5 core nodes)** supports EDA + automation + ChatOps; **Profile B (+ AI stack)** unlocks full Agentic / RAG capabilities. |
| ③ | **Network Assumption** | All nodes reside on a shared management subnet (example `10.210.65.0/24`); SSH / HTTPS / Webhook connectivity between nodes; outbound Internet opened as needed (Satellite, model downloads, etc.). |

## Executive Summary: Resource Overview

**Executive summary**: AIOps DEMO environment resource planning (RHEL 9.6)

| Deployment Profile | Nodes | Total vCPU | Total RAM | Total Disk | GPU VRAM |
| --- | --- | --- | --- | --- | --- |
| **A · Core Platform** | 5 | 24 | 52 GB | 1.9 TB | — |
| **B · + Hermes** | +1 | +2 | +4 GB | +500 GB | — |
| **C · + vLLM (dedicated GPU node)** | +1 | +16 | +64 GB | +240 GB | 160 GB |
| **D · + RAG (dedicated GPU node)** | +1 | +16 | +64 GB | +240 GB | 160 GB |
| **E · Full DEMO (A+B+C, RAG co-located with vLLM)** | 7 | 42 | 120 GB | 2.64 TB | 160 GB |

> For component roles, see [02-01 Components & Environments](02-01-Components-And-Environments-EN.md).

---

## Compute & Storage Details

### Core Platform (Profile A · Required)

| Component | Nodes | vCPU | RAM | Disk | OS | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **AAP** (incl. MCP + EDA) | 1 | 8 | 20 GB | 500 GB | RHEL 9.6 | Automation hub; Web UI, EDA Event Stream, MCP Server |
| **Triage** (incl. Mattermost) | 1 | 4 | 8 GB | 500 GB | RHEL 9.6 | Triage desk; ChatOps unified entry |
| **Chaos Server** | 1 | 4 | 8 GB | 200 GB | RHEL 9.6 | Fault injection target |
| **n8n** | 1 | 4 | 8 GB | 500 GB | RHEL 9.6 | Workflow orchestration engine |
| **Prometheus Server** | 1 | 4 | 8 GB | 200 GB | RHEL 9.6 | Prometheus + Alertmanager + node_exporter |

### AI Agent Stack (Profile B–E · Optional)

| Component | Nodes | vCPU | RAM | Disk | GPU VRAM | OS | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Hermes AI Agent** | 1 | 2 | 4 GB | 500 GB | — | RHEL 9.6 | Agent runtime; may co-locate with Triage in small DEMO |
| **vLLM + Chat LLM** | 1 | 16 | 64 GB | 240 GB | **160 GB** | RHEL 9.6 | Chat model · recommended **Qwen3-32B** |
| **Embedding** | 1* | 16 | 64 GB | 240 GB | **160 GB** | RHEL 9.6 | RAG vector embedding; *may share node with vLLM / Reranking |
| **Reranking** | 1* | — | — | — | — | RHEL 9.6 | RAG reranking; *typically deployed with Embedding |

---

## AI Model Selection Reference

### Chat LLM (vLLM)

| Attribute | Recommended Value |
| --- | --- |
| **Inference Framework** | vLLM |
| **Recommended Model** | Qwen3-32B |
| **Hardware Threshold** | 32B parameters · **160 GB GPU VRAM** + 16 vCPU + 64 GB RAM + 240 GB Disk |
| **Typical API Port** | `8000` (OpenAI-compatible REST) |

### Embedding (RAG Vectors)

| Optional Model | Notes |
| --- | --- |
| qwen-32B | General-purpose text embedding |
| bge-m3 | Multilingual vector retrieval |
| gte-Qwen2-7B-instruct | Instruction-tuned embedding |
| Qwen3-Embedding-8B | Qwen3 series embedding |
| embeddinggemma | Lightweight embedding option |

### Reranking (RAG Rerank)

| Optional Model | Notes |
| --- | --- |
| bge-reranker-v2-minicpm-layerwise | Layerwise reranking |
| Qwen3-Reranker-8B | Qwen3 series reranker |
| jina-reranker-v2-base-multilingual | Multilingual reranking |

> **Co-location Tip**: For DEMO, Embedding + Reranking + vLLM can share one GPU node (Profile E) to save a server; production or load-test scenarios should use separate nodes.

---

## Network & IP Planning

**Subnet assumption**: `10.210.65.0/24` (management network; replace globally before deployment if different)

| Logical Hostname | IP Address | Role | Profile |
| --- | --- | --- | --- |
| `AAP2.6.exmaple.com` | 10.210.65.24 | AAP + EDA + MCP | A |
| `Mattermost.example.com` | 10.210.65.14 | Triage / Mattermost | A |
| `Chaos.example.com` | 10.210.65.148 | Fault injection target | A |
| `n8n.example.com` | 10.210.65.103 | n8n workflows | A |
| `prometheus.example.com` | 10.210.65.174 | Prometheus monitoring stack | A |
| `hermes.example.com` | 10.210.65.25 | Hermes AI Agent | B (example, TBD) |
| `vllm.example.com` | 10.210.65.26 | vLLM + RAG inference | C/E (example, TBD) |

### DNS / Hostname Requirements

| Requirement | Description |
| --- | --- |
| **Forward DNS** | Each node FQDN resolves to the IP above (/etc/hosts or internal DNS) |
| **Reverse DNS** | PTR records recommended for smoother AAP Inventory and TLS validation |
| **Time Sync** | chronyd enabled on all nodes; EDA / alert timestamps aligned |
| **SSH Trust** | AAP execution node → managed hosts (Chaos, etc.) keys or credentials ready |

---

## Port & Communication Matrix

### Core Service Ports

| Component | Port | Protocol | Bind Address | Purpose | Access From |
| --- | --- | --- | --- | --- | --- |
| **AAP Web UI / API** | 443 | HTTPS | AAP node | Console, REST API, EDA Event Stream | Admin browser, Hermes, Webhooks |
| **AAP EDA Event Stream** | 443 | HTTPS | AAP node | Prometheus / Alertmanager alert POST | Prometheus, n8n |
| **Mattermost** | 8065 | HTTP/HTTPS | Triage node | Web UI, Bot Webhook | Browser, Hermes, AAP notifications |
| **n8n** | 5678 | HTTP/HTTPS | n8n node | Web UI, Workflow Webhook | Browser, AAP EDA, Prometheus |
| **Prometheus** | 9090 | HTTP | Prometheus node | Metrics query, rules UI | Browser, Grafana (optional) |
| **Alertmanager** | 9093 | HTTP | Prometheus node | Alert routing, silence management | Prometheus, AAP EDA |
| **node_exporter** | 9100 | HTTP | All managed nodes | Host metrics exposure | Prometheus Server |
| **SSH** | 22 | TCP | All Linux nodes | Ansible / MCP remote execution | AAP, administrators |

### AI Stack Ports (Profile B–E)

| Component | Port | Protocol | Purpose | Access From |
| --- | --- | --- | --- | --- |
| **vLLM** | 8000 | HTTP | OpenAI-compatible Chat Completions API | Hermes AI Agent |
| **Embedding Service** | 8001 | HTTP | Vector embedding API (example) | Hermes / RAG Pipeline |
| **Reranking Service** | 8002 | HTTP | Reranking API (example) | Hermes / RAG Pipeline |
| **Hermes Agent** | 8080 | HTTP | Agent API / Webhook (example) | Mattermost Bot, MCP Client |
| **MCP for SSH** | 8000 | HTTP | SSH MCP Server (container map example 3000→8000) | Hermes / Cursor |

> Ports 8001 / 8002 / 8080 are DEMO suggestions; actual values follow deployment scripts or systemd units — **freeze final ports in this table before go-live**.

### Key Communication Paths

| Source | Target | Port | Purpose |
| --- | --- | --- | --- |
| Prometheus | Chaos / all nodes | 9100 | Scrape node_exporter metrics |
| Alertmanager / Prometheus | AAP EDA | 443 | Push alert events |
| AAP | Chaos / managed hosts | 22 | Ansible Playbook execution |
| AAP EDA | n8n | 5678 | Webhook workflow trigger |
| n8n | AAP / Mattermost | 443 / 8065 | Downstream automation and notifications |
| Hermes | vLLM / MCP / AAP | 8000 / 443 | LLM inference and tool calls |
| Administrators | All Web UIs | 443 / 8065 / 5678 / 9090 | Daily ops and DEMO presentation |

---

## Firewall Policy Essentials

| Node | Recommended Inbound | Notes |
| --- | --- | --- |
| **AAP** | 443/tcp ← management subnet | Web UI + EDA; restrict source to `10.210.65.0/24` |
| **Triage / Mattermost** | 8065/tcp ← management subnet | ChatOps entry |
| **n8n** | 5678/tcp ← management subnet + AAP | Web UI + Webhooks |
| **Prometheus** | 9090, 9093, 9100/tcp ← management subnet | Monitoring UI + self-scrape |
| **Chaos / managed hosts** | 9100/tcp ← Prometheus IP; 22/tcp ← AAP IP | Metrics + Ansible execution |
| **vLLM / GPU node** | 8000/tcp ← Hermes IP | Inference API; **do not** expose to public Internet |
| **All nodes** | 22/tcp ← bastion / AAP | SSH administration |

**firewalld example (Prometheus node)**:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9090" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9093" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9100" accept'
firewall-cmd --reload
```

---

## Disk Partition Recommendations

| Node Type | Mount Points | Capacity | Purpose |
| --- | --- | --- | --- |
| **AAP** | `/` + `/var/lib/awx` | 500 GB | Platform data, job logs, project sync |
| **Prometheus** | `/` + `/opt/prometheus` | 200 GB | Time-series data (DEMO retention 15–30 days) |
| **n8n / Triage** | `/` + app data dir | 500 GB | Workflow exports, Mattermost attachments |
| **Chaos** | `/` | 200 GB | Fault scripts, temporary collection files |
| **GPU node** | `/` + model directory | 240 GB+ | Model weights (~60–80 GB per 32B model) |
| **Hermes** | `/` + knowledge / vector store | 500 GB | RAG documents, embedding indexes |

---

## Deployment Profile Selection

| Profile | Components | Use Case |
| --- | --- | --- |
| **A** | AAP + Triage + Chaos + n8n + Prometheus | EDA alert chain, Playbook automation, basic ChatOps |
| **A + B** | + Hermes (no GPU) | Agent framework integration; external LLM API |
| **A + B + C** | + dedicated GPU (vLLM + RAG co-located) | **Recommended full DEMO** · all Part I Agentic scenarios |
| **A + B + C + D** | vLLM and RAG on separate nodes | High-concurrency demo, production-grade inference isolation |

---

## Pre-Deployment Checklist

| # | Check Item | Pass Criteria |
| --- | --- | --- |
| 1 | VM / bare-metal specs | Each node ≥ vCPU / RAM / disk in tables above |
| 2 | GPU node (if enabled) | VRAM ≥ 160 GB; driver + CUDA / ROCm ready |
| 3 | IP / DNS | IPs assigned; FQDN resolves |
| 4 | Ports / firewall | Communication matrix ports reachable; verify `firewall-cmd --list-all` |
| 5 | Time sync | `chronyc tracking` drift < 1s |
| 6 | AAP → managed hosts | SSH key or credential injection succeeds |
| 7 | Prometheus → 9100 | `curl http://<target>:9100/metrics` reachable |
| 8 | Outbound network | Satellite, container images, model downloads (as needed) |

> **Previous**: [02-01 Components & Environments](02-01-Components-And-Environments-EN.md)  
> **Next**: Part III → [03-01 AAP Prerequisites](../03-aap/03-01-AAP-Prerequisites-EN.md) (TBD)
