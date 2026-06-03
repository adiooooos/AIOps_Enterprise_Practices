# Use Cases - Scenario Introduction & DEMO Index

> **Status**: 20260603 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Insight

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Scenario Layering** | The DEMO Center covers Chat, Schedule, and Event trigger types, progressing from low-risk inspections to high-value incident closed loops. |
| ② | **End-to-End Closure** | Each scenario demonstrates the full chain of trigger → orchestration → execution → write-back / reporting, not a single-tool demo. |
| ③ | **Red Hat Differentiation** | EDA forensics, intelligent denoising, RAG accumulation, and Policy Enforcement form Red Hat AIOps core competencies. |

## DEMO Scenario Index

| Scenario | Trigger | Theme | DEMO Value |
| --- | --- | --- | --- |
| **Use Case 1** | Chat | Intelligent health check | One-sentence trigger for network-wide concurrent inspection with second-level report delivery |
| **Use Case 2** | Chat + Schedule | Intelligent patch management | Emergency CVE and periodic patching dual modes with full HITL + Policy |
| **Use Case 3** | Event (EDA) | Process Hung RCA | Alert-time forensics → AI analysis → trusted healing, end-to-end closed loop |
| **Use Case 4** | Chat + Policy | Web configuration compliance block | Policy Enforcement rejects unsafe operations before execution |

---

## Use Case 1: Chat Trigger based Health Check

**Scenario 1**: Intelligent health check triggered by chat

**Scenario description**: Administrator's single-sentence command → AI Agent takes over → AAP network-wide concurrent inspection → report returned to the chat window in seconds.

### Workflow

| Step | Phase | Description |
| --- | --- | --- |
| 1 | **Command Input** | The administrator issues inspection instructions in natural language through the Chat Tool. |
| 2 | **Workflow Trigger** | Chat interaction triggers the AI Agent Workflow and begins task takeover. |
| 3 | **Template Match** | The AI Agent calls MCP for AAP to query the health inspection Job Template. |
| 4 | **Parallel Execution** | AAP performs concurrent OS / Network inspections; MCP for SSH inspects K8s clusters. |
| 5 | **CMDB Sync** | The AI Agent updates configuration information and status back to CMDB. |
| 6 | **Report Delivery** | Generates a report and automatically pushes it to the Chat Tool and email recipients. |

### Architecture Layers Involved

| Layer | Components |
| --- | --- |
| **Interaction** | Admin · Chat Bot · Chat Tool |
| **AI Orchestration** | Trigger · LLM · RAG · MCP Client · AI Agent |
| **Triage & Execution** | Automation Controller · Scenario JTs |
| **Data Layer** | CMDB · Configurations · Current Status |

---

## Use Case 2: Chat & Schedule Trigger based Patching

**Scenario 2**: Intelligent patch management with chat + schedule dual triggers

**Scenario description**: Emergency CVE → Chat Trigger; monthly patching → Schedule Trigger; full-process HITL approval + Policy Enforcement.

### Workflow

| Step | Phase | Description |
| --- | --- | --- |
| 1 | **Vuln Scan Cmd** | Administrator issues periodic OS vulnerability scan and patch status query commands. |
| 2 | **Workflow Trigger** | Chat interaction triggers the AI Agent vulnerability scanning workflow. |
| 3 | **Template Match** | AI Agent calls MCP for AAP to match vulnerability scanning and patch installation Job Templates. |
| 4 | **High-Risk ID** | Returns vulnerability scan results and issues patch installation for high-risk targets. |
| 5 | **Install & Sync** | MCP for AAP executes patch installation and automatically updates Satellite status. |
| 6 | **Compliance Rpt** | Generates vulnerability and patch compliance report, pushed to chat platforms and email. |

### Dual Trigger Mode Comparison

| Dimension | Chat Trigger · Emergency Response | Schedule Trigger · Periodic Governance |
| --- | --- | --- |
| **Applicable Scenarios** | Emergency CVE (CVSS ≥ 7.0) / 0-Day vulnerability response | Monthly routine patching / quarterly security baseline remediation |
| **Response Time** | Compressed from days to hours | Aligned with maintenance windows; automated Canary → grayscale → full rollout |
| **Approval Mode** | One-click confirmation in Chat + mandatory HITL audit | Policy Enforcement prohibits non-HITL execution for high-risk targets |
| **Typical Process** | Vuln scan → impact assessment → one-click confirm → auto remediation | Cron schedule → batch release → health verification → automatic rollback |

---

## Use Case 3: Event Trigger Based RCA for Process Hung

**Scenario 3**: EDA-driven Process Hung root cause analysis (known failure types)

**Scenario description**: From alert-time forensics → AI analysis → RCA + Resolution → HITL self-healing, end-to-end closed loop.

> **Red Hat Core Competency**: EDA + intelligent denoising + RAG radically cures the chronic problems of "lost evidence and difficult root cause finding."

### End-to-End Flow

