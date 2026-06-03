# How to Land AIOps - Landing Methodology & Three-Phase Evolution

> **Status**: 20260603 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Insight

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Scenario-Driven** | AIOps adoption should not start from tools, but from high-value, high-frequency operations scenarios that can form a closed loop. |
| ② | **Trigger Selection** | Different scenarios require different trigger types: low-risk operations fit Chat, periodic tasks fit Schedule, and high-value incident scenarios fit Event. |
| ③ | **Steady Evolution** | Enterprises do not need AI that is “smart but uncontrollable”; they need an intelligent operations system with **controlled risk, visible value, and progressive automation**. |

## AIOps Landing Methodology

**Landing methodology**: six-step approach + three trigger selection patterns.  
**Methodology core**: select implementation paths based on scenarios, so AIOps becomes **affordable, usable, and valuable**.

### Six-Step Landing Method

| Step | Phase | Key Actions | Output / Design Focus |
| --- | --- | --- | --- |
| 1 | **Pain Point Prioritization** | Select the Top 5 pain points and build a priority matrix based on “efficiency × cost.” | Clarify which scenarios should be addressed first and avoid starting with an overly broad scope. |
| 2 | **Scenario → Trigger Mapping** | Map checks / inspections to Chat, capacity forecasting to Schedule, and incidents / RCA to Event. | Choose the most suitable entry point and trigger mechanism for each scenario. |
| 3 | **ChatOps Platform Selection** | Integrate enterprise chat tools such as Feishu, WeCom, Teams, and Slack, and design the permission model. | Build a unified interaction entry point and make natural language the operations interface. |
| 4 | **AI Orchestration Design** | Separate Complex and Simple flows: Complex covers “analysis + prediction + execution,” while Simple covers “check + analysis + reporting.” | Clarify which scenarios can execute actions and which should only provide analysis and reports. |
| 5 | **Trusted Execution Layer** | Build the execution layer with AAP atomic scenarios, parameter standards, Policy Enforcement, and RBAC-based MCP. | Make automated execution controllable, auditable, and reversible. |
| 6 | **Security & Compliance** | Establish Credentials, RBAC, prompt injection protection, and HITL (Human-in-the-Loop) mechanisms. | Keep AI autonomy within enterprise-grade security boundaries. |

### Three Trigger Type Selection Matrix

| Trigger Type | Applicable Scenarios | Typical Tasks | Selection Guidance |
| --- | --- | --- | --- |
| **Chat Trigger** | Low-risk operations triggered by a single sentence. | Health checks, knowledge base queries, configuration standardization, daily reports. | Best for human-initiated, low-risk operations that require immediate feedback. |
| **Schedule Trigger** | Periodic and predictable operations tasks. | Capacity forecasting, FinOps weekly reports, periodic inspections, certificate renewal. | Best for tasks with a fixed execution frequency and predictable results. |
| **Event Trigger** | High-value scenarios driven by EDA in real time. | Incident RCA, black-box forensics, trusted healing, anomaly detection. | Best for alert-driven, time-sensitive scenarios that require fast closed-loop response. |

## Three-Phase Evolution Roadmap

**Evolution path**: steady progress, safety first.  
**Golden rule**: every phase must ensure **controlled risk and visible value**. Enterprises do not need “smartness” alone; they need stability.

| Phase | Time Window | Theme | Build Focus | Key Benefit / Maturity |
| --- | --- | --- | --- | --- |
| **Phase 1** | 0–2 months | **Foundation** | Standardize atomic Job Templates; connect MCP Server to AAP / OBS / K8s / CMDB; initialize the RAG knowledge base with SOPs / cases / Playbooks; integrate ChatOps platform and permission model. | Build the dual foundation of “automation + data”; maturity around **30%**. |
| **Phase 2** | 2–4 months | **Low-Risk Automation** | Health checks; log collection; capacity alerts; automatic incident analysis reports; CMDB auto write-back; configuration drift detection; FinOps weekly report automation. | Release **70%+** of repetitive L1 / L2 workload; maturity around **65%**. |
| **Phase 3** | 4–6 months | **Constrained Auto-Remediation** | EDA event-driven RCA loop; Policy Enforcement constraints; Human-in-the-Loop approval gates; continuous evolution of trusted healing. | Reduce MTTR by **90%+** and fully release expert capacity; maturity around **95%**. |

