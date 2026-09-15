# Automation Decisions（EDA）配置

> **状态**：20260914 Updated  
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

```bash
# 用户：root · 节点：AAP 本机
export AAP_SSH_HOST="10.210.65.46"   # 改成你的 AAP 节点 IP 或 FQDN
```

### 1.3 一键安装 Git、建用户、裸库与 SSH 授权

```bash
# 用户：root · 节点：AAP 本机
# 说明：创建 git 系统用户并授权 root 公钥；裸库归 git:git 所有

dnf install -y git

mkdir -p /var/lib/git
useradd -r -m -d /var/lib/git -s /bin/bash git 2>/dev/null || true

git init --bare /var/lib/git/eda-project.git
chown -R git:git /var/lib/git/eda-project.git

mkdir -p /var/lib/git/.ssh
touch /var/lib/git/.ssh/authorized_keys
grep -qxF "$(cat /root/.ssh/id_rsa.pub)" /var/lib/git/.ssh/authorized_keys 2>/dev/null \
  || cat /root/.ssh/id_rsa.pub >> /var/lib/git/.ssh/authorized_keys
chown -R git:git /var/lib/git/.ssh
chmod 700 /var/lib/git/.ssh
chmod 600 /var/lib/git/.ssh/authorized_keys

chown git:git /var/lib/git
chmod 750 /var/lib/git

firewall-cmd --permanent --add-service=ssh 2>/dev/null; firewall-cmd --reload 2>/dev/null || true
```

> SSH / SCM 连通性验证见 **5.0 Git SCM 连通性**。

### 1.4 工作区 + 首次推送（main）

```bash
# 用户：root · 节点：AAP 本机
# 工作区：/var/lib/git/work/eda-project · 裸库：/var/lib/git/eda-project.git

mkdir -p /var/lib/git/work
rm -rf /var/lib/git/work/eda-project
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git clone "git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git" /var/lib/git/work/eda-project

cd /var/lib/git/work/eda-project
git checkout -b main 2>/dev/null || git branch -M main

# ① 先把已有 rulebook 资产放入 rulebooks/（内容见第 3 章），例如：
#    01_eda_rule_linuxperformancealerts_ui.yml · TEST_01.yml · TEST_WEBHOOK.yml
mkdir -p /var/lib/git/work/eda-project/rulebooks
cp /path/to/your/*.yml /var/lib/git/work/eda-project/rulebooks/   # 按实际路径修改
printf '%s\n' '# rulebook YAML 目录' > /var/lib/git/work/eda-project/rulebooks/README.txt
ls -la /var/lib/git/work/eda-project/rulebooks/

# ② 若远端已有 main（重跑本步骤），对齐远端后再追加 commit，勿重复 init
git fetch origin 2>/dev/null || true
if git rev-parse --verify origin/main >/dev/null 2>&1; then
  git checkout main && git reset --hard origin/main
fi

git -C /var/lib/git/work/eda-project add rulebooks/
git -C /var/lib/git/work/eda-project config user.email "root@aap27.example.com"
git -C /var/lib/git/work/eda-project config user.name "aap27 lab"
git -C /var/lib/git/work/eda-project commit -m "init: rulebooks/"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git -C /var/lib/git/work/eda-project push -u origin main
```

> **提示**：首次 push 应包含第 3 章全部 rulebook YAML，不要只提交 `README.txt`。若 `git commit` 提示 *nothing to commit*，说明资产已在远端，直接进入第 2 章增量提交。

---

## 2. 后续提交 Rulebook 到 Git

一次完整提交流程（可直接复制）：

```bash
# 用户：root · 工作区：/var/lib/git/work/eda-project

git -C /var/lib/git/work/eda-project branch --show-current
git -C /var/lib/git/work/eda-project checkout main
git -C /var/lib/git/work/eda-project status

git -C /var/lib/git/work/eda-project add /var/lib/git/work/eda-project/rulebooks/rulebook_test.yml
git -C /var/lib/git/work/eda-project commit -m "add: rulebook_test.yml for EDA"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git -C /var/lib/git/work/eda-project push origin main
```

若当前已在 `/var/lib/git/work/eda-project/rulebooks`：

