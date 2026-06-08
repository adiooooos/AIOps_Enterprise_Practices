# Automation Decisions（EDA）配置

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **Git → EDA Project** | AIOps DEMO 在 AAP 本机建裸库 `/var/lib/git/eda-project.git`，Rulebook 放在 `rulebooks/` 子目录，经 EDA Project Sync 进入 Automation Decisions。 |
| ② | **Event Stream 驱动** | Prometheus / Alertmanager 告警经 **EDA Event Stream** URL 注入；Rulebook Activation 匹配 `event.payload` 后 `run_job_template` 触发 Controller Job。 |
| ③ | **UI 版 Rulebook 设计** | 生产/演示 Rulebook **不使用** `run_module` / `run_playbook`（UI 下易报 inventory 错误）；副作用全部放到 Job Template Playbook 中。 |

## 本章目标

在 AAP 2.6 上完成 **Automation Decisions（EDA）** 端到端配置——本地 Git 裸库、Rulebook 提交推送、Gateway UI 中 EDA Project / Credentials / Event Stream / Rulebook Activation，并验证 Event Stream 与 AIOps 用例 Rulebook。

| 项目 | 说明 |
| --- | --- |
| **适用版本** | AAP **2.6**（Container Growth · 含 EDA） |
| **示例节点** | `aap26.example.com` · `10.210.65.24` |
| **裸库路径** | `/var/lib/git/eda-project.git` |
| **工作区** | `/var/lib/git/work/eda-project` |
| **Rulebook 子目录** | `rulebooks/`（EDA Project 填 Branch + 子目录） |
| **适用范围** | 非生产 / 演示环境 |

---

