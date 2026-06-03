# Use Cases — 场景介绍 & DEMO 索引

> **状态**：20260603 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 核心洞察

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **场景分层** | DEMO Center 覆盖 Chat / Schedule / Event 三类触发器，从低风险巡检到高价值故障闭环逐级递进。 |
| ② | **端到端闭环** | 每个场景均展示「触发 → 编排 → 执行 → 回写 / 报告」完整链路，而非单点工具演示。 |
| ③ | **Red Hat 差异化** | EDA 取证、智能降噪、RAG 沉淀、Policy Enforcement 构成 Red Hat AIOps 核心竞争力。 |

## DEMO 场景索引

| 场景 | 触发器 | 主题 | DEMO 价值 |
| --- | --- | --- | --- |
| **Use Case 1** | Chat | 智能健康巡检 | 一句话触发全网并发巡检，秒级报告回传 |
| **Use Case 2** | Chat + Schedule | 智能补丁管理 | 紧急 CVE 与周期补丁双模式，全程 HITL + Policy |
| **Use Case 3** | Event (EDA) | Process Hung RCA | 告警瞬间取证 → AI 分析 → 可信自愈，端到端闭环 |
| **Use Case 4** | Chat + Policy | Web 配置合规拦截 | Policy Enforcement 在执行前拒绝不安全操作 |

---

## Use Case 1: Chat Trigger based Health Check

**场景一**：基于聊天触发的智能健康巡检

**场景描述**：管理员一句话指令 → AI Agent 接管 → AAP 全网并发巡检 → 报告秒级回传聊天窗口。

### 工作流程

| Step | 阶段 | 说明 |
| --- | --- | --- |
| 1 | **指令下发**<br>Command Input | 管理员通过 Chat Tool 以自然语言下发巡检指令。 |
| 2 | **工作流触发**<br>Workflow Trigger | 聊天交互触发 AI Agent Workflow，开始接管任务。 |
| 3 | **模板匹配**<br>Template Match | AI Agent 调用 MCP for AAP，查询健康巡检 Job Template。 |
| 4 | **并发执行**<br>Parallel Execution | AAP 并发巡检 OS / Network；MCP for SSH 巡检 K8s 集群。 |
| 5 | **状态回写**<br>CMDB Sync | AI Agent 将配置信息与状态更新回 CMDB。 |
| 6 | **报告推送**<br>Report Delivery | 生成报告并自动推送至 Chat Tool 与邮件收件人。 |

### 涉及架构层

| 架构层 | 组件 |
| --- | --- |
| **交互层** Interaction | Admin · Chat Bot · Chat Tool |
| **AI 编排层** AI Orchestration | Trigger · LLM · RAG · MCP Client · AI Agent |
| **执行层** Triage & Execution | Automation Controller · Scenario JTs |
| **数据层** Data Layer | CMDB · Configurations · Current Status |

---

## Use Case 2: Chat & Schedule Trigger based Patching

**场景二**：聊天 + 周期双触发的智能补丁管理

**场景描述**：紧急 CVE → Chat Trigger；月度补丁 → Schedule Trigger；全流程 HITL 审批 + Policy Enforcement。

### 工作流程

| Step | 阶段 | 说明 |
| --- | --- | --- |
| 1 | **漏洞指令**<br>Vuln Scan Cmd | 管理员下发周期性 OS 漏洞扫描与补丁状态查询指令。 |
| 2 | **工作流触发**<br>Workflow Trigger | 聊天交互触发 AI Agent 漏洞扫描工作流。 |
| 3 | **模板匹配**<br>Template Match | AI Agent 调用 MCP for AAP，匹配漏洞扫描与补丁安装 Job Template。 |
| 4 | **高危识别**<br>High-Risk ID | 返回漏洞扫描结果，针对高危目标下发补丁安装。 |
| 5 | **安装与回写**<br>Install & Sync | MCP for AAP 执行补丁安装，并自动更新 Satellite 状态。 |
| 6 | **合规报告**<br>Compliance Rpt | 生成漏洞与补丁合规报告，推送至 Chat 平台与邮件。 |