```bash
# 用户：root
git -C /var/lib/git/work/eda-project add /var/lib/git/work/eda-project/rulebooks/rulebook_test.yml
git -C /var/lib/git/work/eda-project commit -m "add: rulebook_test.yml for EDA"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git -C /var/lib/git/work/eda-project push origin main
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

路径：**Automation Decisions → Infrastructure → Credentials → Create / Edit**

| 字段 | 填写值 | 说明 |
| --- | --- | --- |
| **Name** | `git` | 任意可识别名称；后续 EDA Project 会引用此凭据 |
| **Organization** | `Default` | 与 EDA Project 同一 Organization |
| **Credential type** | `Source Control` | Git SCM 专用类型 |
| **Username** | **`git`** | SSH URL 中的登录用户（`git@host:...`），**不是** `root` |
| **Password** | *留空* | 本 DEMO 使用 SSH 密钥认证，无需密码 |
| **SCM Private Key** | `/root/.ssh/id_rsa` 全文 | 与第 1 章手工 `git push` 使用的是**同一把私钥**；公钥已写入 `git` 用户的 `authorized_keys` |

获取私钥内容（在 AAP 节点 root shell 执行）：

```bash
cat /root/.ssh/id_rsa
```

将输出整段粘贴到 **SCM Private Key**（含 `-----BEGIN ... PRIVATE KEY-----` 与 `-----END ... PRIVATE KEY-----`），或 **Browse...** 上传该文件。若界面显示 `$encrypted$`，表示已保存过密钥；需更换时先 **Clear** 再重新粘贴。

确认公钥与私钥配对（可选）：

```bash
grep -q "$(ssh-keygen -y -f /root/.ssh/id_rsa)" /var/lib/git/.ssh/authorized_keys \
  && echo "key pair OK"
```

> **要点**：命令行里用 `root` 执行 `git push`，是因为 root 持有私钥；AAP/EDA 通过 Credential 里的 **`git` 用户名 + 同一把私钥** 以 `git@` 身份访问裸库，与第 1.4 节 `ssh git@host` 自测逻辑一致。


### 4.2 EDA Project

配置 **Automation Decisions** 的 EDA Project（SCM URL、Branch `main`、子目录 `rulebooks/` 等）。

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/018493e6-2941-424e-affc-0ca8c98ae5d6" />

路径：**Automation Decisions → Projects → Create / Edit**

#### Source control URL 填什么

本 DEMO 使用 **SSH 协议** 指向本机裸库（与第 1.5 节 `git clone` 地址相同）：

```text
git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git
```

| 环境 | 示例（将 `${AAP_SSH_HOST}` 换成你的 AAP 节点 IP 或 FQDN） |
| --- | --- |
| 文档示例 | `git@10.210.65.24:/var/lib/git/eda-project.git` |
| AAP 2.7 lab | `git@10.210.65.46:/var/lib/git/eda-project.git` |

**不要**写成 HTTP(S)（如 `https://...`），除非另行部署了 Git HTTP 前端；本指南裸库仅开放 SSH。

#### 如何测试 URL 是否正确

在 AAP 节点 root shell 执行（`AAP_SSH_HOST` 与 UI 中 URL 的 host 部分必须一致）：

```bash
export AAP_SSH_HOST="10.210.65.46"   # 改成你的 IP

# 1) SSH 连通性（对应 4.1 凭据：git 用户 + id_rsa）
ssh-keyscan -H "${AAP_SSH_HOST}" >> /root/.ssh/known_hosts 2>/dev/null
ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes "git@${AAP_SSH_HOST}" exit

# 2) 远端能否读到 main 分支（等同 EDA Sync 前的 SCM 探测）
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git ls-remote "git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git" refs/heads/main

# 3) 完整 clone 自测（可选）
rm -rf /tmp/eda-project-sync-test
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git clone "git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git" /tmp/eda-project-sync-test
ls /tmp/eda-project-sync-test/rulebooks/
```

| 命令 | 通过标准 |
| --- | --- |
| `ssh git@host exit` | 退出码 0（或 Git shell 拒绝交互 shell 但公钥认证成功） |
| `git ls-remote ... refs/heads/main` | 输出一行 commit hash + `refs/heads/main` |
| `ls .../rulebooks/` | 可见 `README.txt` 或已提交的 `*.yml` |

以上 CLI 测试通过后，UI 里 **Source control URL** 填同一 SSH 地址即可。

#### EDA Project 字段一览

