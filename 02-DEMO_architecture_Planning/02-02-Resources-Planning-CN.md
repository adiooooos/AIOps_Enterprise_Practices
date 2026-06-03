# 资源规划

> **状态**：20260603 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 核心洞察

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **规划分层** | 02-01 定义「装什么」；本章固定「每台多少资源、用什么 IP、开哪些端口」——部署前必须冻结的基线表。 |
| ② | **两种 Profile** | **Profile A（核心 5 台）** 可演示 EDA + 自动化 + ChatOps；**Profile B（+AI 栈）** 才具备完整 Agentic / RAG 能力。 |
| ③ | **网络前提** | 默认所有节点位于同一管理网段（示例 `10.210.65.0/24`），节点间 SSH / HTTPS / Webhook 互通；出站 Internet 按需开放（Satellite、模型下载等）。 |

## Executive Summary: 资源汇总

**执行摘要**：AIOps DEMO 环境资源规划总览（RHEL 9.6）

| 部署 Profile | 节点数 | vCPU 合计 | 内存合计 | 磁盘合计 | GPU 显存 |
| --- | --- | --- | --- | --- | --- |
| **A · 核心平台** | 5 | 24 | 52 GB | 1.9 TB | — |
| **B · + Hermes** | +1 | +2 | +4 GB | +500 GB | — |
| **C · + vLLM（独立 GPU 节点）** | +1 | +16 | +64 GB | +240 GB | 160 GB |
| **D · + RAG（独立 GPU 节点）** | +1 | +16 | +64 GB | +240 GB | 160 GB |
| **E · 完整 DEMO（A+B+C，RAG 与 vLLM 同机）** | 7 | 42 | 120 GB | 2.64 TB | 160 GB |

> 组件职责说明见 [02-01 组件与环境规划](02-01-Components-And-Environments-CN.md)。

---

## 计算与存储资源明细

### 核心平台（Profile A · 必选）

| 组件 | 节点数 | vCPU | 内存 | 磁盘 | 操作系统 | 说明 |
| --- | --- | --- | --- | --- | --- | --- |
| **AAP**（含 MCP + EDA） | 1 | 8 | 20 GB | 500 GB | RHEL 9.6 | 自动化中枢；含 Web UI、EDA Event Stream、MCP Server |
| **Triage**（含 Mattermost） | 1 | 4 | 8 GB | 500 GB | RHEL 9.6 | 分诊台；ChatOps 统一入口 |
| **Chaos Server** | 1 | 4 | 8 GB | 200 GB | RHEL 9.6 | 故障仿真靶机 |
| **n8n** | 1 | 4 | 8 GB | 500 GB | RHEL 9.6 | 工作流编排引擎 |
| **Prometheus Server** | 1 | 4 | 8 GB | 200 GB | RHEL 9.6 | Prometheus + Alertmanager + node_exporter |

### AI 智能体栈（Profile B～E · 可选）

| 组件 | 节点数 | vCPU | 内存 | 磁盘 | GPU 显存 | 操作系统 | 说明 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Hermes AI Agent** | 1 | 2 | 4 GB | 500 GB | — | RHEL 9.6 | 智能体运行时；小规模 DEMO 可与 Triage 同机 |
| **vLLM + Chat LLM** | 1 | 16 | 64 GB | 240 GB | **160 GB** | RHEL 9.6 | 对话模型 · 推荐 **Qwen3-32B** |
| **Embedding** | 1* | 16 | 64 GB | 240 GB | **160 GB** | RHEL 9.6 | RAG 向量嵌入；*可与 vLLM / Reranking 同机 |
| **Reranking** | 1* | — | — | — | — | RHEL 9.6 | RAG 重排序；*通常与 Embedding 同进程部署 |

---

## AI 模型选型参考

### Chat LLM（vLLM）

| 属性 | 建议值 |
| --- | --- |
| **推理框架** | vLLM |
| **推荐模型** | Qwen3-32B |
| **硬件门槛** | 32B 参数量 · **160 GB GPU 显存** + 16 vCPU + 64 GB RAM + 240 GB Disk |
| **典型 API 端口** | `8000`（OpenAI 兼容 REST） |

### Embedding（RAG 向量）

| 可选模型 | 说明 |
| --- | --- |
| qwen-32B | 通用文本嵌入 |
| bge-m3 | 多语言向量检索 |
| gte-Qwen2-7B-instruct | 指令微调嵌入 |
| Qwen3-Embedding-8B | Qwen3 系列嵌入 |
| embeddinggemma | 轻量嵌入方案 |

