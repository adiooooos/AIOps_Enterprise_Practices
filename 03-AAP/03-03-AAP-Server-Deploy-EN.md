# AAP Server Deploy

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Install entry point** | Edit `inventory-growth` in the bundle directory from [03-02](03-02-AAP-Preparation-EN.md), then run `ansible-playbook` as **`admin`** to install the Container Growth topology. |
| ② | **Offline bundle** | DEMO uses `bundle_install=true` and a local `bundle/` directory — no external image pull during install. |
| ③ | **MCP co-located** | AIOps DEMO enables the `[ansiblemcp]` group on the same host as Gateway / Controller / Hub / EDA for later AI Agent integration (see [03-05](03-05-AAP-MCP-For-Agent-CN.md)). |

## Chapter Objective

With host preparation complete and the offline bundle extracted, install AAP 2.6 **containerized Growth topology** — configure `inventory-growth`, run the install playbook, verify containers, then proceed to [03-04 Configuration](03-04-AAP-Configuration-CN.md).

| Item | Description |
| --- | --- |
| **Version** | Ansible Automation Platform **2.6** (Containerized) |
| **Topology** | [Container Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies) · single VM |
| **Inventory file** | `inventory-growth` |
| **Example node** | `aap26.example.com` · `10.210.65.24` |
| **Run as** | `admin` |
| **Working directory** | `/home/admin/ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64` |

---

## Executive Summary: Deploy Steps at a Glance

| # | Step | Key Actions | Required |
| --- | --- | --- | --- |
| 1 | **Enter bundle directory** | `cd` into the extracted installer directory | ✅ |
| 2 | **Edit inventory** | Fill `inventory-growth` for DEMO nodes | ✅ |
| 3 | **Confirm offline bundle** | `bundle_install=true`, `bundle_dir` points to `./bundle` | ✅ |
| 4 | **Run install** | `ansible-playbook -i inventory-growth ansible.containerized_installer.install` | ✅ |
| 5 | **Verify containers** | `podman ps` — Gateway / Controller / Hub / EDA / MCP, etc. | ✅ |
| 6 | **Access UI** | Browser → `https://<FQDN>` | ✅ |

---

## 1. Pre-flight Checks

