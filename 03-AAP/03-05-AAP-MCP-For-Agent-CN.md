# AAP MCP Server → AI Agent 连接配置

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 洞察 |
| --- | --- | --- |
| ① | **后置配置** | MCP 在 [03-03](03-03-AAP-Server-Deploy-CN.md) 随平台安装；本章在 [03-04](03-04-AAP-Configuration-CN.md) 完成后，将 MCP 端点接入 **Cursor / n8n** 等 AI Agent。也可在 AI Agent 部署就绪后再做。 |
| ② | **以实测为准** | AIOps DEMO 实测可用 **`http://<IP>:3000/mcp/<toolset>`**（与 Cursor `mcp.json` 一致）；Red Hat 部分文档另写 **8448/HTTPS** + `/<toolset>/mcp`，见 §2.1。 |
| ③ | **双因素鉴权** | 连接需 **API Token**（继承用户 RBAC）+ inventory 中 MCP 服务端权限（DEMO：`mcp_allow_write_operations=true`）。 |

## 本章目标

为 Ansible MCP Server 创建 API Token，并在 AI Agent（Cursor、n8n 等）中配置 MCP 端点，使自然语言可查询 AAP 环境、触发 Job Template、管理 Inventory 等自动化操作。

| 项目 | 说明 |
| --- | --- |
| **适用版本** | AAP **2.6.4+**（MCP Technology Preview） |
| **MCP 容器** | `ansiblemcp`（[03-03](03-03-AAP-Server-Deploy-CN.md) inventory 已启用） |
| **示例节点** | `aap26.example.com` · `10.210.65.24` |
| **MCP Base URL（DEMO 实测）** | `http://10.210.65.24:3000` |

---

## Executive Summary: 配置步骤一览

| # | 步骤 | 操作要点 | 必须 |
| --- | --- | --- | --- |
| 1 | **确认 MCP 运行** | `podman ps` 中 `ansiblemcp` 为 `Up` | ✅ |
| 2 | **创建 API Token** | Gateway UI → Users → Tokens，Scope 选 Write（DEMO） | ✅ |
| 3 | **记录 Token** | 复制并安全保存（仅显示一次） | ✅ |
| 4 | **配置 AI Agent** | 在 Cursor `mcp.json` 或 n8n MCP Client 填入 Endpoint + Bearer Token | ✅ |
| 5 | **验证连接** | 在 AI 对话中询问可用 MCP 工具 | ✅ |

---

## 1. 前置条件

| # | 条件 | 验证 |
| --- | --- | --- |
| 1 | AAP 容器化安装完成 | [03-03](03-03-AAP-Server-Deploy-CN.md) Playbook 成功 |
| 2 | `[ansiblemcp]` 已写入 inventory | 03-03 附录 `inventory-growth` |
| 3 | AAP 许可已激活 | [03-04](03-04-AAP-Configuration-CN.md) Gateway UI 无过期提示 |
| 4 | `ansiblemcp` 容器运行中 | `podman ps --filter name=ansiblemcp` |
| 5 | AI Agent 客户端就绪 | Cursor / n8n 等（可后装） |

---

## 2. MCP 端点一览

### 2.1 为何文档里会出现两种 URL？

