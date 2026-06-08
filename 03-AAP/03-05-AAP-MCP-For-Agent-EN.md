# AAP MCP Server → AI Agent Connection

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Post-install wiring** | MCP is installed with the platform in [03-03](03-03-AAP-Server-Deploy-EN.md); this chapter connects MCP endpoints to **Cursor / n8n** after [03-04](03-04-AAP-Configuration-EN.md). May be done once the AI Agent is ready. |
| ② | **Verify on your env** | AIOps DEMO verified: **`http://<IP>:3000/mcp/<toolset>`** (matches Cursor `mcp.json`); some RH docs use **8448/HTTPS** + `/<toolset>/mcp` — see §2.1. |
| ③ | **Dual-layer auth** | Connection requires an **API Token** (inherits user RBAC) plus MCP server-side permissions from inventory (DEMO: `mcp_allow_write_operations=true`). |

## Chapter Objective

Create an API Token for Ansible MCP Server and configure MCP endpoints in AI Agents (Cursor, n8n, etc.) so natural-language prompts can query the AAP environment, launch job templates, and manage inventories.

| Item | Description |
| --- | --- |
| **Version** | AAP **2.6.4+** (MCP Technology Preview) |
| **MCP container** | `ansiblemcp` (enabled in [03-03](03-03-AAP-Server-Deploy-EN.md) inventory) |
| **Example node** | `aap26.example.com` · `10.210.65.24` |
| **MCP Base URL (DEMO verified)** | `http://10.210.65.24:3000` |

---

## Executive Summary: Configuration at a Glance

| # | Step | Key Actions | Required |
| --- | --- | --- | --- |
| 1 | **Confirm MCP running** | `ansiblemcp` shows `Up` in `podman ps` | ✅ |
| 2 | **Create API Token** | Gateway UI → Users → Tokens; Scope Write (DEMO) | ✅ |
| 3 | **Save Token** | Copy and store securely (shown only once) | ✅ |
| 4 | **Configure AI Agent** | Add endpoints + Bearer Token in Cursor `mcp.json` or n8n MCP Client | ✅ |
| 5 | **Verify** | Ask the AI which MCP tools are available | ✅ |

---

## 1. Prerequisites

| # | Requirement | Verification |
| --- | --- | --- |
| 1 | AAP containerized install complete | [03-03](03-03-AAP-Server-Deploy-EN.md) playbook succeeded |
| 2 | `[ansiblemcp]` in inventory | 03-03 appendix `inventory-growth` |
| 3 | AAP license activated | [03-04](03-04-AAP-Configuration-EN.md) — no expiry warning in UI |
| 4 | `ansiblemcp` container running | `podman ps --filter name=ansiblemcp` |
| 5 | AI Agent client ready | Cursor / n8n (may install later) |

---

## 2. MCP Endpoints

### 2.1 Why two URL formats appear in docs?