Confirm the [03-02 checklist](03-02-AAP-Preparation-EN.md#pre-deployment-checklist) is complete:

```bash
su - admin
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64
ls inventory-growth bundle/
```

| Check | Expected |
| --- | --- |
| `inventory-growth` | Present and editable |
| `bundle/` | Contains offline images and dependencies |
| `ansible-playbook` | `ansible --version` works |
| FQDN | `hostname -f` → `aap26.example.com` |

---

## 2. Configure `inventory-growth`

Edit `inventory-growth` in the bundle directory (based on the bundle `2.6-7` template). AIOps DEMO applies these changes on top of the official template:

| # | Change | DEMO setting |
| --- | --- | --- |
| 1 | **All host groups** | `automationgateway` / `automationcontroller` / `automationhub` / `automationeda` / `database` all point to `aap26.example.com` |
| 2 | **Offline install** | `bundle_install=true`, `bundle_dir` points to `./bundle` under the current directory |
| 3 | **Component passwords** | Gateway / Controller / Hub / EDA and PostgreSQL passwords set to `redhat` (demo only) |
| 4 | **Controller memory** | `controller_percent_memory_capacity=0.5` |
| 5 | **Hub seed collections** | `hub_seed_collections=false` |
| 6 | **Ansible MCP** | Enable `[ansiblemcp]` group at end of file; append a second `[all:vars]` block for MCP permissions |
| 7 | **Lightspeed** | Leave commented out (not deployed in DEMO) |

> **Password hygiene**: DEMO uses `redhat` everywhere for convenience; production requires strong per-component and database passwords. Full optional variables: [Inventory File Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars).

Verify after editing; **see the [Appendix](#appendix-complete-inventory-growth-file) at the end for the full file**.

```bash
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-7-x86_64
more inventory-growth
```

---

## 3. Run Installation

As **`admin`** from the bundle directory:

```bash
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64
ansible-playbook -i inventory-growth ansible.containerized_installer.install
```

| Item | Notes |
| --- | --- |
| **Duration** | ~30–60 minutes (hardware and disk IOPS dependent) |
| **Identity** | Must run as `admin` (`linger` and `sudo` configured in 03-02) |
| **Troubleshooting** | Check playbook output; common issues: insufficient `/home`, subscription/repos, wrong `bundle_dir` |

On success, components run as **Podman containers** under `admin` systemd user services.

---

## 4. Post-Install Verification

> Run as **`admin`** over SSH (do not use `sudo su` before running `podman`).

### 4.1 Container Status (primary check)

```bash
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Pass criteria**: all core containers show `Up`, including `ansiblemcp`. Example from DEMO environment:

```
NAMES                               STATUS      PORTS
postgresql                          Up 2 weeks  5432/tcp
redis-unix                          Up 2 weeks  6379/tcp
redis-tcp                           Up 2 weeks  6379/tcp
automation-gateway-proxy            Up 2 weeks
automation-gateway                  Up 2 weeks
receptor                            Up 2 weeks
automation-controller-rsyslog       Up 2 weeks  8052/tcp
automation-controller-task          Up 2 weeks  8052/tcp
automation-controller-web           Up 2 weeks  8052/tcp
automation-eda-api                  Up 2 weeks
automation-eda-daphne               Up 2 weeks
automation-eda-web                  Up 2 weeks  8080/tcp, 8443/tcp
automation-eda-worker-1             Up 2 weeks
automation-eda-worker-2             Up 2 weeks
automation-eda-activation-worker-1  Up 2 weeks
automation-eda-activation-worker-2  Up 2 weeks
automation-eda-scheduler            Up 2 weeks
automation-hub-api                  Up 2 weeks
automation-hub-content              Up 2 weeks
automation-hub-web                  Up 2 weeks  8080/tcp, 8443/tcp
automation-hub-worker-1             Up 2 weeks
automation-hub-worker-2             Up 2 weeks
ansiblemcp                          Up 2 weeks  8080/tcp, 8086/tcp
```

| Component | Key container names |
| --- | --- |
| **Gateway** | `automation-gateway`, `automation-gateway-proxy` |
| **Controller** | `automation-controller-web`, `automation-controller-task` |
| **Hub** | `automation-hub-web`, `automation-hub-api` |
| **EDA** | `automation-eda-web`, `automation-eda-worker-*` |
| **Database / Redis** | `postgresql`, `redis-tcp` |
| **MCP** | `ansiblemcp` |

> The `PORTS` column shows **container-internal** ports.

### 4.2 Web UI

| Item | Value |
| --- | --- |
| **URL** | `https://aap26.example.com` |
| **Default user** | `admin` |
| **Password** | `gateway_admin_password` from inventory (DEMO: `redhat`) |

```bash
curl -k -I https://aap26.example.com
# Expected: HTTP/1.1 200 OK, server: envoy
```

The browser should show the AAP Gateway login page. License activation, MCP Token, and AI Agent wiring are in [03-04 AAP Configuration](03-04-AAP-Configuration-CN.md).

---

## Post-Deployment Checklist

| # | Check | Pass Criteria |
| --- | --- | --- |
| 1 | Playbook | `ansible.containerized_installer.install` completes with no failed tasks |
| 2 | Containers | `podman ps` shows core AAP containers Running |
| 3 | Gateway UI | `curl -k -I https://<FQDN>` returns `200 OK`; browser loads login page |
| 4 | Login | `admin` / configured password works |
| 5 | License | Pending activation in [03-04](03-04-AAP-Configuration-CN.md) |

---

## References

- [03-02 AAP Preparation](03-02-AAP-Preparation-EN.md)
- [Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation)
- [Inventory File Variables](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars)
- [Container Topologies — Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies)

---

## Appendix: Complete `inventory-growth` File

Below is the **full `inventory-growth`** as configured for AIOps DEMO (under `ansible-automation-platform-containerized-setup-bundle-2.6-7-x86_64`):

```ini
# This is the AAP installer inventory file intended for the Container growth deployment topology.
# This inventory file expects to be run from the host where AAP will be installed.
# Please consult the Ansible Automation Platform product documentation about this topology's tested hardware configuration.
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies
#
# Please consult the docs if you're unsure what to add
# For all optional variables please consult the included README.md
# or the Ansible Automation Platform documentation:
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation

# This section is for your AAP Gateway host(s)
# -----------------------------------------------------
[automationgateway]
aap26.example.com
# This section is for your AAP Controller host(s)
# -----------------------------------------------------
[automationcontroller]
aap26.example.com
# This section is for your AAP Automation Hub host(s)
# -----------------------------------------------------
[automationhub]
aap26.example.com
# This section is for your AAP EDA Controller host(s)
# -----------------------------------------------------
[automationeda]
aap26.example.com
# This section is for your AAP Lightspeed host(s)
# -----------------------------------------------------
# [ansiblelightspeed]
# aap.example.org

# This section is for your Ansible MCP Server host(s)
# -----------------------------------------------------
# [ansiblemcp]
# aap.example.org

# This section is for the AAP database
# -----------------------------------------------------
[database]
aap26.example.com
[all:vars]
# Ansible
ansible_connection=local
validate_certs=false

# Common variables
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#general-variables
# -----------------------------------------------------
postgresql_admin_username=postgres
postgresql_admin_password=redhat

bundle_install=true
# The bundle directory must include /bundle in the path
bundle_dir='{{ lookup("ansible.builtin.env", "PWD") }}/bundle'


redis_mode=standalone

# AAP Gateway
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#platform-gateway-variables
# -----------------------------------------------------
gateway_admin_password=redhat
gateway_pg_host=aap26.example.com
gateway_pg_password=redhat

# AAP Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#controller-variables
# -----------------------------------------------------
controller_admin_password=redhat
controller_pg_host=aap26.example.com
controller_pg_password=redhat
controller_percent_memory_capacity=0.5

# AAP Automation Hub
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#hub-variables
# -----------------------------------------------------
hub_admin_password=redhat
hub_pg_host=aap26.example.com
hub_pg_password=redhat
hub_seed_collections=false

# AAP EDA Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#event-driven-ansible-variables
# -----------------------------------------------------
eda_admin_password=redhat
eda_pg_host=aap26.example.com
eda_pg_password=redhat

# AAP Lightspeed
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/appendix-inventory-files-vars#lightspeed-variables
# -----------------------------------------------------
# lightspeed_admin_password=<set your own>
# lightspeed_pg_host=aap.example.org
# lightspeed_pg_password=<set your own>

# In case chabot is enabled, default provider is "rhoai"
# lightspeed_chatbot_model_url=<set your own>
# lightspeed_chatbot_model_api_key=<set your own>
# lightspeed_chatbot_model_id=<set your own>

# In case "azure" provider
# lightspeed_chatbot_default_provider = "azure"

# In case "openai" provider
# lightspeed_chatbot_default_provider = "openai"

# lightspeed_mcp_controller_enabled=true
# lightspeed_mcp_lightspeed_enabled=true
# lightspeed_wca_model_api_key=<set your own>
# lightspeed_wca_model_id=<set your own>
[ansiblemcp]
aap26.example.com

# This section is for Ansible MCP server permissions
# --------------------------------------------------
[all:vars]
mcp_allow_write_operations=true
mcp_ignore_certificate_errors=true
#mcp_tls_cert= <path to tls certificate>
#mcp_tls_key= <path to tls key>
```

> **Next:** [03-04 AAP Configuration](03-04-AAP-Configuration-CN.md) — activate license, configure UI and MCP Token