| 来源 | 格式示例 | 说明 |
| --- | --- | --- |
| **AIOps DEMO 实测（推荐）** | `http://10.210.65.24:3000/mcp/job_management` | 与 [ansible/aap-mcp-server](https://github.com/ansible/aap-mcp-server) 默认一致；**本环境 Cursor 已验证可调用 Job API** |
| **Red Hat 容器化安装文档** | `https://aap26.example.com:8448/job_management/mcp` | 部分 AAP 2.6 安装指南中的暴露方式；路径顺序与端口可能因 bundle 版本而异 |

> **结论**：AI Agent 配置**以你环境实测为准**。若 Cursor / n8n 已能列出 MCP 工具并查询 Job，则**无需**改为 8448。两种格式差异在于 **端口（3000 vs 8448）**、**协议（HTTP vs HTTPS）**、**路径顺序（`/mcp/<toolset>` vs `/<toolset>/mcp`）**。

### 2.2 DEMO 环境端点（实测可用）

| Toolset | 用途 | Endpoint URL |
| --- | --- | --- |
| **Job Management** | 列出 / 启动作业模板、查看 Job 状态与日志 | `http://10.210.65.24:3000/mcp/job_management` |
| **Inventory Management** | 查询清单、主机、分组 | `http://10.210.65.24:3000/mcp/inventory_management` |
| **System Monitoring** | 平台健康检查、Job 日志、审计 | `http://10.210.65.24:3000/mcp/system_monitoring` |
| **User Management** | 用户、团队、RBAC 管理 | `http://10.210.65.24:3000/mcp/user_management` |
| **Security / Compliance** | 凭据与安全策略查看 / 管理 | `http://10.210.65.24:3000/mcp/security_compliance` |
| **Platform Configuration** | 系统设置、License、Execution Environment | `http://10.210.65.24:3000/mcp/platform_configuration` |

也可用 FQDN：`http://aap26.example.com:3000/mcp/job_management`（需 `/etc/hosts` 或 DNS 可解析）。

> **注意**：对 `:3000` 做简单 `curl` GET 可能返回 **404**，这不代表 MCP 不可用——MCP 客户端使用带 Bearer Token 的 **POST** 协议，与裸 `curl` 探测行为不同。**以 AI Agent 能否调用工具为准。**

---

## 3. 创建 API Token

在 AAP Gateway UI 中为 MCP 客户端创建 OAuth 2 Token。

### 3.1 操作步骤

1. 导航面板选择 **Access Management → Users**
2. 选择要用于 MCP 的**用户名**（通常为 `admin` 或专用服务账号）
3. 打开 **Tokens** 选项卡
4. 点击 **Create token**，填写：

| 字段 | DEMO 建议 |
| --- | --- |
| **Application** | 输入关联应用名，或 Browse 搜索；留空则创建 PAT（Personal Access Token） |
| **Description** | 可选，如 `cursor-mcp-demo` |
| **Scope** | **Write**（DEMO 需触发 Job）；只读场景选 **Read** |

5. 点击 **Create token**
6. 在令牌信息页点击 **Copy**，**立即保存**——令牌**仅显示一次**

> **权限继承**：Token 继承对应用户的 RBAC 权限。即使用户 Token 为 Write，若 inventory 中 `mcp_allow_write_operations=false`，MCP 服务端仍会拒绝写操作。

### 3.2 验证 Token

1. **Access Management → OAuth Applications**
2. 选择关联的 Application（若创建 PAT 可跳过）
3. **Tokens** 选项卡中应显示新创建的 Token

---

## 4. 在 AI Agent 中配置 MCP

### 4.1 Cursor（`mcp.json`）

在 Cursor MCP 配置中添加 AAP MCP Server。建议为每个 Toolset 注册独立 Server（名称宜 **≤ 20 字符**，避免与工具名组合后超过 AI 客户端 64 字符限制）：

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

> 将 `<YOUR_AAP_TOKEN>` 替换为 §3 创建的 Token。上表 URL 与 AIOps DEMO 现网 Cursor 配置一致。

**最小示例**（仅 Job Management，与现网 `mcp.json` 相同结构）：

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

在 [n8n 部署章节](../04-n8n/) 最后添加 MCP Client Tool 配置，Endpoint 使用 §2 表格中的 URL，Authentication 设为 **Bearer Token**，填入 §3 创建的 Token。

| 配置项 | 值 |
| --- | --- |
| **Transport** | HTTP |
| **URL** | 如 `http://10.210.65.24:3000/mcp/job_management` |
| **Authorization** | `Bearer <YOUR_AAP_TOKEN>` |

---

## 5. 验证与排错

### 5.1 功能验证

在 AI Agent 对话窗口输入：

```
What MCP tools are available for my Ansible Automation Platform?
```

或：

```
Give me a list of my Ansible Automation Platform jobs.
```

AI 应返回 MCP 工具列表或 Job 查询结果。

### 5.2 常见问题

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| `curl :3000` 返回 404 | 裸 GET 非 MCP 协议 | 正常；以 AI Agent 能否调用工具为准 |
| AI Agent 连不上 | URL / Token 错误 | 对照 §2.2；或尝试 RH 文档格式 `https://<FQDN>:8448/<toolset>/mcp` |
| 连接拒绝 | `ansiblemcp` 未运行 | `podman ps --filter name=ansiblemcp` |
| 401 / 403 | Token 无效或过期 | 重新创建 Token 并更新 Agent 配置 |
| 400 Bad Request | 自签证书校验失败 | inventory 设 `mcp_ignore_certificate_errors=true`，或配置正式证书 |
| 406 Not Acceptable | API 输出格式非 JSON | 提示 AI 先以 JSON 格式请求 |
| 写操作被拒 | 服务端只读 | 检查 `mcp_allow_write_operations` 与用户 Token Scope |

---

## 配置后检查清单

| # | 检查项 | 通过标准 |
| --- | --- | --- |
| 1 | MCP 容器 | `ansiblemcp` 容器 `Up` |
| 2 | API Token | 已创建并安全保存 |
| 3 | Cursor / n8n | MCP Server 配置已保存并启用 |
| 4 | 连通性 | AI 对话可列出 MCP 工具 |
| 5 | 权限 | 按 DEMO 需求可查询 Inventory / 触发 Job（Write Scope） |

---

## 参考文档

- [03-03 AAP Server 部署](03-03-AAP-Server-Deploy-CN.md) — `[ansiblemcp]` inventory 配置
- [03-04 AAP 配置](03-04-AAP-Configuration-CN.md)
- [Deploy Ansible MCP Server (AAP 2.6)](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/extend-assembly_deploying_ansible_mcp_server)
- [n8n MCP Client Tool 文档](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)

> **下一步**：[03-06 Automation Decisions 配置](03-06-Automation-Decisions-CN.md)
