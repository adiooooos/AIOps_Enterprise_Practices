# Biz Value - Value Proposition & ROI Analysis

> **Status**: 20260603 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Insight

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Essential Shift** | AIOps is not about tool replacement, but a reshaping of production relations: lowering the technical barrier while fully releasing expert capacity. |
| ② | **Quantitative Baseline** | The ROI analysis below is based on a **100-unit scale** simulation; step times are cumulative totals. |
| ③ | **Four-Dimension Value** | Efficiency ROI, human-efficiency ROI, quality / compliance ROI, and accessibility ROI form a complete business value loop. |

## Executive Summary: AIOps Business Value

**Executive summary**: AIOps business value overview (100-unit scale)

| Core Scenario | Before · Traditional Mode | After · AIOps Mode | Efficiency Gain | Core Business Value |
| --- | --- | --- | --- | --- |
| **Health Check** | **12.5 hours**<br>Pure manual CLI checks, cross-system screenshots, manual reporting; time-consuming and prone to fatigue and omissions. | **64 minutes**<br>ChatOps one-click trigger; AAP network-wide concurrency; Agent intelligent cleansing + second-level reporting. | **>91.5%** | Free expert capacity: strip away repetitive L2/L3 labor; consistency: 100% full coverage with zero missed reports, automatic CMDB write-back. |
| **Patch Mgmt** | **13.0 hours**<br>Manual CVE assessment, cross-department long-process ticketing, machine-by-machine patching during maintenance windows. | **50 minutes**<br>Agent matches templates and scans concurrently; one-click confirmation for automatic installation; Satellite synchronization. | **>93.6%** | Extreme response: MTTR compressed from days to immediate blocking; compliance loop: 100% record-to-reality consistency, audit reports in seconds. |
| **RCA & Remediation** | **15.2 hours**<br>Incident scenes easily destroyed; manual performance capture is tedious; highly dependent on vendor experts with long wait times. | **21 minutes**<br>EDA event-driven instantaneous scene retention; LLM precise denoising; empowers L1 with assisted self-healing. | **>97.7%** | 100% on-site demarcation: EDA instantaneously captures stacks; diagnosis decentralization: L1 gains expert-level troubleshooting capabilities. |
| **Capacity Mgmt** | **3.5 hours**<br>Pure manual CLI checks and manual reporting; time-consuming and prone to fatigue and omissions. | **23 minutes**<br>ChatOps one-click trigger; AAP network-wide concurrency; Agent intelligent report generation. | **>90.0%** | Free expert capacity: strip away L2/L3 screenshot-and-form-filling labor; data quality: 100% accuracy + real-time acquisition. |

---

## Business Value Analysis · Health Check

**Business value analysis — intelligent health check**

| Comparison | Traditional Mode | AIOps Mode |
| --- | --- | --- |
| **Total Time** | **750 minutes** (12.5 h) | **64 minutes** |
| **Efficiency Gain** | — | **91.5% ↓** |

### Step-by-Step Comparison

| Step | Manual & Semi-Automated Mode | Time | AIOps Automated Mode · Agentic + Automation | Time |
| --- | --- | --- | --- | --- |
| 1 | Task reception and jump host login preparation (VPN / MFA / SSH Key) | 30 min | Conversation command issuance and workflow triggering (Chat Tool) | 2 min |
| 2 | Basic system resource metric collection (CPU / Memory / Disk / IO) | 120 min | Inspection template query and policy orchestration (MCP for AAP) | 1 min |
| 3 | System core log and kernel anomaly troubleshooting (OOM / packet loss / bad sectors) | 180 min | Automated concurrent execution and data collection (AAP / MCP SSH) | 10 min |
| 4 | Key daemon and Agent status inspection | 90 min | LLM intelligent diagnosis and root cause analysis (with RAG) | 5 min |
| 5 | Network connection and security policy verification | 120 min | CMDB status synchronization and comparison update | 1 min |
| 6 | Inspection evidence retention and data organization | 90 min | Automated inspection report writing and generation | 5 min |
| 7 | Report writing and ticket workflow closure | 120 min | Result pushing and one-click ticket closure (manual review) | 40 min |

### Key ROI Highlights

| Dimension | Benefit |
| --- | --- |
| **Efficiency ROI** | Compressed from 12.5h to 1h, efficiency soared **12x**. |
| **Human Efficiency ROI** | Released **11+ hours** of L2/L3 expert capacity. |
| **Quality ROI** | **100%** zero missed reports, CMDB automatic write-back. |
| **Accessibility ROI** | L1 can obtain expert-level diagnosis, making operations more inclusive. |

---

## Business Value Analysis · Patching

**Business value analysis — vulnerability scanning and patch installation**

| Comparison | Traditional Mode | AIOps Mode |
| --- | --- | --- |
| **Total Time** | **780 minutes** (13.0 h) | **50 minutes** |
| **Efficiency Gain** | — | **93.6% ↓** |

### Step-by-Step Comparison

