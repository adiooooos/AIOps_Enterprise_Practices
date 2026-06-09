# MCP for SSH Deployment and Configuration

> **Status**: 20260609 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Description |
| --- | --- | --- |
| ① | **Deploy host** | `Mattermost.example.com` / `10.210.65.14` (co-located with Triage / Mattermost) |
| ② | **Image** | [`firstfinger/ssh-mcp`](https://hub.docker.com/r/firstfinger/ssh-mcp) (Podman, offline import) |
| ③ | **Endpoint** | `http://10.210.65.14:3000/mcp` (for N8N MCP Client) |
| ④ | **Core capability** | Container-managed SSH identity → AI Agent remote diagnostics / remediation |

```
n8n (10.210.65.103)  ──HTTP Streamable──▶  ssh-mcp Podman (10.210.65.14:3000)
                                                    │
                                    SSH ──▶ chaos / prometheus / aap26 / ...
```

**Key design**: the container auto-generates `id_ed25519` on the named volume `/data`; distribute the public key once to each target’s `authorized_keys`. AI calls `connect(host, username, alias)` only—no key parameters required.

---

## 1. Pre-Deployment

> Run all commands below on the **Mattermost host (10.210.65.14) as root**.

### 1.1 Verify passwordless SSH from root to all targets

```bash
for IP in 10.210.65.148 10.210.65.174 10.210.65.24 10.210.65.17 10.210.65.103; do
  ssh -o StrictHostKeyChecking=accept-new root@$IP hostname
done
```

If any host fails, run `ssh-copy-id root@<IP>` first.

### 1.2 Working directory and named volume

```bash
mkdir -p /opt/mcp-ssh

podman volume create ssh-mcp-data
podman volume ls | grep ssh-mcp-data
```

### 1.3 Firewall (allow N8N source IP only)

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.103/32" port port="3000" protocol="tcp" accept'
firewall-cmd --reload
```

---

## 2. Offline Image Import

### 2.1 Win11: pull → save → scp

```powershell
docker pull firstfinger/ssh-mcp:latest
docker save -o ssh-mcp_latest.tar firstfinger/ssh-mcp:latest
scp ssh-mcp_latest.tar root@10.210.65.14:/opt/mcp-ssh/
```

### 2.2 Mattermost: load

```bash
podman load -i /opt/mcp-ssh/ssh-mcp_latest.tar
podman images | grep ssh-mcp
# docker.io/firstfinger/ssh-mcp   latest   ...   ~11.7 MB
```

---

## 3. Start Container

```bash
podman rm -f mcp-ssh 2>/dev/null

podman run -d \
  --name mcp-ssh \
  --restart=always \
  --no-healthcheck \
  -p 3000:8000 \
  -e SSH_MCP_GLOBAL_STATE=true \
  -e SSH_MCP_SESSION_TIMEOUT=900 \
  -v ssh-mcp-data:/data \
  docker.io/firstfinger/ssh-mcp:latest

sleep 3
podman ps --filter name=mcp-ssh
podman logs --tail 5 mcp-ssh
```

**Expected logs**:

```text
starting ssh-mcp commit=... mode=http port=8000 ...
listening on :8000/mcp
```

---

## 4. Distribute Container-Managed Public Key (one-time, required)

> The scratch image has no shell; use `podman cp` to extract the public key, then distribute from the host.

### 4.1 Extract container public key

```bash
podman cp mcp-ssh:/data/id_ed25519.pub /tmp/mcp-ssh.pub
cat /tmp/mcp-ssh.pub
# 形如：ssh-ed25519 AAAA...xxxxx SSH-MCP
```

If the file is missing → `podman restart mcp-ssh && sleep 3`, then retry.

### 4.2 Distribute to all target hosts

```bash
PUBKEY=$(cat /tmp/mcp-ssh.pub)
for IP in 10.210.65.148 10.210.65.174 10.210.65.24 10.210.65.17 10.210.65.103; do
  ssh -o StrictHostKeyChecking=accept-new root@$IP \
    "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
     grep -qxF '$PUBKEY' ~/.ssh/authorized_keys 2>/dev/null || echo '$PUBKEY' >> ~/.ssh/authorized_keys && \
     chmod 600 ~/.ssh/authorized_keys && echo OK on \$(hostname)"
done
```

Each host should print `OK on <hostname>`.

### 4.3 Verify via MCP API

Streamable HTTP MCP requires a **initialize → notifications/initialized → tools/call** three-step handshake; a single curl returns `Invalid session ID`. Full script:

```bash
SID=$(curl -s -i -X POST http://127.0.0.1:3000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"1"}}}' \
  | grep -i '^mcp-session-id:' | awk '{print $2}' | tr -d '\r')

curl -s -X POST http://127.0.0.1:3000/mcp \
  -H "mcp-session-id: $SID" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -s -X POST http://127.0.0.1:3000/mcp \
  -H "mcp-session-id: $SID" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"connect","arguments":{"host":"10.210.65.148","username":"root","alias":"chaos"}}}'
