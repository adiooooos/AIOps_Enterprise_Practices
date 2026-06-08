# Automation Decisions (EDA) Configuration

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Git → EDA Project** | AIOps DEMO uses a local bare repo `/var/lib/git/eda-project.git` on the AAP host; rulebooks live under `rulebooks/` and sync via EDA Project. |
| ② | **Event Stream driven** | Prometheus / Alertmanager alerts post to an **EDA Event Stream** URL; Rulebook Activation matches `event.payload` and triggers Controller jobs via `run_job_template`. |
| ③ | **UI rulebook pattern** | Production/demo rulebooks **avoid** `run_module` / `run_playbook` (inventory errors in UI); side effects go in Job Template playbooks. |

## Chapter Objective

Complete end-to-end **Automation Decisions (EDA)** setup on AAP 2.6 — local Git bare repo, rulebook commit/push, Gateway UI EDA Project / Credentials / Event Stream / Rulebook Activation, and verify Event Stream plus AIOps use-case rulebooks.

| Item | Description |
| --- | --- |
| **Version** | AAP **2.6** (Container Growth · includes EDA) |
| **Example node** | `aap26.example.com` · `10.210.65.24` |
| **Bare repo** | `/var/lib/git/eda-project.git` |
| **Working copy** | `/var/lib/git/work/eda-project` |
| **Rulebook subdir** | `rulebooks/` (EDA Project: Branch + subdirectory) |
| **Scope** | Non-production / demo only |

---

## Executive Summary: Configuration at a Glance