### Reranking（RAG 重排序）

| 可选模型 | 说明 |
| --- | --- |
| bge-reranker-v2-minicpm-layerwise | 分层重排序 |
| Qwen3-Reranker-8B | Qwen3 系列重排序 |
| jina-reranker-v2-base-multilingual | 多语言重排序 |

> **合并部署建议**：DEMO 环境可将 Embedding + Reranking + vLLM 部署于同一 GPU 节点（Profile E），以节省 1 台 GPU 服务器；生产或压测场景建议分机。

---

## 网络与 IP 规划

**网段假设**：`10.210.65.0/24`（管理网；可按实际环境替换，部署前全局替换）

| 逻辑主机名 | IP 地址 | 角色 | Profile |
| --- | --- | --- | --- |
| `AAP2.6.exmaple.com` | 10.210.65.24 | AAP + EDA + MCP | A |
| `Mattermost.example.com` | 10.210.65.14 | Triage / Mattermost | A |
| `Chaos.example.com` | 10.210.65.148 | 故障仿真靶机 | A |
| `n8n.example.com` | 10.210.65.103 | n8n 工作流 | A |
| `prometheus.example.com` | 10.210.65.174 | Prometheus 监控栈 | A |
| `hermes.example.com` | 10.210.65.25 | Hermes AI Agent | B（示例，待分配） |
| `vllm.example.com` | 10.210.65.26 | vLLM + RAG 推理 | C/E（示例，待分配） |

### DNS / 主机名要求

| 要求 | 说明 |
| --- | --- |
| **正向解析** | 各节点 FQDN 可解析至上表 IP（/etc/hosts 或内网 DNS） |
| **反向解析** | 建议配置 PTR，AAP Inventory 与 TLS 证书校验更顺畅 |
| **时间同步** | 所有节点启用 chronyd，EDA / 告警时间戳对齐 |
| **SSH 互信** | AAP 执行节点 → 被管主机（Chaos 等）密钥或凭证就绪 |

---

## 端口与通信矩阵

### 核心服务端口

| 组件 | 端口 | 协议 | 监听地址 | 用途 | 访问来源 |
| --- | --- | --- | --- | --- | --- |
| **AAP Web UI / API** | 443 | HTTPS | AAP 节点 | 控制台、REST API、EDA Event Stream | 管理员浏览器、Hermes、Webhook |
| **AAP EDA Event Stream** | 443 | HTTPS | AAP 节点 | Prometheus / Alertmanager 告警 POST | Prometheus、n8n |
| **Mattermost** | 8065 | HTTP/HTTPS | Triage 节点 | Web UI、Bot Webhook | 浏览器、Hermes、AAP 通知 |
| **n8n** | 5678 | HTTP/HTTPS | n8n 节点 | Web UI、Workflow Webhook | 浏览器、AAP EDA、Prometheus |
| **Prometheus** | 9090 | HTTP | Prometheus 节点 | 指标查询、规则评估 UI | 浏览器、Grafana（可选） |
| **Alertmanager** | 9093 | HTTP | Prometheus 节点 | 告警路由、Silence 管理 | Prometheus、AAP EDA |
| **node_exporter** | 9100 | HTTP | 各被管节点 | 主机指标暴露 | Prometheus Server |
| **SSH** | 22 | TCP | 全部 Linux 节点 | Ansible / MCP 远程执行 | AAP、管理员 |

### AI 栈端口（Profile B～E）

| 组件 | 端口 | 协议 | 用途 | 访问来源 |
| --- | --- | --- | --- | --- |
| **vLLM** | 8000 | HTTP | OpenAI 兼容 Chat Completions API | Hermes AI Agent |
| **Embedding 服务** | 8001 | HTTP | 向量嵌入 API（示例） | Hermes / RAG Pipeline |
| **Reranking 服务** | 8002 | HTTP | 重排序 API（示例） | Hermes / RAG Pipeline |
| **Hermes Agent** | 8080 | HTTP | Agent API / Webhook（示例） | Mattermost Bot、MCP Client |
| **MCP for SSH** | 8000 | HTTP | SSH MCP Server（容器映射示例 3000→8000） | Hermes / Cursor |

> 端口 8001 / 8002 / 8080 为 DEMO 建议值，实际以部署脚本或 systemd 配置为准；**部署前须在本表冻结最终端口**。