| Phase | Step | Action | Description |
| --- | --- | --- | --- |
| **Observe** | ① | Anomaly perception | Observability detects process hang (high CPU / D State). |
| | ② | Context collection | Observability pushes context to EDA. |
| **EDA** | ③ | Automated diagnosis | Identify failure type → trigger automated RCA diagnosis. |
| | ④ | ITSM ticket creation | Ansible automatically creates ITSM ticket for tracking. |
| | ⑤ | Data denoising | Triage Server performs data transformation, cleaning, denoising, and normalization. |
| **AI Analysis** | ⑥ | Data forwarding | Cleaned data is sent to AI Agent. |
| | ⑦ | Initial AI judgment | LLM + RAG outputs initial RCA report + remediation suggestions. |
| | ⑧ | Interactive consultation | Experts collaborate with Agent via ChatOps + MCP to confirm final RCA. |
| | ⑪ | RAG accumulation | Generate RAG knowledge base source files and auto-update. |
| **Trusted Execution** | ⑨A | AAP self-healing (Policy) | MCP for AAP performs self-healing under Policy Enforcement. |
| | ⑨B | SSH self-healing (Approval) | After administrator approval, MCP for SSH performs self-healing. |
| | ⑩ | ITSM ticket closing | ChatOps triggers AI Agent to close the ITSM ticket. |

### Use Case 3 (cont.): Red Hat Experience · 5-Step Automated Diagnostic Collection

**Red Hat Core Competency**: RCA algorithms distilled from 30 years of technical support experience → automated diagnosis.

| Step | Command | Purpose |
| --- | --- | --- |
| 1 | `top -b -n 1 \| head -n 20` | Capture system load snapshot and list top CPU-consuming processes. |
| 2 | `pidstat -u -p ALL 1 5` | Record CPU usage statistics for all processes over 5-second intervals. |
| 3 | `ps -eo pid,%cpu` | Identify specific process PIDs using > 90% CPU. |
| 4 | `perf record -p [PID] -a -g --call-graph dwarf` | Trigger function-level performance profiling and capture call stack. |
| 5 | `perf report -g --stdio` | Generate performance analysis report readable by AI Agent. |

### Use Case 3 (cont.): Red Hat Best Practices · 4-Step Data Cleansing

**Red Hat Best Practices**: Let AI see "high-quality signals" instead of "raw noise."

| Step | Phase | Description |
| --- | --- | --- |
| 1 | **Threshold Filtering** | Apply CPU > 90% threshold filter to keep only truly abnormal processes and filter out low CPU noise. |
| 2 | **Time-Window Statistics** | Analyze 5-second interval statistics to keep only PIDs with sustained high CPU usage and eliminate transient spikes. |
| 3 | **List Truncation (Top-N)** | Run Top-N Limit algorithm to remove long-tail noise from the process list. |
| 4 | **Data Aggregation** | Execute data merging algorithm to aggregate different diagnostic outputs into a unified JSON file format. |

**Output**: Unified format JSON file → sent to AI Agent for further processing and intelligent reasoning.

### Use Case 3 (cont.): Red Hat Core Competency · RAG Source File Auto-Accumulation

Based on Red Hat's 30 years of technical support experience, automatically transform each incident into an evolving organizational knowledge base.

| Step | Phase | Description | Output |
| --- | --- | --- | --- |
| 1 | **Admin Generates RCA Report** | Administrator requests via ChatOps: "Please generate an RCA Report based on this Process Hung failure"; AI Agent integrates collected data + denoising results + analysis process; auto-uploads to Google Drive after completion. | Standardized RCA Report (Markdown + JSON) |
| 2 | **N8N Workflow Auto-Loads to RAG** | N8N workflow monitors Google Drive directory for new files; automatically triggers KB generation based on Red Hat best practices; auto-loads into RAG Vector Store. | Evolving organizational "digital brain" |

**Value loop**: Next similar failure → AI directly hits knowledge base → second-level diagnosis.

---

## Use Case 4: Web Issue Example — Policy Enforcement

**Scenario 4**: Web configuration compliance block example

**Red Hat Core Competency**: Policy Enforcement evaluates and rejects unsafe / non-compliant operations before execution.

### Policy Enforcement Flow

| Step | Phase | Description | Result |
| --- | --- | --- | --- |
| 1 | **AI Agent Discovers Self-Healing JT** | For identified issues, AI Agent retrieves executable atomic self-healing Job Templates through MCP for AAP. | ✓ Discovered `web_config_restore` template |
| 2 | **Policy Engine Evaluates & Rejects** | Policy execution module evaluates compliance and risk; due to `web_dev` group configuration, AAP rejects `web_config_restore` template that conflicts with group settings (restore to port 80). | ✗ Rejected unsafe operation |

### Rego Policy Example

**File**: `extra_vars_validation_update.rego`

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

**Policy logic**: `nginxport` only allows `8080` / `8081`; `index` only allows `index.html` / `index.htm`; any value outside the whitelist will be rejected by the Policy Engine.

---

## Scenario to DEMO Environment Mapping

| Scenario | Related DEMO Host / Component | Reference Chapters |
| --- | --- | --- |
| Use Case 1 | Chat Tool · AAP · CMDB | `03-aap/` · `07-triage-and-mattermost/` |
| Use Case 2 | AAP · Satellite · HITL | `03-aap/` · `08-n8n/` |
| Use Case 3 | Prometheus · EDA · Triage · n8n · RAG | `04-prometheus/` · `06-eda-rulebooks/` · `07-triage-and-mattermost/` · `08-n8n/` |
| Use Case 4 | AAP Policy · Rego | `03-aap/` · `03-06-Automation-Decisions-CN.md` |