| # | Step | Key Actions | Required |
| --- | --- | --- | --- |
| 1 | **Local Git bare repo** | Create `git` user, bare repo, root→git SSH | ✅ |
| 2 | **Init rulebooks/** | Clone workspace, first commit push to `main` | ✅ |
| 3 | **Gateway UI** | EDA Credentials, Project, Event Stream | ✅ |
| 4 | **Rulebook Activation** | Sync Project → create Activation | ✅ |
| 5 | **Event Stream test** | `curl -k -X POST` + Basic Auth | ✅ |
| 6 | **AIOps rulebooks** | Performance / network alerts → Job Template | Recommended |

---

## 1. Local Git Repository on AAP 2.6

> **Scope**: Non-production / demo. Bare repo path: `/var/lib/git/eda-project.git`. SSH uses root's `/root/.ssh/id_rsa` and `id_rsa.pub` (public key for `git` user; private key in AAP credential).

### 1.1 Important Notes

| Point | Description |
| --- | --- |
| **Bare repo** | `/var/lib/git/eda-project.git` stores Git objects — **do not** hand-edit rulebooks inside it |
| **Rulebook layout** | Put YAML under `rulebooks/*.yml`; EDA Project uses **Branch** + subdirectory `rulebooks/` |
| **Local editing** | Use a separate working tree beside the bare repo, e.g. `/var/lib/git/work/eda-project` |

### 1.2 Define Variables

As root:

```bash
export AAP_SSH_HOST="10.210.65.24"
```

### 1.3 Install Git, User, Bare Repo, SSH Auth

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

### 1.4 SSH Self-Test

Root connects to `git@localhost` with its private key:

```bash
ssh-keyscan -H "${AAP_SSH_HOST}" >> /root/.ssh/known_hosts 2>/dev/null
ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes "git@${AAP_SSH_HOST}"
```

Expect a non-interactive `git` shell or Git message (key match). If shell login is denied, rely on successful AAP Project Sync.

### 1.5 Workspace + rulebooks/ + First Push (main)

```bash
mkdir -p /var/lib/git/work
cd /var/lib/git/work
rm -rf eda-project
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git clone "git@${AAP_SSH_HOST}:/var/lib/git/eda-project.git" eda-project
cd /var/lib/git/work/eda-project

git checkout -b main 2>/dev/null || git branch -M main
mkdir -p rulebooks
printf '%s\n' '# Place your rulebook YAML here' > rulebooks/README.txt
git add rulebooks
git config user.email "root@aap26.example.com"
git config user.name "aap26 lab"
git commit -m "init: rulebooks/"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git push -u origin main
```

---

## 2. Commit Rulebooks to Git (Ongoing)

Full commit flow (copy-paste ready):

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

If already in `/var/lib/git/work/eda-project/rulebooks`:

```bash
git add rulebook_test.yml
git commit -m "add: rulebook_test.yml for EDA"
GIT_SSH_COMMAND="ssh -i /root/.ssh/id_rsa -o IdentitiesOnly=yes" \
  git push origin main
```

After push: **Sync EDA Project** in Gateway UI, then create or update Rulebook Activation.

---

## 3. Rulebook Examples

### 3.1 Test / Debug Rulebooks

After Git commit, configure **Rulebook Activations** in UI for event source testing.

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
        # UI Event Stream association usually overrides this
        port: 5000
  rules:
    - name: "Log full event payload"
      condition: true
      action:
        debug:
```

### 3.2 AIOps Use Case: Complex Issues RCA

**`rulebooks/01_eda_rule_linuxperformancealerts_ui.yml`**

```yaml
---
# Rulebook (AAP UI / Event Stream)
# - No run_module / run_playbook (avoids inventory errors in UI)
# - Side effects in Job Template playbook
# - Matches HighSystemCpuUsage / CriticalSystemCpuUsage
# - Passes event.payload via job_args.extra_vars.webhook_payload

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

> Pre-create Job Template `01_EDA_action_linuxperformancealerts_UI` with playbook `01_EDA_action_linuxperformancealerts_UI.yml`, Inventory, and Project.

**`rulebooks/02_eda_rule_linux_network_down.yml`**

```yaml
---
# Rulebook (AAP UI / Event Stream) — NetworkInterfaceDown alerts
# Same design pattern as 01_eda_rule_linuxperformancealerts_ui.yml

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

## 4. EDA Configuration in AAP UI

### 4.1 Git Credentials

Configure **Automation Decisions** Git authentication Credentials.

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/852c8745-849b-470f-99df-55518a8dc640" />


### 4.2 EDA Project

Configure EDA Project (SCM URL, Branch `main`, subdirectory `rulebooks/`, etc.).

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/4523626c-8f72-40e8-978b-dd0299eac33a" />


### 4.3 Event Stream Credentials

Configure Credentials for **EDA Event Stream** authentication.
<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/30406282-971b-4b38-be72-60f889f1bc9a" />



### 4.4 Rulebook Credentials

Configure Credentials for **EDA Rulebook** authentication.

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/8b509d0a-22c8-46e1-bc25-29282f4f8a7d" />


### 4.5 EDA Event Stream

Create EDA Event Stream.

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/7068f6eb-bc8e-4337-8ea9-29907c547003" />


Record the Event Stream **POST URL** for rulebooks and testing. DEMO example:

```
https://aap26.example.com:443/eda-event-streams/api/eda/v1/external_event_stream/a8e18ec4-ede4-43be-a3d9-dbc83701b811/post/
```

> UUID (`a8e18ec4-...`) is assigned when created in UI — use your actual value.

---

## 5. Test Event Stream & Rulebook

### 5.1 Write and Commit Test Rulebook

Example working directory:

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
        port: 5000
  rules:
    - name: "Log full event payload"
      condition: true
      action:
        debug:
```

Commit and push:

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

### 5.2 Configure Rulebook Activation

1. **Sync EDA Project** in UI
2. Configure **Rulebook Activation** (Event Stream, rulebook file, etc.)

<img width="2560" height="1347" alt="image" src="https://github.com/user-attachments/assets/e5920a07-23e5-4e56-a2a8-6d9a6f9548c7" />


### 5.3 CLI Event Stream Test

On the server:

```bash
curl -k -X POST \
  'https://aap26.example.com:443/eda-event-streams/api/eda/v1/external_event_stream/a8e18ec4-ede4-43be-a3d9-dbc83701b811/post/' \
  -u 'event_stream:redhat' \
  -H 'Content-Type: application/json' \
  -d '{"message": "Hello from manual header"}'
```

| Parameter | Description |
| --- | --- |
| **URL** | POST URL from Event Stream detail page |
| **`-u event_stream:redhat`** | Event Stream username/password from EDA Credentials (DEMO example) |

### 5.4 Confirm in UI

Check Rulebook Activation event details in Gateway UI — payload received and rule executed.
<img width="751" height="484" alt="image" src="https://github.com/user-attachments/assets/d4663ff6-8e05-47b2-abfd-1c5f7161559f" />
<img width="747" height="481" alt="image" src="https://github.com/user-attachments/assets/245ed85b-4265-4d2e-a2f2-15d5f36b10b7" />

---

## Post-Configuration Checklist

| # | Check | Pass Criteria |
| --- | --- | --- |
| 1 | Git bare repo | `/var/lib/git/eda-project.git` exists; `git@host` SSH works |
| 2 | rulebooks/ | `main` branch contains `rulebooks/*.yml` |
| 3 | EDA Project | Sync succeeds; rulebooks visible |
| 4 | Event Stream | POST URL recorded; `curl` test succeeds |
| 5 | Rulebook Activation | Running; test events visible in UI |
| 6 | AIOps rulebooks | Performance / network alert rulebooks trigger Job Templates |

---

## References

- [03-04 AAP Configuration](03-04-AAP-Configuration-EN.md)
- [03-05 AAP MCP For Agent](03-05-AAP-MCP-For-Agent-EN.md)
- [Event-Driven Ansible in AAP 2.6](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/event-driven_ansible)

> **Note**: This is the final chapter in the 03-aap series. Continue with Prometheus Event Stream sources, n8n integration, etc. in other DEMO guides.