| 字段 | 填写值 |
| --- | --- |
| **Name** | `eda-project`（或自定义） |
| **Organization** | `Default` |
| **Source control URL** | `git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git` |
| **Source control credential** | 第 4.1 节创建的 `git` 凭据 |
| **Source control branch** | `main` |
| **Source control ref** | *留空*（使用 branch 即可） |
| **Rulebook directory** / **SCM sub-directory** | `rulebooks/` |

保存后点击 **Sync**（或 **Sync project**）。Sync 成功且 Project 详情中可见 rulebook 列表，即表示 URL、凭据、分支与子目录均正确。




### 4.3 Event Stream Credentials

配置用于认证 **EDA Event Stream** 的 Credentials；**Username / Password 必须与 Prometheus Alertmanager 的 `basic_auth` 严格一致**，否则 webhook 会返回 401，告警无法进入 EDA。
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/a328df6e-457b-4f55-a489-9be998604804" />



路径：**Automation Decisions → Infrastructure → Credentials → Create / Edit**

| 字段 | DEMO 填写值 | 说明 |
| --- | --- | --- |
| **Name** | `event-stream` | 创建 Event Stream 时引用此凭据 |
| **Organization** | `Default` | 与 Event Stream 同一 Organization |
| **Credential type** | `Basic Event Stream` | 非 Source Control |
| **Username** | `event_stream` | 与 Alertmanager `basic_auth.username` **完全一致** |
| **Password** | `redhat` | 与 Alertmanager `basic_auth.password` **完全一致** |

#### 与 Alertmanager 对齐

Alertmanager 侧（`/opt/alertmanager/alertmanager.yml` 或等价路径）webhook 示例：

```yaml
receivers:
  - name: 'EDA'
    webhook_configs:
    - url: 'https://aap26.example.com:443/eda-event-streams/api/eda/v1/external_event_stream/<UUID>/post/'
      http_config:
        basic_auth:
          username: "event_stream"
          password: "redhat"
        tls_config:
          insecure_skip_verify: true
```

| 配置位置 | 必须一致的项 |
| --- | --- |
| **AAP Credentials（本节）** | Username = `event_stream`，Password = `redhat` |
| **Alertmanager `basic_auth`** | `username` / `password` 同上 |
| **Event Stream POST URL** | 填入 Alertmanager `webhook_configs.url`（见 **4.5**） |
| **手动测试 `curl -u`** | `-u 'event_stream:redhat'`（见 **5.3**） |

> 修改任一侧的用户名或密码后，**AAP Credentials 与 Alertmanager 必须同步更新**，并 reload Alertmanager（`curl -X POST http://localhost:9093/-/reload` 或重启服务）。


### 4.4 Rulebook Credentials

> UI 英文标题：**Config Credentials for rulebook action which need to connect to Automation Controller**

配置 **Rulebook 中 action 触发时、EDA 连接 Automation Controller 所需的 Credentials**——**不是** Rulebook YAML 文件本身的认证，也**不是** Event Stream 入站 webhook 认证（后者见 **4.3**）。

| 对比 | 用途 |
| --- | --- |
| **4.1 Git 凭据** | EDA Project 从 Git 拉取 rulebook |
| **4.3 Event Stream 凭据** | Alertmanager / 外部系统 POST 事件到 Event Stream |
| **4.4 本节凭据** | Rulebook **action** 执行时调用 **Automation Controller**（如 `run_job_template`） |

当 Rulebook 使用 `run_job_template` 等需要 Controller 的 action 时（见第 3 章 AIOps 用例），EDA 须凭此 Credential 向 Controller 发起 API 请求（启动 Job Template）。未配置或配置错误时，Activation 可能收到事件但 **Job 无法启动**。

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/3c5c16f4-1cd7-4f5e-91a6-dad0701f33d2" />

路径：**Automation Decisions → Infrastructure → Credentials → Create / Edit**（类型为连接 Automation Controller 的凭据；创建 **Rulebook Activation** 时在 Controller 相关字段引用，见 **5.2**）。

| 要点 | 说明 |
| --- | --- |
| **何时需要** | Rulebook 含 `run_job_template`（或同类 Controller action） |
| **与 Job Template 关系** | 凭据负责 **EDA → Controller 鉴权**；Job Template 名称 / Organization 在 rulebook YAML 中指定，且须在 Controller 中 **预先创建** |
| **DEMO 示例** | Rulebook 调用 `01_EDA_action_linuxperformancealerts_UI` 等模板前，确认本节凭据已在 Activation 中关联 |


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

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/7dc9e40f-80b5-468b-b52d-7be3b39637d1" />



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