| Step | Manual & Semi-Automated Mode | Time | AIOps Automated Mode · Agentic + Automation | Time |
| --- | --- | --- | --- | --- |
| 1 | Task reception and login preparation (CVE announcement + bastion host) | 30 min | Command issuance and workflow trigger (Chat Tool) | 2 min |
| 2 | Vulnerability scanning and asset inventory (export vulnerability details) | 120 min | Scan template query and orchestration (MCP for AAP) | 1 min |
| 3 | Vulnerability analysis and impact assessment (remediation plan) | 180 min | Concurrent scanning and direct result output (high-risk asset list) | 15 min |
| 4 | Change request application and cross-departmental approval (CAB) | 120 min | Dialogue confirmation and automatic installation (one-click confirm + verification) | 20 min |
| 5 | Patch installation and status verification (maintenance window batches) | 180 min | Automatic platform status synchronization (Satellite / CMDB) | 2 min |
| 6 | Asset ledger and baseline update (CMDB / Excel) | 60 min | Automatic remediation report generation (RAG enhanced) | 5 min |
| 7 | Report writing and ticket closure | 90 min | Result pushing and closure | 5 min |

### Key ROI Highlights

| Dimension | Benefit |
| --- | --- |
| **Efficiency ROI** | Compressed from 13h to 50min, MTTR from days to immediate blocking. |
| **Human Efficiency ROI** | Released **12+ hours**, converted to zero-trust architecture productivity. |
| **Compliance ROI** | **100%** record-to-reality consistency, audit reports in seconds. |
| **Accessibility ROI** | Dialogue trigger + one-click confirmation, extreme agility. |

---

## Business Value Analysis · RCA & Auto-Remediation

**Business value analysis — incident root cause analysis and assisted self-healing**

| Comparison | Traditional Mode | AIOps Mode |
| --- | --- | --- |
| **Total Time** | **915 minutes** (15.2 h) | **21 minutes** |
| **Efficiency Gain** | — | **97.7% ↓** |

### Step-by-Step Comparison

| Step | Manual & Semi-Automated Mode | Time | AIOps Automated Mode · Agentic + Automation | Time |
| --- | --- | --- | --- | --- |
| 1 | Alarm reception and login troubleshooting (MFA login to target) | 10 min | Anomaly perception and automatic ticket creation (EDA trigger + ITSM) | 1 min |
| 2 | Basic metrics and process localization (top / dmesg) | 15 min | Automated diagnosis and data collection (top / pidstat / perf stack) | 2 min |
| 3 | Deep tracking and call analysis (lsof / strace) | 20 min | Intelligent data cleansing and denoising (threshold + truncation + aggregation) | 1 min |
| 4 | Manual performance data capture (perf record + sosreport) | 30 min | AI intelligent root cause analysis (RAG + LLM) | 3 min |
| 5 | Original manufacturer ticket submission and long wait (vendor L2/L3 analysis) | 720 min | ChatOps interactive consultation and confirmation | 10 min |
| 6 | Cross-team ticket routing and approval | 60 min | Authorization trigger and assisted self-healing (one-click authorization) | 3 min |
| 7 | Manual repair and RCA report writing | 60 min | Knowledge accumulation and full-chain closed loop (RAG new knowledge + automatic ticket closing) | 1 min |

### Key ROI Highlights

| Dimension | Benefit |
| --- | --- |
| **Efficiency ROI** | Compressed from 15h to 21min, recovering downtime losses. |
| **Quality ROI** | **100%** on-site demarcation, curing evidence loss. |
| **Human Efficiency ROI** | Get rid of expert dependence, SRE returns to architecture optimization. |
| **Accessibility ROI** | Cryptic call chains → intuitive suggestions, L1 has RCA capability. |

---

## Business Value Analysis · Capacity Management

**Business value analysis — capacity management**

| Comparison | Traditional Mode | AIOps Mode |
| --- | --- | --- |
| **Total Time** | **200 minutes** (3.3 h) | **23 minutes** |
| **Efficiency Gain** | — | **90.0% ↓** |

### Step-by-Step Comparison

| Step | Manual & Semi-Automated Mode | Time | AIOps Automated Mode · Agentic + Automation | Time |
| --- | --- | --- | --- | --- |
| 1 | Task reception and jump host login preparation | 30 min | Conversation command issuance and workflow triggering (Chat Tool) | 2 min |
| 2 | System storage capacity metric collection and baseline comparison (PV / VG / LV / iNode) | 120 min | Capacity template query and policy orchestration (MCP for AAP) | 1 min |
| 3 | Report writing and high-risk hazard ticket creation | 50 min | Automated concurrent execution and capacity data collection | 15 min |
| — | — | — | Automated report writing and generation (standardized format) | 5 min |
| — | — | — | Result pushing and one-click ticket closed loop (manual review) | 5 min |

### Key ROI Highlights

| Dimension | Benefit |
| --- | --- |
| **Efficiency ROI** | Compressed from 200min to 23min. |
| **Quality ROI** | **100%** data accuracy and timeliness. |
| **Expert ROI** | Offloading L2/L3 screenshot-and-form-filling labor. |
| **Proactive ROI** | Shift from viewing charts to viewing conclusions, proactive governance. |

---

## ROI Summary

| Scenario | Traditional Time | AIOps Time | Efficiency Gain | Related Use Case |
| --- | --- | --- | --- | --- |
| Health Check | 12.5 h | 64 min | >91.5% | [01-04 Use Case 1](01-04-Use-Cases-EN.md) |
| Vuln Scan & Patching | 13.0 h | 50 min | >93.6% | [01-04 Use Case 2](01-04-Use-Cases-EN.md) |
| Incident RCA & Healing | 15.2 h | 21 min | >97.7% | [01-04 Use Case 3](01-04-Use-Cases-EN.md) |
| Capacity Management | 3.5 h | 23 min | >90.0% | Chat Trigger scenario |

> **Note**: Step times are cumulative totals for a **100-unit scale** simulation. Actual benefits vary based on environment complexity and automation maturity.
