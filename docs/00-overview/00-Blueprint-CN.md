# AIOps DEMO Center Step by step Guide — Blueprint

> **文档性质**：本文件是整套 *AIOps DEMO Center Deploy & Setup Step-by-Step Guide* 的**总纲与规范**，后续所有章节文档均按此执行。  
> **状态**：规划稿 v1.1（结构化优化）  
> **目标仓库**：[AIOps_Enterprise_Practices](https://github.com/adiooooos/AIOps_Enterprise_Practices)

---

## 1. 主题与目标


| 项目           | 说明                                                                                              |
| ------------ | ----------------------------------------------------------------------------------------------- |
| **系列名称（EN）** | AIOps DEMO Center Deploy & Setup Step-by-Step Guide                                             |
| **系列名称（CN）** | AIOps DEMO Center 部署与配置分步指南                                                                     |
| **内容来源**     | 已落地的 `01_AIOPS_DEMO_CENTER部署及配置` 实战环境（RHEL 9 + AAP 2.6 + EDA + Prometheus + n8n + Mattermost 等） |
| **读者**       | 希望复现 GCG AIOps Demo Center 的工程师、售前/架构师、客户 PoC 实施人员                                              |
| **交付形态**     | 可发布到 GitHub 的分章节 Markdown（中英双语 README），配套英文技术资产（playbook / rulebook / yml）                      |


**一句话目标**：把「我已经搭好的 DEMO 环境」沉淀为**可复现、可审计、可同步到 GitHub** 的标准化实施手册，而不是零散笔记。

---

## 2. 背景与范围

### 2.1 已有环境（写作素材）

本指南描述的环境与仓库内目录一一对应，写作时以**真实 IP / 主机名 / 路径**为准（可在各章「环境变量表」中统一列出，便于替换）：


| 逻辑主机                     | 工作区目录                        | 典型 IP         | 角色                                   |
| ------------------------ | ---------------------------- | ------------- | ------------------------------------ |
| `AAP2.6.exmaple.com`     | `01_AAP2.6.exmaple.com/`     | 10.210.65.24  | AAP Controller、EDA Rulebook/Playbook |
| `Mattermost.example.com` | `02_Mattermost.example.com/` | 10.210.65.14  | 分诊台 / ChatOps                        |
| `n8n.example.com`        | `03_n8n.example.com/`        | 10.210.65.103 | 工作流编排                                |
| `prometheus.example.com` | `04_prometheus.example.com/` | 10.210.65.174 | 监控与告警                                |
| `Chaos.example.com`      | `05_Chaos.example.com/`      | 10.210.65.148 | 故障仿真靶机                               |


> 架构总览可参考：`01_AAP2.6.exmaple.com/00-AIOPS架构.md`。


## 3. 编写与发布规范

### 3.1 输出位置

- **本地主目录**：`01_AIOPS_DEMO_CENTER部署及配置/90_AIOps_DEMO_StepbyStep_Guide/`
- **GitHub**：通过 Cursor MCP（GitHub）同步至 [AIOps_Enterprise_Practices](https://github.com/adiooooos/AIOps_Enterprise_Practices)；建议仓库内使用**英文路径**（见 §4.2）

### 3.2 文件命名约定


| 类型                      | 规则                                                                 | 示例                                            |
| ----------------------- | ------------------------------------------------------------------ | --------------------------------------------- |
| **说明文档（README / 章节正文）** | 中文：`*-CN.md`；英文：`*-EN.md`                                          | `04-aap-deploy-CN.md` / `04-aap-deploy-EN.md` |
| **本蓝图**                 | 可保留 `01.The idea of writing a guide.md` 或后续改为 `00-Blueprint-CN.md` | —                                             |
| **技术资产**                | **统一英文**：`.yml`、`.yaml`、rulebook、playbook、`.rego` 等                | `01_eda_rule_linuxperformancealerts_n8n.yml`  |
| **截图**                  | 放在对应章节目录 `images/`，引用用相对路径                                         | `images/aap-ui-credentials.png`               |


> **修正说明**：README 类文档后缀应为 `**.md`**，不是 `.yml`；仅 Ansible/Prometheus 等配置文件使用 `.yml`/`.yaml`。

### 3.3 单章文档建议结构（模板）

每章中英文各一份，结构对齐，便于对照翻译：

```markdown
# 章节标题

## 目标 / Objectives
## 前置条件 / Prerequisites
## 架构关系（在本 DEMO 中的位置）/ Context in DEMO Center
## 分步操作 / Step-by-Step Procedure
## 验证与排错 / Verification & Troubleshooting
## 与上下游章节的衔接 / Next Steps
## 参考 / References
```


## 4. 目录结构（优化后）

### 4.1 逻辑目录（读者视角）


| Part         | 章节    | 说明                                             |
| ------------ | ----- | ---------------------------------------------- |
| **Part 0**   | 0. 总览 | 本蓝图、术语表、环境清单                                   |
| **Part I**   | 1–2   | 方案介绍（Why / What / How / Use Cases / Biz Value） |
| **Part II**  | 3     | DEMO Center 架构与资源规划                            |
| **Part III** | 4–11  | **分步实施**（与部署顺序一致）                              |


### 4.2 建议仓库目录映射（GitHub）

```text
AIOps_Enterprise_Practices/
├── README-EN.md / README-CN.md          # 系列入口
├── docs/
│   ├── 00-overview/
│   ├── 01-solution-introduction/
│   ├── 02-architecture/
│   ├── 03-aap/
│   ├── 04-prometheus/
│   ├── 05-chaos/
│   ├── 06-eda-rulebooks/
│   ├── 07-triage-and-mattermost/
│   ├── 08-n8n/
│   ├── 09-mcp-ssh/
│   └── 10-aiops-agent-workflow/
└── assets/                              # 英文 playbook/rulebook 副本或链接说明
```

本地 `90_AIOps_DEMO_StepbyStep_Guide/` 子目录可与上表**同名对齐**，便于 MCP 同步。

---

## 5. 章节明细与待产出文件

> 下表「建议文件名」为中文稿；英文稿将 `-CN` 换为 `-EN` 即可。

### Part 0 — 总览


| 节   | 主题              | 建议文件名                       | 状态  |
| --- | --------------- | --------------------------- | --- |
| 0.1 | 本系列使用说明、版本、贡献方式 | `00-Guide-How-To-Use-CN.md` | 待写  |


### Part I — AIOps Solution Introduction（概念，可精简）




| 节   | 主题                           | 建议文件名                       | 状态  |
| --- | ---------------------------- | --------------------------- | --- |
| 2.1 | Why AIOps — 传统运维瓶颈 & AI 核心价值 | `02-01-Why-AIOps-CN.md`     | 待写  |
| 2.2 | What is AIOps — 核心能力定义与参考架构  | `02-02-What-Is-AIOps-CN.md` | 待写  |
| 2.3 | How to Land — 落地方法论与三阶段演进    | `02-03-How-To-Land-CN.md`   | 待写  |
| 2.4 | Use Cases — 场景介绍 & DEMO 索引   | `02-04-Use-Cases-CN.md`     | 待写  |
| 2.5 | Biz Value — 价值主张 & ROI 分析    | `02-05-Biz-Value-CN.md`     | 待写  |


### Part II — Architecture Design


| 节   | 主题                    | 建议文件名                                     | 素材来源            |
| --- | --------------------- | ----------------------------------------- | --------------- |
| 3.1 | AIOps 组件与环境规划         | `03-01-Components-And-Environments-CN.md` | `00-AIOPS架构.md` |
| 3.2 | 资源规划（CPU/内存/磁盘/网络/端口） | `03-02-Resources-Planning-CN.md`          | 待补充清单           |


### Part III — 分步实施（核心）

#### 4. Ansible Automation Platform（AAP）


| 节   | 主题                                | 建议文件名                              | 素材来源            |
| --- | --------------------------------- | ---------------------------------- | --------------- |
| 4.1 | AAP 部署先决条件                        | `04-01-AAP-Prerequisites-CN.md`    | 待从实战提炼          |
| 4.2 | AAP 部署前准备                         | `04-02-AAP-Preparation-CN.md`      | 待写              |
| 4.3 | AAP Server 部署                     | `04-03-AAP-Server-Deploy-CN.md`    | 待写              |
| 4.4 | AAP 配置（Inventory、EE、Project 等）    | `04-04-AAP-Configuration-CN.md`    | 待写              |
| 4.5 | 为 AAP 配置 MCP Server → AI Agent 连接 | `04-05-AAP-MCP-For-Agent-CN.md`    | MCP / Cursor 实践 |
| 4.6 | Automation Decisions 配置           | `04-06-Automation-Decisions-CN.md` | Policy / AD 相关  |


#### 5. Prometheus 监控


| 节   | 主题           | 建议文件名                            | 素材来源                         |
| --- | ------------ | -------------------------------- | ---------------------------- |
| 5.1 | 软件包下载及解压     | `05-01-Prometheus-Package-CN.md` | `04_prometheus.example.com/` |
| 5.2 | 标准化配置部署      | `05-02-Prometheus-Config-CN.md`  | `prometheus.yml` 等           |
| 5.3 | 规则文件部署       | `05-03-Prometheus-Rules-CN.md`   | `10-Prometheus规则文件/`         |
| 5.4 | systemd 服务配置 | `05-04-Prometheus-Systemd-CN.md` | 待写                           |
| 5.5 | 启动和验证服务      | `05-05-Prometheus-Verify-CN.md`  | `README.md`                  |


#### 6. Chaos Server（故障仿真靶机）


| 节   | 主题                        | 建议文件名                              | 说明                            |
| --- | ------------------------- | ---------------------------------- | ----------------------------- |
| 6.1 | Use Case 1：Process Hung   | `06-01-Chaos-Process-Hung-CN.md`   | 对齐 EDA / 性能告警场景               |
| 6.2 | Use Case 2：Network Error  | `06-02-Chaos-Network-Error-CN.md`  | 对齐 `linux_network_alerts`     |
| 6.3 | Use Case 3：Service Outage | `06-03-Chaos-Service-Outage-CN.md` | 原稿「故障访问」应为 **Service Outage** |


#### 7. EDA Rulebook


| 节   | 主题                      | 建议文件名                           | 素材来源                                       |
| --- | ----------------------- | ------------------------------- | ------------------------------------------ |
| 7.1 | Process Hung 相关 EDA 配置  | `07-01-EDA-Process-Hung-CN.md`  | `01_eda_rule_linuxperformancealerts_*.yml` |
| 7.2 | Network Error 相关 EDA 配置 | `07-02-EDA-Network-Error-CN.md` | `02_eda_rule_linux_network_down.yml`       |


#### 8. Triage Server（分诊台）


| 节   | 主题               | 建议文件名                            | 素材来源                         |
| --- | ---------------- | -------------------------------- | ---------------------------- |
| 8.1 | 诊断数据配置           | `08-01-Triage-Diagnostics-CN.md` | 待写                           |
| 8.2 | Mattermost 安装及配置 | `08-02-Mattermost-Setup-CN.md`   | `02_Mattermost.example.com/` |


#### 9. n8n Server


| 节   | 主题        | 建议文件名                             | 素材来源                  |
| --- | --------- | --------------------------------- | --------------------- |
| 9.1 | 软件包准备     | `09-01-n8n-Package-CN.md`         | `03_n8n.example.com/` |
| 9.2 | n8n 工作流导入 | `09-02-n8n-Workflow-Import-CN.md` | 待写                    |


#### 10. MCP for SSH


| 节    | 主题                | 建议文件名                       | 素材来源                                       |
| ---- | ----------------- | --------------------------- | ------------------------------------------ |
| 10.1 | MCP for SSH 部署及配置 | `10-01-MCP-SSH-Setup-CN.md` | `02_Mattermost.example.com/mcp_for_ssh.md` |


#### 11. AIOps Agent Workflow 联调


| 节    | 主题                             | 建议文件名                            | 说明                                 |
| ---- | ------------------------------ | -------------------------------- | ---------------------------------- |
| 11.1 | AIOPS Agent n8n Workflow 配置及测试 | `11-01-Agent-Workflow-E2E-CN.md` | 端到端：告警 → EDA → n8n → Chat →（可选）AAP |




## 附录 A：原目录清单对照（快速索引）

原稿中的章节序号已完整收纳至 §5，主要优化点：

- 统一 Part 0 / I / II / III 分层  
- 补充 **0 总览**、**验收标准**、**GitHub 目录映射**  
- 明确 **README 用 .md、配置用英文 .yml**  
- **6.3** 更正为 Service Outage  
- **2.5** 更正为 Biz Value