### 关键通信链路

| 源 | 目标 | 端口 | 用途 |
| --- | --- | --- | --- |
| Prometheus | Chaos / 各节点 | 9100 | 拉取 node_exporter 指标 |
| Alertmanager / Prometheus | AAP EDA | 443 | 推送告警 Event |
| AAP | Chaos / 被管主机 | 22 | Ansible Playbook 执行 |
| AAP EDA | n8n | 5678 | Webhook 触发工作流 |
| n8n | AAP / Mattermost | 443 / 8065 | 下游自动化与通知 |
| Hermes | vLLM / MCP / AAP | 8000 / 443 | LLM 推理与工具调用 |
| 管理员 | 各 Web UI | 443 / 8065 / 5678 / 9090 | 日常运维与 DEMO 演示 |

---

## 防火墙策略要点

| 节点 | 建议放行（入站） | 说明 |
| --- | --- | --- |
| **AAP** | 443/tcp ← 管理网段 | Web UI + EDA；限制源地址为 `10.210.65.0/24` |
| **Triage / Mattermost** | 8065/tcp ← 管理网段 | ChatOps 入口 |
| **n8n** | 5678/tcp ← 管理网段 + AAP | Web UI + Webhook |
| **Prometheus** | 9090, 9093, 9100/tcp ← 管理网段 | 监控 UI + 自采集 |
| **Chaos / 被管主机** | 9100/tcp ← Prometheus IP；22/tcp ← AAP IP | 指标暴露 + Ansible 执行 |
| **vLLM / GPU 节点** | 8000/tcp ← Hermes IP | 推理 API；**禁止**对公网暴露 |
| **全部节点** | 22/tcp ← 跳板机 / AAP | SSH 管理 |

**firewalld 示例（Prometheus 节点）**：

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9090" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9093" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9100" accept'
firewall-cmd --reload
```

---

## 磁盘分区建议

| 节点类型 | 挂载点建议 | 容量参考 | 用途 |
| --- | --- | --- | --- |
| **AAP** | `/` + `/var/lib/awx` | 500 GB | 平台数据、Job 日志、Project 同步 |
| **Prometheus** | `/` + `/opt/prometheus` | 200 GB | 时序数据（DEMO 保留 15～30 天） |
| **n8n / Triage** | `/` + 应用数据目录 | 500 GB | Workflow 导出、Mattermost 附件 |
| **Chaos** | `/` | 200 GB | 故障脚本、临时采集文件 |
| **GPU 节点** | `/` + 模型目录 | 240 GB+ | 模型权重（32B 约 60～80 GB/模型） |
| **Hermes** | `/` + 知识库 / 向量库 | 500 GB | RAG 文档、Embedding 索引 |

---

## 部署 Profile 选型

| Profile | 包含组件 | 适用场景 |
| --- | --- | --- |
| **A** | AAP + Triage + Chaos + n8n + Prometheus | EDA 告警链路、Playbook 自动化、基础 ChatOps |
| **A + B** | + Hermes（无 GPU） | Agent 框架联调；LLM 可外接 API |
| **A + B + C** | + 独立 GPU（vLLM + RAG 同机） | **推荐完整 DEMO** · Part I 全部 Agentic 场景 |
| **A + B + C + D** | vLLM 与 RAG 分机 | 高并发演示、生产级推理隔离 |

---

## 资源检查清单（部署前）

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | 虚拟机 / 物理机规格 | 各节点 ≥ 上表 vCPU / 内存 / 磁盘 |
| 2 | GPU 节点（若启用） | 显存 ≥ 160 GB；驱动 + CUDA / ROCm 就绪 |
| 3 | IP / DNS | 上表 IP 已分配，FQDN 可解析 |
| 4 | 端口 / 防火墙 | 通信矩阵端口互通，`firewall-cmd --list-all` 验证 |
| 5 | 时间同步 | `chronyc tracking` 偏差 < 1s |
| 6 | AAP → 被管主机 | SSH 免密或凭证注入成功 |
| 7 | Prometheus → 9100 | `curl http://<target>:9100/metrics` 可达 |
| 8 | 出站网络 | Satellite、容器镜像、模型下载（按需） |

> **上一步**：[02-01 组件与环境规划](02-01-Components-And-Environments-CN.md)  
> **下一步**：Part III → [03-01 AAP 部署先决条件](../03-aap/03-01-AAP-Prerequisites-CN.md)（待写）