## Executive Summary: 配置步骤一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **本地 Git 裸库** | 建 `git` 用户、裸库、root→git SSH | ✅ |
| 2 | **初始化 rulebooks/** | clone 工作区、首次 commit 推送 `main` | ✅ |
| 3 | **Gateway UI** | EDA Credentials、Project、Event Stream | ✅ |
| 4 | **Rulebook Activation** | Sync Project → 创建 Activation | ✅ |
| 5 | **Event Stream 测试** | `curl -k -X POST` + Basic Auth | ✅ |
| 6 | **AIOps 用例 Rulebook** | 性能 / 网络告警 → Job Template | 建议 |

---

## 1. 在 AAP 2.6 配置本地 Git 仓库

> **适用范围**：非生产 / 演示。裸库路径固定为 `/var/lib/git/eda-project.git`；SSH 仅使用 root 的 `/root/.ssh/id_rsa` 与 `id_rsa.pub`（公钥写入 `git` 用户，私钥粘贴进 AAP 凭据）。

### 1.1 重要说明

| 要点 | 说明 |
| --- | --- |
| **裸库** | `/var/lib/git/eda-project.git` 是 Git 内部对象存储，**不要**直接手写 rulebook 文件进去 |
| **Rulebook 位置** | 在仓库版本内容里用子目录 `rulebooks/*.yml`；EDA Project 填 **Branch** + 子目录 `rulebooks/` |
| **本机编辑** | 在 `/var/lib/git/` 下单独建工作区（与裸库并列），例如 `/var/lib/git/work/eda-project` |

### 1.2 定义变量

在 root shell 执行：

```bash
export AAP_SSH_HOST="10.210.65.24"
```

### 1.3 一键安装 Git、建用户、裸库与 SSH 授权

```bash
dnf install -y git

mkdir -p /var/lib/git
useradd -r -m -d /var/lib/git -s /bin/bash git 2>/dev/null || true

git init --bare /var/lib/git/eda-project.git
chown -R git:git /var/lib/git/eda-project.git

mkdir -p /var/lib/git/.ssh
touch /var/lib/git/.ssh/authorized_keys
grep -qxF "$(cat /root/.ssh/id_rsa.pub)" /var/lib/git/.ssh/authorized_keys 2>/dev/null || cat /root/.ssh/id_rsa.pub >> /var/lib/git/.ssh/authorized_keys
chown -R git:git /var/lib/git/.ssh
chmod 700 /var/lib/git/.ssh
chmod 600 /var/lib/git/.ssh/authorized_keys

chown git:git /var/lib/git
chmod 750 /var/lib/git

firewall-cmd --permanent --add-service=ssh 2>/dev/null; firewall-cmd --reload 2>/dev/null || true
```

### 1.4 本机自测 SSH

root 使用自己的私钥连接 `git@本机`：

```bash
ssh-keyscan -H "${AAP_SSH_HOST}" >> /root/.ssh/known_hosts 2>/dev/null
ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes "git@${AAP_SSH_HOST}"
```

期望：能显示 `git` 用户的非交互 shell 提示或 Git 相关输出（公钥匹配）。若策略禁止 shell，以 AAP Project Sync 成功为准。

### 1.5 工作区 + rulebooks 子目录 + 首次推送（main）

```bash
mkdir -p /var/lib/git/work
cd /var/lib/git/work
rm -rf eda-project
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git clone "git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git" eda-project
cd /var/lib/git/work/eda-project

git checkout -b main 2>/dev/null || git branch -M main
mkdir -p rulebooks
printf '%s\n' '# 将你的 rulebook YAML 放在此目录' > rulebooks/README.txt
git add rulebooks
git config user.email "root@aap26.example.com"
git config user.name "aap26 lab"
git commit -m "init: rulebooks/"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git push -u origin main
```

---

## 2. 后续提交 Rulebook 到 Git

一次完整提交流程（可直接复制）：

```bash
cd /var/lib/git/work/eda-project

# 1) 确认当前分支是 main（与 EDA Project 配置一致）
git branch --show-current
git checkout main

# 2) 查看变更
git status

# 3) 提交到本地仓库（从仓库根目录执行）
git add rulebooks/rulebook_test.yml
git commit -m "add: rulebook_test.yml for EDA"

# 4) 推送到远端裸库
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git push origin main
```

若当前在 `/var/lib/git/work/eda-project/rulebooks` 目录下：

```bash
git add rulebook_test.yml
git commit -m "add: rulebook_test.yml for EDA"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git push origin main
```

推送后在 Gateway UI **Sync EDA Project**，再创建或更新 Rulebook Activation。

---

## 3. Rulebook 示例

### 3.1 测试 / Debug 用 Rulebook

Git 提交后，在 UI 中配置相应 **Rulebook Activations**，用于 Event Source 消息接收测试。

**`rulebooks/TEST_01.yml`**

```yaml
---
- name: "Linux Performance Alerts Remediation"
  hosts: all

  sources:
    - name: "Listen for Prometheus Linux Performance Alerts (Alertmanager webhook)"
      ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  execution_strategy: sequential

  rules:
    - name: "Linux perf alerts — AAP Event Stream payload (event.payload)"
      condition: >
        event.payload is defined or
        event.payload.status == "firing"
      actions:
        - debug:
            msg: "Received event payload: {{ event.payload }}"

        - print_event:
            pretty: true
```

**`rulebooks/TEST_WEBHOOK.yml`**

```yaml
---
- name: "Event Stream Debugger"
  hosts: all
  sources:
    - ansible.eda.webhook:
        # 在 EDA UI 中关联 Event Stream 时，这里的配置通常会被 UI 设置覆盖
        port: 5000
  rules:
    - name: "Log full event payload"
      condition: true  # 强制匹配所有进入该通道的事件
      action:
        debug:
```

### 3.2 AIOps 用例：Complex Issues RCA

**`rulebooks/01_eda_rule_linuxperformancealerts_ui.yml`**

```yaml
---
# =============================================================================
# Rulebook (AAP UI / Event Stream 版本)
# -----------------------------------------------------------------------------
# 用途：在 AAP 2.6 UI 下作为 Rulebook Activation 运行；接收来自 AAP
#       Event Stream 转发过来的 Prometheus Alertmanager Webhook Payload。
#
# 关键改造点（相对 CMD 版本 01_eda_rule_linuxperformancealerts_n8n.yml）：
#   1. 不再使用 run_module / run_playbook，避免 AAP UI 下
#      "action which needs inventory to be defined" 错误。
#   2. 全部副作用动作（写 alert 文件、调用 n8n webhook、采集性能数据）
#      都迁移到 job_template 对应的 Playbook 中执行。
#   3. condition 仅匹配 HighSystemCpuUsage / CriticalSystemCpuUsage 两类告警。
#   4. 通过 job_args.extra_vars.webhook_payload 把整份 event.payload
#      原样透传给被触发的 Job Template。
# =============================================================================

- name: "Linux Performance Alerts (AAP UI / Event Stream)"
  hosts: all

  sources:
    - name: "AAP Event Stream webhook listener"
      ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  execution_strategy: sequential

  rules:
    - name: "Linux perf alerts — match HighSystemCpuUsage or CriticalSystemCpuUsage"
      condition: >-
        event.payload.groupLabels.alertname is defined and
        (event.payload.groupLabels.alertname == "HighSystemCpuUsage" or
         event.payload.groupLabels.alertname == "CriticalSystemCpuUsage")
      actions:
        - debug:
            msg: |
              ✅ Linux Performance Alert received from AAP Event Stream
              -------------------------------------------------------------
              Alert Name     : {{ event.payload.groupLabels.alertname }}
              Instance       : {{ event.payload.commonLabels.instance }}
              Instance Name  : {{ event.payload.commonLabels.instance_name }}
              Severity       : {{ event.payload.commonLabels.severity }}
              Status         : {{ event.payload.status }}
              Receiver       : {{ event.payload.receiver }}
              -------------------------------------------------------------
              FULL EVENT PAYLOAD:
              {{ event.payload }}

        - run_job_template:
            name: "01_EDA_action_linuxperformancealerts_UI"
            organization: "Default"
            job_args:
              limit: "{{ event.payload.commonLabels.instance_name }}"
              extra_vars:
                webhook_payload: "{{ event.payload }}"
```

> Job Template `01_EDA_action_linuxperformancealerts_UI` 须在 UI 中预先创建，Playbook 选用 `01_EDA_action_linuxperformancealerts_UI.yml`，并指定 Inventory 与 Project。

**`rulebooks/02_eda_rule_linux_network_down.yml`**

```yaml
---
# =============================================================================
# Rulebook (AAP UI / Event Stream 版本)
# -----------------------------------------------------------------------------
# 用途：匹配「网络接口异常」类告警，调用 Job Template 启动修复 Playbook。
# 设计点同 01_eda_rule_linuxperformancealerts_ui.yml。
# =============================================================================

- name: "Linux Network Interface Down Alerts (AAP UI / Event Stream)"
  hosts: all

  sources:
    - name: "AAP Event Stream webhook listener"
      ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000

  execution_strategy: sequential

  rules:
    - name: "Linux network alerts — match NetworkInterfaceDown"
      condition: >-
        event.payload.groupLabels.alertname is defined and
        event.payload.groupLabels.alertname == "NetworkInterfaceDown"
      actions:
        - debug:
            msg: |
              Linux Network Interface Down Alert received from AAP Event Stream
              -------------------------------------------------------------
              Alert Name     : {{ event.payload.groupLabels.alertname }}
              Instance       : {{ event.payload.commonLabels.instance }}
              Instance Name  : {{ event.payload.commonLabels.instance_name }}
              Device         : {{ event.payload.commonLabels.device }}
              Severity       : {{ event.payload.commonLabels.severity }}
              Status         : {{ event.payload.status }}
              Receiver       : {{ event.payload.receiver }}
              -------------------------------------------------------------
              FULL EVENT PAYLOAD:
              {{ event.payload }}

        - run_job_template:
            name: "02_EDA_action_network_down_UI"
            organization: "Default"
            job_args:
              limit: "{{ event.payload.commonLabels.instance_name }}"
              extra_vars:
                webhook_payload: "{{ event.payload }}"
```

---

## 4. 在 AAP UI 上配置 EDA

### 4.1 Git 认证 Credentials

配置 **Automation Decisions** 用于 Git 认证的 Credentials。
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/8d41bfc4-4142-42e2-a65b-976a05d6a6c2" />




### 4.2 EDA Project

配置 **Automation Decisions** 的 EDA Project（SCM URL、Branch `main`、子目录 `rulebooks/` 等）。

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/018493e6-2941-424e-affc-0ca8c98ae5d6" />



### 4.3 Event Stream Credentials

配置用于认证 **EDA Event Stream** 的 Credentials。
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/d14061c0-3609-42a9-b3f8-931cbc5364a7" />




### 4.4 Rulebook Credentials

配置用于认证 **EDA Rulebook** 的 Credentials。

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/3c5c16f4-1cd7-4f5e-91a6-dad0701f33d2" />



### 4.5 EDA Event Stream

配置 EDA Event Stream。

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/8b482df1-8de9-488f-af8e-c06a0679b186" />



记录 Event Stream 的 **POST URL**（后续 Rulebook / 测试需用到）。DEMO 示例：

```
https://aap26.example.com:443/eda-event-streams/api/eda/v1/external_event_stream/a8e18ec4-ede4-43be-a3d9-dbc83701b811/post/
```

> UUID（`a8e18ec4-...`）以 UI 创建后实际值为准。

---

## 5. 测试 Event Stream 与 Rulebook

### 5.1 撰写并提交测试 Rulebook

工作目录示例：

```bash
cd /var/lib/git/work/eda-project/rulebooks
```

**`rulebook_test.yml`**

```yaml
---
- name: "Event Stream Debugger"
  hosts: all
  sources:
    - ansible.eda.webhook:
        # 在 EDA UI 中关联 Event Stream 时，这里的配置通常会被 UI 设置覆盖
        port: 5000
  rules:
    - name: "Log full event payload"
      condition: true  # 强制匹配所有进入该通道的事件
      action:
        debug:
```

提交并推送：

```bash
cd /var/lib/git/work/eda-project
git branch --show-current
git checkout main
git status
git add rulebooks/rulebook_test.yml
git commit -m "add: rulebook_test.yml for EDA"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git push origin main
```

### 5.2 配置 Rulebook Activation

1. 在 UI **Sync EDA Project**
2. 配置 **Rulebook Activation**（关联 Event Stream、Rulebook 文件等）

<img width="1016" height="488" alt="image" src="https://github.com/user-attachments/assets/45e9f0ec-f2b6-4a5f-875f-f7b16a224a1c" />


### 5.3 命令行测试 Event Stream

在 Server 上执行：

```bash
curl -k -X POST \
  'https://aap26.example.com:443/eda-event-streams/api/eda/v1/external_event_stream/a8e18ec4-ede4-43be-a3d9-dbc83701b811/post/' \
  -u 'event_stream:redhat' \
  -H 'Content-Type: application/json' \
  -d '{"message": "Hello from manual header"}'
```

| 参数 | 说明 |
| --- | --- |
| **URL** | Event Stream 详情页中的 POST URL |
| **`-u event_stream:redhat`** | EDA Credentials 中 Event Stream 的用户名与密码（DEMO 示例） |

### 5.4 UI 确认结果

在 Gateway UI 查看 Rulebook Activation 的 Event 详情，确认 payload 已被 Rulebook 接收并执行。

<img width="751" height="484" alt="image" src="https://github.com/user-attachments/assets/a3c1ba7a-e2b9-48ce-9e5e-34d91b6a6f3d" />

<img width="747" height="481" alt="image" src="https://github.com/user-attachments/assets/d9e4d9ea-d7a3-4551-9be5-9a84102b52e5" />


---

## 配置后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | Git 裸库 | `/var/lib/git/eda-project.git` 存在，`git@host` SSH 可用 |
| 2 | rulebooks/ | `main` 分支含 `rulebooks/*.yml` |
| 3 | EDA Project | Sync 成功，可见 Rulebook 列表 |
| 4 | Event Stream | POST URL 已记录；`curl` 测试返回成功 |
| 5 | Rulebook Activation | 状态 Running；测试事件在 UI 可见 |
| 6 | AIOps 用例 | 性能 / 网络告警 Rulebook 可触发对应 Job Template |

---

## 参考文档

- [03-04 AAP 配置](03-04-AAP-Configuration-CN.md)
- [03-05 AAP MCP For Agent](03-05-AAP-MCP-For-Agent-CN.md)
- [Event-Driven Ansible in AAP 2.6](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/event-driven_ansible)

> **说明**：本章为 AAP 系列（03-aap）最后一章；后续可继续配置 Prometheus Event Stream 源、n8n 联动等（见 DEMO 其他分册）。