## Complex Scenario Model A - Known Failure

**Complex scenario diagnosis model A**: a “specialist triage” model for known failure types.  
**Methodology core**: turn “passive treatment” into “proactive triage” and achieve fast closure through predefined failure fingerprints.

### Analogy: Hospital Flow for a Known Disease

| Step | Hospital Flow | Description |
| --- | --- | --- |
| 1 | Identify symptoms | Known medical history or a clear pain point, such as thyroid nodules. |
| 2 | Register with a specialist | Manual registration and precise routing to the thyroid specialist. |
| 3 | Specialist-guided tests | Perform targeted checks such as ultrasound, ECG, and thyroid function blood tests. |
| 4 | Expert consultation | Specialists analyze test reports, clinical experience, and pathology. |
| 5 | Treatment execution | Execute targeted medication or surgery according to medical advice. |

### AIOps Flow for a Known Failure (Example: Process Hung)

| Step | AIOps Flow | Description |
| --- | --- | --- |
| 1 | Identify fingerprint | Identify a specific failure fingerprint, such as Process Hung / D State. |
| 2 | EDA auto-registration | EDA triggers the diagnosis flow within seconds and routes the event into the specialist workflow. |
| 3 | Intelligent triage | Automatically collect CPU stacks, Perf data, and logs, then perform noise reduction. |
| 4 | AI brain analysis | AI Model + RAG + specialized knowledge base jointly analyze the evidence and generate precise RCA. |
| 5 | Trusted healing | Execute atomic remediation actions under Policy Enforcement constraints. |

## Complex Scenario Model B - Unknown Failure

**Complex scenario diagnosis model B**: a “general emergency” model for unknown failure types.  
**Methodology core**: when facing black-box failures, let the AI Agent take over tedious evidence collection and initial correlation analysis.

### Analogy: Hospital Flow for an Unknown Disease

| Step | Hospital Flow | Description |
| --- | --- | --- |
| 1 | Ambiguous symptoms | Only an abnormal symptom is known, such as fever, while the root cause remains unclear. |
| 2 | Register for emergency | Manually go to the emergency department / pre-triage desk. |
| 3 | Full examination | Perform broad baseline checks, such as blood tests, urine tests, CT, and other examinations. |
| 4 | General physician diagnosis | The physician evaluates the whole-body condition and forms an initial diagnosis. |
| 5 | On-site treatment / specialist referral | Treat symptoms on site, or refer the patient to a deeper specialist if needed. |

### AIOps Flow for an Unknown Failure (Example: Unexpected Business Restart)

| Step | AIOps Flow | Description |
| --- | --- | --- |
| 1 | Anomaly sensing | Detect deviation from the system baseline, such as unexpected frequent business restarts. |
| 2 | ChatOps trigger | Operations experts manually initiate diagnosis through a chat window with the AI Agent. |
| 3 | Full health check with SOS | Use MCP to call AAP and execute deep SOS Report collection. |
| 4 | xsos screening + AI | Submit the initial xsos screening results to AI and analyze them with the historical knowledge base. |
| 5 | Human intervention or healing | AI outputs detailed RCA; experts confirm execution, or the process moves into automated healing. |

## Landing Principles Summary

| Principle | Description |
| --- | --- |
| **Chat First, Then Event** | Start with low-risk, human-controlled ChatOps scenarios, then gradually move into event-driven closed loops. |
| **Analyze First, Then Execute** | In the early stage, output diagnostic reports and recommendations first; execute automation only after policy, permissions, and audit controls are ready. |
| **Known First, Then Unknown** | Known failures can be closed quickly through fingerprints and specialist workflows; unknown failures require full evidence collection and AI correlation analysis. |
| **Constrain First, Then Autonomize** | All healing capabilities must first be bounded by Policy, RBAC, HITL, and other security controls before autonomy is expanded. |
