# n8n 工作流导入

> **状态**：20260609 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 说明 |
| --- | --- | --- |
| ① | **导入方式** | Editor UI「Import from File」（推荐） |
| ② | **备选方式** | 画布粘贴 JSON / CLI `import:workflow` |
| ③ | **Workflow 文件** | 同级目录下 3 个 JSON 文件 |
| ④ | **官方文档** | [Export and import workflows](https://docs.n8n.io/workflows/export-import/) |

| 文件名 |
| --- |
| `02_EDA_Trigger_Workflow_A_0519.json` |
| `02_EDA_Trigger_Workflow_B_0519.json` |
| `01_Chat_Triger_A_0519.json` |

---

## 一、导入 N8N Workflow 文件

> n8n 以 JSON 格式保存工作流，可通过 Editor UI 或 CLI 导入。详见 [n8n 官方文档 — Export and import workflows](https://docs.n8n.io/workflows/export-import/)。

### 1.1 前置条件

- 已完成 n8n 部署（参见 `08-01-n8n-Deploy-CN.md`）。
- 浏览器可访问 n8n Web UI（本环境示例：`http://10.210.65.103:8080` 或对应 `N8N_HOST` 地址）。
- 已创建或登录管理员账号（首次访问时设置，示例：`admin@example.com`）。

### 1.2 方式一：Editor UI — Import from File（推荐）

按以下步骤，**分别导入** §1.4 所列三个 JSON 文件：

1. 打开 n8n Web UI，使用管理员账号登录。
2. 点击左侧 **+**（或 **Create Workflow**），新建一个空白工作流（Blank Workflow）。
3. 进入工作流编辑器后，点击**右上角导航栏**的 **⋯**（三个点）菜单。
4. 选择 **Import from File**。
5. 在文件选择对话框中，定位到本目录下的 JSON 文件（例如 `02_EDA_Trigger_Workflow_A_0519.json`），点击 **Open**。
6. 工作流节点加载到画布后，检查工作流名称与节点是否完整。
7. 点击右上角 **Save** 保存工作流。
8. 对其余两个 JSON 文件重复步骤 2～7。

**菜单中其他相关选项（官方文档）：**

| 菜单项 | 说明 |
| --- | --- |
| **Import from File** | 从本地 JSON 文件导入工作流 |
| **Import from URL** | 从 URL 导入已发布的工作流 JSON |
| **Download** | 将当前工作流导出为 JSON 文件下载 |

### 1.3 方式二：画布 Copy-Paste（备选）

1. 用文本编辑器打开 JSON 文件，全选并复制全部内容（`Ctrl + C` / `Cmd + C`）。
2. 在 n8n 中新建空白工作流。
3. 在编辑器画布空白处粘贴（`Ctrl + V` / `Cmd + V`）。
4. 工作流节点出现后，点击 **Save** 保存。

> 官方文档说明：也可在画布上框选部分节点，用 `Ctrl + C` / `Ctrl + V` 复制粘贴工作流的局部片段。

### 1.4 Workflow 文件

file 在同级目录 下：

```text
02_EDA_Trigger_Workflow_A_0519.json
02_EDA_Trigger_Workflow_B_0519.json
01_Chat_Triger_A_0519.json
```

### 1.5 方式三：CLI 导入（备选）

适用于批量导入或自动化场景。官方命令参见 [CLI commands — Import workflows](https://docs.n8n.io/hosting/cli-commands/)。

**导入单个文件：**

```bash
n8n import:workflow --input=file.json
```

**从目录批量导入所有 `*.json`：**

```bash
n8n import:workflow --separate --input=path/to/directory/
```

本环境 n8n 运行于 Podman 容器，可在宿主机先将 JSON 文件放入已挂载目录（`/opt/n8n/files`，容器内对应 `/tmp`），再执行：

```bash
podman exec -it n8n n8n import:workflow --input=/tmp/02_EDA_Trigger_Workflow_A_0519.json
```

> **CLI 默认行为（官方文档）**：`import:workflow` 默认将导入的工作流设为**未激活**（inactive）。若需保留 JSON 中 `active` 字段的状态，可使用 `--activeState=fromJson`（仅 multi-main & queue 模式支持）。

### 1.6 导入后注意事项

> 官方文档 — [Sharing credentials](https://docs.n8n.io/workflows/export-import/#sharing-credentials)

- 导出的 JSON 文件中包含 **Credential 名称与 ID**；导入后需在 n8n UI 中为各节点**重新绑定或创建凭据**（Credentials），否则节点无法正常运行。
- HTTP Request 等节点若包含认证头信息，导入前应注意脱敏；共享 JSON 文件前请移除敏感信息。
- 三个 Workflow 导入并配置凭据后，在编辑器中分别 **Activate**（激活）工作流，使其可被 Trigger 正常调用。