### 双触发模式对比

| 维度 | Chat Trigger · 紧急响应模式 | Schedule Trigger · 周期治理模式 |
| --- | --- | --- |
| **适用场景** | 紧急 CVE（CVSS ≥ 7.0）/ 0-Day 漏洞响应 | 月度例行补丁 / 季度安全基线整改 |
| **响应时间** | 从「天级」压缩至「小时级」 | 对齐维护窗口，自动化 Canary → 灰度 → 全量 |
| **审批模式** | Chat 界面一键确认 + 强制 HITL 审计 | Policy Enforcement 禁止高危目标非 HITL 执行 |
| **典型流程** | 漏洞扫描 → 影响评估 → 一键确认 → 自动修复 | Cron 定时 → 批量发布 → 健康验证 → 自动回滚 |

---

## Use Case 3: Event Trigger Based RCA for Process Hung

**场景三**：EDA 驱动的进程挂起根因分析（已知故障类型）

**场景描述**：从告警瞬间取证 → AI 分析 → RCA + Resolution → HITL 自愈，端到端闭环。

> **Red Hat Core Competency**：EDA + 智能降噪 + RAG，根治「证据丢失、根因难寻」顽疾。

### 端到端流程

| 阶段 | Step | 动作 | 说明 |
| --- | --- | --- | --- |
| **Observe 观测取证** | ① | 异常感知 | Observability 检测进程假死（CPU 高 / D State）。 |
| | ② | 上下文采集 | Observability 推送上下文给 EDA。 |
| **EDA 事件驱动** | ③ | 自动诊断 | 识别故障类型 → 触发自动 RCA 诊断。 |
| | ④ | ITSM 建单 | Ansible 自动创建 ITSM 工单跟踪。 |
| | ⑤ | 数据降噪 | Triage Server 进行数据转换、清洗、降噪、归一化。 |
| **AI Analysis 智能分析** | ⑥ | 数据转发 | 清洗后数据送给 AI Agent。 |
| | ⑦ | AI 初判 | LLM + RAG 输出初始 RCA 报告 + 修复建议。 |
| | ⑧ | 交互问诊 | 专家通过 ChatOps + MCP 与 Agent 协同确认最终 RCA。 |
| | ⑪ | RAG 沉淀 | 生成 RAG 知识库源文件，自动更新。 |
| **Trusted Execution 可信执行** | ⑨A | AAP 自愈 (Policy) | MCP for AAP 在 Policy Enforcement 下执行自愈。 |
| | ⑨B | SSH 自愈 (审批) | 管理员审批后，MCP for SSH 执行自愈。 |
| | ⑩ | ITSM 关单 | ChatOps 触发 AI Agent 关闭 ITSM 工单。 |

### 场景三（续）：Red Hat 经验 · 5 步自动化诊断采集

**Red Hat Core Competency**：基于 30 年技术支持经验沉淀的 RCA 算法 → 自动化诊断。

| Step | 命令 | 目的 |
| --- | --- | --- |
| 1 | `top -b -n 1 \| head -n 20` | 捕获系统负载快照，列出 Top CPU 消耗进程。 |
| 2 | `pidstat -u -p ALL 1 5` | 记录所有进程在 5 秒间隔内的 CPU 使用统计。 |
| 3 | `ps -eo pid,%cpu` | 识别使用 > 90% CPU 的特定进程 PID。 |
| 4 | `perf record -p [PID] -a -g --call-graph dwarf` | 触发函数级性能 Profiling，捕获调用堆栈。 |
| 5 | `perf report -g --stdio` | 生成 AI Agent 可读的性能分析报告。 |

### 场景三（续）：Red Hat 最佳实践 · 4 步数据清洗

**Red Hat Best Practices**：让 AI 看到的不是「原始噪声」，而是「高质量信号」。

