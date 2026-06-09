# n8n Workflow Import

> **Status**: 20260609 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Description |
| --- | --- | --- |
| ① | **Import method** | Editor UI “Import from File” (recommended) |
| ② | **Alternatives** | Paste JSON on canvas / CLI `import:workflow` |
| ③ | **Workflow files** | 3 JSON files in the same directory |
| ④ | **Official docs** | [Export and import workflows](https://docs.n8n.io/workflows/export-import/) |

| File name |
| --- |
| `02_EDA_Trigger_Workflow_A_0519.json` |
| `02_EDA_Trigger_Workflow_B_0519.json` |
| `01_Chat_Triger_A_0519.json` |

---

## 1. Import N8N Workflow Files

> n8n stores workflows in JSON format and supports import via the Editor UI or CLI. See [n8n Docs — Export and import workflows](https://docs.n8n.io/workflows/export-import/).

### 1.1 Prerequisites

- n8n is deployed (see `08-01-n8n-Deploy-EN.md`).
- n8n Web UI is reachable in a browser (example in this environment: `http://10.210.65.103:8080` or your `N8N_HOST` URL).
- An admin account is created or you are logged in (set on first visit; example: `admin@example.com`).

### 1.2 Method 1: Editor UI — Import from File (Recommended)

Follow these steps to import **each** of the three JSON files listed in §1.4:

1. Open the n8n Web UI and sign in with an admin account.
2. Click **+** (or **Create Workflow**) on the left to create a new blank workflow.
3. In the workflow editor, open the **⋯** (three dots) menu in the **top-right** navigation bar.
4. Select **Import from File**.
5. In the file picker, locate the JSON file in this directory (e.g. `02_EDA_Trigger_Workflow_A_0519.json`) and click **Open**.
6. After nodes load on the canvas, verify the workflow name and nodes are complete.
7. Click **Save** in the top-right corner.
8. Repeat steps 2–7 for the remaining two JSON files.

**Other menu options (official docs):**

| Menu item | Description |
| --- | --- |
| **Import from File** | Import a workflow from a local JSON file |
| **Import from URL** | Import workflow JSON from a URL |
| **Download** | Export the current workflow as a JSON file |

### 1.3 Method 2: Copy-Paste on Canvas (Alternative)

1. Open the JSON file in a text editor, select all, and copy (`Ctrl + C` / `Cmd + C`).
2. Create a new blank workflow in n8n.
3. Paste on an empty area of the canvas (`Ctrl + V` / `Cmd + V`).
4. After nodes appear, click **Save**.

> Official docs: you can also select a subset of nodes on the canvas and copy/paste partial workflow fragments with `Ctrl + C` / `Ctrl + V`.

### 1.4 Workflow Files

Files are in the same directory:

```text
02_EDA_Trigger_Workflow_A_0519.json
02_EDA_Trigger_Workflow_B_0519.json
01_Chat_Triger_A_0519.json
```

### 1.5 Method 3: CLI Import (Alternative)

For batch import or automation. See [CLI commands — Import workflows](https://docs.n8n.io/hosting/cli-commands/).

**Import a single file:**

```bash
n8n import:workflow --input=file.json
```

**Import all `*.json` files from a directory:**

```bash
n8n import:workflow --separate --input=path/to/directory/
```

In this environment n8n runs in a Podman container. Copy JSON files to the mounted host path (`/opt/n8n/files`, mapped to `/tmp` in the container), then run:

```bash
podman exec -it n8n n8n import:workflow --input=/tmp/02_EDA_Trigger_Workflow_A_0519.json
```

> **CLI default behavior (official docs)**: `import:workflow` deactivates imported workflows by default. To preserve the `active` field from each JSON file, use `--activeState=fromJson` (multi-main & queue mode only).

### 1.6 Post-Import Notes

> Official docs — [Sharing credentials](https://docs.n8n.io/workflows/export-import/#sharing-credentials)

- Exported JSON includes **credential names and IDs**; after import, **re-bind or create credentials** for each node in the n8n UI, or nodes will not run correctly.
- HTTP Request nodes may contain auth headers; sanitize before sharing JSON files.
- After importing all three workflows and configuring credentials, **Activate** each workflow in the editor so triggers can invoke them.
