# MCP for SSH 部署及配置

> **状态**：20260609 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 说明 |
| --- | --- | --- |
| ① | **部署位置** | `Mattermost.example.com` / `10.210.65.14`（与 Triage / Mattermost 同机） |
| ② | **镜像** | [`firstfinger/ssh-mcp`](https://hub.docker.com/r/firstfinger/ssh-mcp)（Podman，离线导入） |
| ③ | **访问端点** | `http://10.210.65.14:3000/mcp`（供 N8N MCP Client 调用） |
| ④ | **核心能力** | 容器托管 SSH 身份 → AI Agent 远程诊断 / 修复 |

```
n8n (10.210.65.103)  ──HTTP Streamable──▶  ssh-mcp Podman (10.210.65.14:3000)
                                                    │
                                    SSH ──▶ chaos / prometheus / aap26 / ...
```

**关键设计**：容器在命名卷 `/data` 自动生成 `id_ed25519`，公钥一次性分发到各目标 `authorized_keys`；AI 调用 `connect(host, username, alias)` 即可，无需传密钥。

---

## 一、部署前准备

> 以下命令均在 **Mattermost 节点（10.210.65.14）以 root 执行**。

### 1.1 验证 root 对各目标免密 SSH

```bash
for IP in 10.210.65.148 10.210.65.174 10.210.65.24 10.210.65.17 10.210.65.103; do
  ssh -o StrictHostKeyChecking=accept-new root@$IP hostname
done
```

如某台未通，先 `ssh-copy-id root@<IP>`。

### 1.2 工作目录与命名卷

```bash
mkdir -p /opt/mcp-ssh

podman volume create ssh-mcp-data
podman volume ls | grep ssh-mcp-data
```

### 1.3 防火墙（仅放行 N8N 来源 IP）

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.103/32" port port="3000" protocol="tcp" accept'
firewall-cmd --reload
```

---

## 二、离线导入镜像

### 2.1 Win11：pull → save → scp

```powershell
docker pull firstfinger/ssh-mcp:latest
docker save -o ssh-mcp_latest.tar firstfinger/ssh-mcp:latest
scp ssh-mcp_latest.tar root@10.210.65.14:/opt/mcp-ssh/
```

### 2.2 Mattermost：load

```bash
podman load -i /opt/mcp-ssh/ssh-mcp_latest.tar
podman images | grep ssh-mcp
# docker.io/firstfinger/ssh-mcp   latest   ...   ~11.7 MB
```

---

## 三、启动容器

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

**预期日志**：

```text
starting ssh-mcp commit=... mode=http port=8000 ...
listening on :8000/mcp
```

---

## 四、分发容器托管公钥（一次性，必做）

> scratch 镜像无 shell，用 `podman cp` 取出公钥后从宿主机分发。

### 4.1 取容器公钥

```bash
podman cp mcp-ssh:/data/id_ed25519.pub /tmp/mcp-ssh.pub
cat /tmp/mcp-ssh.pub
# 形如：ssh-ed25519 AAAA...xxxxx SSH-MCP
```

若文件不存在 → `podman restart mcp-ssh && sleep 3`，再重试。

### 4.2 分发到所有目标主机

```bash
PUBKEY=$(cat /tmp/mcp-ssh.pub)
for IP in 10.210.65.148 10.210.65.174 10.210.65.24 10.210.65.17 10.210.65.103; do
  ssh -o StrictHostKeyChecking=accept-new root@$IP \
    "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
     grep -qxF '$PUBKEY' ~/.ssh/authorized_keys 2>/dev/null || echo '$PUBKEY' >> ~/.ssh/authorized_keys && \
     chmod 600 ~/.ssh/authorized_keys && echo OK on \$(hostname)"
done
```

每台应回显 `OK on <hostname>`。

### 4.3 通过 MCP API 验证

Streamable HTTP MCP 协议要求 **initialize → notifications/initialized → tools/call** 三步握手，单次 curl 必返回 `Invalid session ID`，下面是完整脚本：

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

**预期最后一行**：

```json
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"Connected to root@10.210.65.148 (alias: chaos)"}]}}
```

若返回 `permission denied (publickey)` → 4.2 未在该主机生效，重跑分发。

---

## 五、N8N 对接

### 5.1 MCP Client Tool 节点

| 参数 | 值 |
| --- | --- |
| **Credential** | 留空（防火墙白名单已限制访问源） |
| **Server Transport** | `HTTP Streamable` |
| **Endpoint URL** | `http://10.210.65.14:3000/mcp` |
| **Tools to Include** | `All` |

### 5.2 AI Agent System Prompt（必填）

把以下内容粘贴到 AI Agent 节点的 `System Message`：

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
or in another way PINGME fzhang@redhat.com
```

---

## 六、故障排查（速查）

| 现象 | 处理 |
| --- | --- |
| N8N 跨机访问拒绝连接 | 检查 §1.3 防火墙规则 |
| MCP `tools/call` 返回 `Invalid session ID` | 走 §4.3 三步握手；N8N MCP Client 已自动处理 |
| `connect` 返回 `permission denied (publickey)` | 该目标 `authorized_keys` 未写入，重跑 §4.2 |
| `podman cp` 看不到 `/data/id_ed25519.pub` | `podman restart mcp-ssh`，等 3s 再取 |
| AI 反复问"请提供密码 / 密钥" | System Prompt 未粘贴或被覆盖，回到 §5.2 |
| 删除卷 `ssh-mcp-data` 后 SSH 全失败 | 重新执行第四节（取新公钥 + 重新分发） |

```bash
podman ps --filter name=mcp-ssh
podman logs --tail 50 mcp-ssh
curl -i --max-time 2 http://127.0.0.1:3000/mcp        # 应返回 200 + text/event-stream
```

---

## 参考

- [firstfinger/ssh-mcp · Docker Hub](https://hub.docker.com/r/firstfinger/ssh-mcp)
- [N8N MCP Client Tool 文档](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)
- [Model Context Protocol 规范](https://modelcontextprotocol.io/specification/)