| Step | 阶段 | 说明 |
| --- | --- | --- |
| 1 | **Threshold Filtering** 阈值过滤 | 应用 CPU > 90% 阈值过滤，仅保留真正异常的进程，过滤低 CPU 噪声。 |
| 2 | **Time-Window Statistics** 时间窗口统计 | 分析 5 秒间隔统计数据，仅保留持续高 CPU 使用率的 PID，消除瞬态尖峰。 |
| 3 | **List Truncation (Top-N)** 列表截断 | 运行 Top-N Limit 算法，从进程列表中去除长尾噪声。 |
| 4 | **Data Aggregation** 数据聚合归一化 | 执行数据合并算法，将不同诊断输出聚合到统一格式的 JSON 文件。 |

**输出物**：统一格式的 JSON 文件 → 送入 AI Agent 进行进一步处理与智能推理。

### 场景三（续）：Red Hat 核心竞争力 · RAG 源文件自动沉淀

基于 Red Hat 30 年技术支持经验，自动将每一次故障转化为可演进的组织级知识库。

| Step | 阶段 | 说明 | 输出 |
| --- | --- | --- | --- |
| 1 | **Admin Generates RCA Report** | 管理员通过 ChatOps 向 ChatBot 提出请求：「请基于本次 Process Hung 故障生成 RCA Report」；AI Agent 整合采集数据 + 降噪结果 + 分析过程；撰写完成后自动上传至 Google Drive。 | 标准化 RCA Report（Markdown + JSON） |
| 2 | **N8N Workflow Auto-Loads to RAG** | N8N 工作流监听 Google Drive 目录新文件；自动触发基于 Red Hat 最佳实践的 KB 生成；自动 Loading 到 RAG Vector Store。 | 可演进的组织级「数字大脑」 |

**价值闭环**：下次同类故障 → AI 直接命中知识库 → 秒级判别。

---

## Use Case 4: Web Issue Example — Policy Enforcement

**场景四**：Web 配置合规拦截示例

**Red Hat Core Competency**：Policy Enforcement 在执行前评估并拒绝不安全 / 不合规操作。

### Policy Enforcement 流程

| Step | 阶段 | 说明 | 结果 |
| --- | --- | --- | --- |
| 1 | **AI Agent Discovers Self-Healing JT** | 针对已识别问题，AI Agent 通过 MCP for AAP 检索可执行的原子化自愈 Job Template。 | ✓ 发现 `web_config_restore` 模板 |
| 2 | **Policy Engine Evaluates & Rejects** | 策略执行模块评估合规性与风险；因 `web_dev` 组特定配置，AAP 拒绝执行与组设置冲突的 `web_config_restore` 模板（恢复至 port 80）。 | ✗ 拒绝不安全操作 |

### Rego Policy 示例

**文件**：`extra_vars_validation_update.rego`

```rego
package aap_policy_examples
import rego.v1

# Strict Mode: Key whitelist + Value whitelist
allowed_values_map := {
    "extra_var_key_nginxport": ["8080", "8081"],
    "extra_var_key_index": ["index.html", "index.htm"]
}

# Default policy: Allow, no violations
default extra_vars_validation_update = {
    "allowed": true,
    "violations": [],
}

# Main decision rule
extra_vars_validation_update := result if {
    provided_extra_vars := object.get(input, "extra_vars", {})
    invalid_value_violations := [msg |
        some key
        allowed_values := allowed_values_map[key]
        provided_value := object.get(provided_extra_vars, key, null)
        provided_value != null
        not array_contains(allowed_values, provided_value)
        msg := sprintf("Invalid value for %v", [key])
    ]
    # ... result construction
}
```

**策略逻辑**：`nginxport` 仅允许 `8080` / `8081`，`index` 仅允许 `index.html` / `index.htm`；任何超出白名单的值将被 Policy Engine 拒绝。

---

## 场景与 DEMO 环境映射
TBU