```

**Expected last line**:

```json
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"Connected to root@10.210.65.148 (alias: chaos)"}]}}
```

If you get `permission denied (publickey)` → §4.2 did not apply on that host; re-run distribution.

---

## 5. N8N Integration

### 5.1 MCP Client Tool node

| Parameter | Value |
| --- | --- |
| **Credential** | Leave empty (firewalld whitelist restricts source) |
| **Server Transport** | `HTTP Streamable` |
| **Endpoint URL** | `http://10.210.65.14:3000/mcp` |
| **Tools to Include** | `All` |

### 5.2 AI Agent System Prompt (required)

Paste the following into the AI Agent node **System Message**:

```text
You are an AIOps assistant with access to an SSH MCP tool. Follow these rules strictly.

# Connection rules
The MCP server holds a managed SSH identity. Every connect() call MUST use exactly these three arguments:
  connect(
    host="<target-IP>",
    username="root",
    alias="<short-name>"
  )
Never pass password, private_key_path, or any other auth argument. Never ask the user for credentials.

# Available hosts
| alias       | host IP        | role                              |
|-------------|----------------|-----------------------------------|
| aap26       | 10.210.65.24   | Ansible Automation Platform       |
| testrhel79  | 10.210.65.17   | RHEL 7.9 test                     |
| n8n         | 10.210.65.103  | N8N                               |
| prometheus  | 10.210.65.174  | Prometheus                        |
| chaos       | 10.210.65.148  | Fault simulation (primary target) |
| mattermost  | 10.210.65.14   | Self                              |

# Workflow
1. Call connect() first with the 3 required args.
2. Then use run(command, target=<alias>), usage(target=<alias>), ps(target=<alias>), etc.
3. Read-only commands only: top -bn1, free -h, df -h, iostat, journalctl, systemctl status, ps.
4. NEVER execute: rm, reboot, shutdown, mkfs, dd.
5. End with a structured summary: Findings → Probable cause → Recommended action.
```

---

## 6. Troubleshooting (Quick Reference)

| Symptom | Action |
| --- | --- |
| N8N cross-host connection refused | Check §1.3 firewall rule |
| MCP `tools/call` returns `Invalid session ID` | Use §4.3 three-step handshake; N8N MCP Client handles this automatically |
| `connect` returns `permission denied (publickey)` | Target `authorized_keys` missing key; re-run §4.2 |
| `podman cp` cannot find `/data/id_ed25519.pub` | `podman restart mcp-ssh`, wait 3s, retry |
| AI keeps asking for password / key | System Prompt missing or overwritten; restore §5.2 |
| SSH fails after deleting `ssh-mcp-data` volume | Re-run §4 (new public key + redistribute) |

```bash
podman ps --filter name=mcp-ssh
podman logs --tail 50 mcp-ssh
curl -i --max-time 2 http://127.0.0.1:3000/mcp        # 应返回 200 + text/event-stream
```

---

## References

- [firstfinger/ssh-mcp · Docker Hub](https://hub.docker.com/r/firstfinger/ssh-mcp)
- [N8N MCP Client Tool docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)
- [Model Context Protocol specification](https://modelcontextprotocol.io/specification/)