| Source | Example | Notes |
| --- | --- | --- |
| **AIOps DEMO verified (recommended)** | `http://10.210.65.24:3000/mcp/job_management` | Matches [ansible/aap-mcp-server](https://github.com/ansible/aap-mcp-server) defaults; **Cursor in this env successfully calls Job API** |
| **Red Hat containerized install docs** | `https://aap26.example.com:8448/job_management/mcp` | Some AAP 2.6 guides; port/path may vary by bundle version |

> **Bottom line**: Configure AI Agents from **what works in your environment**. If Cursor/n8n already lists MCP tools and queries jobs, **do not** switch to 8448 blindly. Differences: **port (3000 vs 8448)**, **protocol (HTTP vs HTTPS)**, **path order (`/mcp/<toolset>` vs `/<toolset>/mcp`)**.

### 2.2 DEMO endpoints (verified)

| Toolset | Purpose | Endpoint URL |
| --- | --- | --- |
| **Job Management** | List / launch job templates; view job status and logs | `http://10.210.65.24:3000/mcp/job_management` |
| **Inventory Management** | Query inventories, hosts, groups | `http://10.210.65.24:3000/mcp/inventory_management` |
| **System Monitoring** | Platform health, job logs, audit | `http://10.210.65.24:3000/mcp/system_monitoring` |
| **User Management** | Users, teams, RBAC | `http://10.210.65.24:3000/mcp/user_management` |
| **Security / Compliance** | Credentials and security policies | `http://10.210.65.24:3000/mcp/security_compliance` |
| **Platform Configuration** | System settings, license, execution environments | `http://10.210.65.24:3000/mcp/platform_configuration` |

FQDN also works: `http://aap26.example.com:3000/mcp/job_management` (requires DNS or `/etc/hosts`).

> **Note**: A plain `curl` GET to `:3000` may return **404** — that does not mean MCP is down. MCP clients use **POST** with a Bearer Token. **Judge by whether the AI Agent can invoke tools.**

---

## 3. Create API Token

Create an OAuth 2 Token in AAP Gateway UI for the MCP client.

### 3.1 Procedure

1. Navigate to **Access Management → Users**
2. Select the **username** for MCP (typically `admin` or a dedicated service account)
3. Open the **Tokens** tab
4. Click **Create token** and fill in:

| Field | DEMO recommendation |
| --- | --- |
| **Application** | App name or Browse; leave blank for a Personal Access Token (PAT) |
| **Description** | Optional, e.g. `cursor-mcp-demo` |
| **Scope** | **Write** (DEMO needs job launch); use **Read** for read-only |

5. Click **Create token**
6. On the token page, click **Copy** and **save immediately** — the token is **shown only once**

> **Permission inheritance**: the token inherits the user's RBAC. Even with Write scope, writes are blocked if `mcp_allow_write_operations=false` in inventory.

### 3.2 Verify Token

1. **Access Management → OAuth Applications**
2. Select the associated Application (skip for PAT)
3. **Tokens** tab should list the new token

---

## 4. Configure MCP in AI Agent

### 4.1 Cursor (`mcp.json`)

Add AAP MCP servers to Cursor MCP config. Register each toolset separately (keep names **≤ 20 chars** to stay under typical 64-char combined identifier limits):

```json
{
  "mcpServers": {
    "aap-mcp-job-management": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/job_management",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    },
    "aap-mcp-inventory": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/inventory_management",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    },
    "aap-mcp-monitor": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/system_monitoring",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    },
    "aap-mcp-users": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/user_management",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    },
    "aap-mcp-security": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/security_compliance",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    },
    "aap-mcp-platform": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/platform_configuration",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    }
  }
}
```

> Replace `<YOUR_AAP_TOKEN>` with the token from §3. URLs match the live AIOps DEMO Cursor config.

**Minimal example** (Job Management only — same structure as production `mcp.json`):

```json
{
  "mcpServers": {
    "aap-mcp-job-management": {
      "type": "http",
      "url": "http://10.210.65.24:3000/mcp/job_management",
      "headers": {
        "Authorization": "Bearer <YOUR_AAP_TOKEN>"
      }
    }
  }
}
```

### 4.2 n8n

Add MCP Client Tool configuration at the end of the [n8n deployment chapter](../04-n8n/). Use URLs from §2; set Authentication to **Bearer Token** with the token from §3.

| Setting | Value |
| --- | --- |
| **Transport** | HTTP |
| **URL** | e.g. `http://10.210.65.24:3000/mcp/job_management` |
| **Authorization** | `Bearer <YOUR_AAP_TOKEN>` |

---

## 5. Verification & Troubleshooting

### 5.1 Functional Test

In the AI Agent chat, enter:

```
What MCP tools are available for my Ansible Automation Platform?
```

Or:

```
Give me a list of my Ansible Automation Platform jobs.
```

The AI should return MCP tool lists or job query results.

### 5.2 Common Issues

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `curl :3000` returns 404 | Plain GET is not MCP protocol | Expected; judge by AI Agent tool calls |
| AI Agent cannot connect | Wrong URL / token | Match §2.2; or try RH format `https://<FQDN>:8448/<toolset>/mcp` |
| Connection refused | `ansiblemcp` not running | `podman ps --filter name=ansiblemcp` |
| 401 / 403 | Invalid or expired token | Recreate token; update agent config |
| 400 Bad Request | Self-signed cert rejected | Set `mcp_ignore_certificate_errors=true` or use proper certs |
| 406 Not Acceptable | Non-JSON API output | Ask AI to request JSON format first |
| Write ops denied | Server read-only | Check `mcp_allow_write_operations` and token Scope |

---

## Post-Configuration Checklist

| # | Check | Pass Criteria |
| --- | --- | --- |
| 1 | MCP container | `ansiblemcp` container `Up` |
| 2 | API Token | Created and stored securely |
| 3 | Cursor / n8n | MCP server config saved and enabled |
| 4 | Connectivity | AI chat lists MCP tools |
| 5 | Permissions | DEMO can query inventory / launch jobs (Write scope) |

---

## References

- [03-03 AAP Server Deploy](03-03-AAP-Server-Deploy-EN.md) — `[ansiblemcp]` inventory
- [03-04 AAP Configuration](03-04-AAP-Configuration-EN.md)
- [Deploy Ansible MCP Server (AAP 2.6)](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/extend-assembly_deploying_ansible_mcp_server)
- [n8n MCP Client Tool docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)

> **Next:** [03-06 Automation Decisions Configuration](03-06-Automation-Decisions-CN.md)
