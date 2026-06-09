# EDA Rulebook and Action Artifacts (Use Case 1/2/3)

> **Status**: 20260609 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Description |
| --- | --- | --- |
| ① | Rulebooks | Use Case 1 (Process Hung) · Use Case 2 (Network Error) |
| ② | Action Playbooks | `01_EDA_action_linuxperformancealerts_UI.yml` · `02_EDA_action_network_down_UI_OK.yml` |
| ③ | AAP UI config | Rulebook Activation · Action Job Template (`webhook_payload`) |
| ④ | Other orchestration | Use Case 1/2 self-healing · Use Case 3 xsos diagnostic playbook |

> **Note:** YAML/Playbook bodies below are identical to the tested CN artifacts.

---

## 1. EDA Rulebook Development

### 1.1 Use Case 1: Process Hung Rulebook

Rulebook for Use Case 1 (Process Hung fault simulation):


```text
[root@aap26 rulebooks]# more 01_eda_rule_linuxperformancealerts_ui.yml
```
```yaml
---
# =============================================================================
# Rulebook (AAP UI / Event Stream 版本)
# -----------------------------------------------------------------------------
# 用途：在 AAP 2.6 UI 下作为 Rulebook Activation 运行；接收来自 AAP
#       Event Stream 转发过来的 Prometheus Alertmanager Webhook Payload。
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
        # -------------------------------------------------------------------
        # Step 1: 打印整份 event.payload，便于在 EDA Activation 的
        #         Event 详情里直接查看 Alertmanager 推送过来的原始数据
        # -------------------------------------------------------------------
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

        # -------------------------------------------------------------------
        # Step 2: 启动一个 Job Template，把整份 event.payload
        #         以 extra_vars.webhook_payload 的形式传入 Playbook。
        # -------------------------------------------------------------------
        - run_job_template:
            name: "01_EDA_action_linuxperformancealerts_UI"
            organization: "Default"
            job_args:
              limit: "{{ event.payload.commonLabels.instance_name }}"
              extra_vars:
                webhook_payload: "{{ event.payload }}"
```

### 1.2 Use Case 2: Network Error Rulebook

Rulebook for Use Case 2 (Network Error fault simulation):


```text
[root@aap26 rulebooks]# more 02_eda_rule_linux_network_down.yml
```
```yaml
---
# =============================================================================
# Rulebook (AAP UI / Event Stream 版本)
# -----------------------------------------------------------------------------
# 用途：在 AAP 2.6 UI 下作为 Rulebook Activation 运行；接收来自 AAP
#       Event Stream 转发过来的 Prometheus Alertmanager Webhook Payload，
#       匹配「网络接口异常」类告警，然后调用 Job Template 启动修复 Playbook。
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
        # -------------------------------------------------------------------
        # Step 1: 打印整份 event.payload，便于在 EDA Activation 的
        #         Event 详情里直接查看 Alertmanager 推送过来的原始数据
        # -------------------------------------------------------------------
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

        # -------------------------------------------------------------------
        # Step 2: 启动一个 Job Template，把整份 event.payload
        #         以 extra_vars.webhook_payload 的形式传入 Playbook。
        #
        #         注意：
        #         - job_template 必须在 AAP UI 中预先创建好，
        #           并指定 Inventory（例如包含 chaos.example.com 的 inventory）
        #           和 Project，Playbook 选用：
        #             02_EDA_action_network_down_UI_OK.yml
        #         - limit 用 event.payload.commonLabels.instance_name
        #           将动作限定到真正发生告警的目标主机上（例如
        #           chaos.example.com）。
        # -------------------------------------------------------------------
        - run_job_template:
            name: "02_EDA_action_network_down_UI"
            organization: "Default"
            job_args:
              limit: "{{ event.payload.commonLabels.instance_name }}"
              extra_vars:
                webhook_payload: "{{ event.payload }}"
```

---

## 2. Rulebook Activations Configuration

### 2.1 Use Case 1 Rulebook Activation

<put image here>

### 2.2 Use Case 2 Rulebook Activation

<put image here>



---

## 3. EDA Rulebook Action Playbook Development

### 3.1 Use Case 1 Action Playbook


```text
[admin@aap26 AIOPS]$ more 01_EDA_action_linuxperformancealerts_UI.yml
```
```yaml
---
# =============================================================================
# Playbook (AAP UI / Job Template 版本)
# -----------------------------------------------------------------------------
# 用途：作为 EDA Rulebook (01_eda_rule_linuxperformancealerts_ui.yml)
#       触发的 Job Template 对应的 Playbook，完成下列工作：
# =============================================================================

- name: "Linux Performance Analysis for High CPU Usage"
  hosts: all
  become: true

  vars:
    perf_duration: 15
    output_base: "/TS"
    diagnose_server: "10.210.65.14"
    n8n_webhook_url: "http://10.210.65.103:8080/webhook/9a1d6d25-2a9b-4906-afd7-21c3b4a49860"

    # -------------------------------------------------------------------------
    # 需求 2：EDA rulebook 通过 job_args.extra_vars.webhook_payload
    # 把 event.payload 透传进来。这里在 vars 中给出空默认值，使本 Playbook
    # 在手工运行（未带 extra_vars）时也不会因变量缺失而失败。
    # 通过 extra_vars 传入的同名变量优先级最高，会覆盖此处的默认值。
    # -------------------------------------------------------------------------
    webhook_payload: {}

    # 从 webhook_payload 抽取常用字段（带兜底默认值）
    alert_name: >-
      {{ (webhook_payload.groupLabels | default({})).alertname
         | default((webhook_payload.commonLabels | default({})).alertname
         | default('UnknownAlert')) }}
    alert_instance: >-
      {{ (webhook_payload.commonLabels | default({})).instance
         | default((webhook_payload.groupLabels | default({})).instance
         | default('UnknownInstance')) }}
    alert_severity: >-
      {{ (webhook_payload.commonLabels | default({})).severity | default('Unknown') }}
    alert_status: "{{ webhook_payload.status | default('Unknown') }}"
    alert_receiver: "{{ webhook_payload.receiver | default('Unknown') }}"

    timestamp_suffix: "{{ ansible_date_time.date }}-{{ ansible_date_time.hour }}-{{ ansible_date_time.minute }}"

    alert_info_file: "/tmp/{{ inventory_hostname }}-{{ alert_name }}-alert_info.txt"

  tasks:
    # ===========================================
    # Phase 0：从 webhook_payload 生成 alert_info 文件
    # -------------------------------------------------
    # ===========================================
    - name: "Show received webhook_payload (from EDA extra_vars)"
      ansible.builtin.debug:
        msg: |
          ===== Webhook payload received from EDA Rulebook =====
          alert_name     : {{ alert_name }}
          alert_instance : {{ alert_instance }}
          severity       : {{ alert_severity }}
          status         : {{ alert_status }}
          receiver       : {{ alert_receiver }}
          ------ full payload ------
          {{ webhook_payload | to_nice_yaml }}

    - name: "Write alert_info file on target host from webhook_payload"
      ansible.builtin.copy:
        dest: "{{ alert_info_file }}"
        mode: "0644"
        content: |
          === ALERT INFORMATION (from AAP Event Stream) ===
          Alert Name : {{ alert_name }}
          Instance   : {{ alert_instance }}
          Severity   : {{ alert_severity }}
          Status     : {{ alert_status }}
          Receiver   : {{ alert_receiver }}

          === FULL EVENT PAYLOAD ===
          {{ webhook_payload | to_nice_yaml }}

    # ===========================================
    # Phase 1：环境准备和工具检查
    # ===========================================
    - name: "Check required tools availability"
      ansible.builtin.command: "which {{ item }}"
      loop: [perf, pidstat]
      register: tool_check
      ignore_errors: yes
      changed_when: false

    - name: "Install required packages if missing"
      ansible.builtin.dnf:
        name:
          - perf
          - sysstat
          - kernel-debuginfo
          - kernel-debuginfo-common
        state: present
      when: tool_check.results | selectattr('rc', 'ne', 0) | list | length > 0
      register: package_install
      ignore_errors: yes

    # ===========================================
    # Phase 2：系统性能数据收集
    # ===========================================
    - name: "Create timestamped output directory"
      ansible.builtin.file:
        path: "{{ output_base }}/{{ inventory_hostname }}-{{ timestamp_suffix }}"
        state: directory
        mode: '0755'
      register: output_dir

    - name: "Capture system load snapshot with top (limited to 30 lines)"
      ansible.builtin.shell: "top -b -n 1 | head -n 30"
      register: top_output
      changed_when: false

    - name: "Record CPU usage statistics with pidstat (sorted by CPU usage, limited to 30 lines)"
      ansible.builtin.shell: |
        pidstat -u -p ALL 1 5 | \
        awk 'NR<=3 || ($8 != "" && $8 != "0.00" && $8 != "CPU") {print}' | \
        sort -k8 -nr | \
        head -n 30
      register: pidstat_output
      changed_when: false

    - name: "Find processes with CPU usage greater than 90%"
      ansible.builtin.shell: |
        ps -eo pid,%cpu --sort=-%cpu | awk '$2 > 90 {print $1}'
      register: high_cpu_processes
      changed_when: false

    - name: "Fail if no processes with CPU usage greater than 90% found"
      ansible.builtin.fail:
        msg: "No process found with CPU usage > 90%"
      when: high_cpu_processes.stdout == ""

    - name: "Get the first process ID with high CPU usage"
      ansible.builtin.set_fact:
        pid: "{{ high_cpu_processes.stdout_lines[0] }}"

    - name: "Display target process info"
      ansible.builtin.debug:
        msg: "Targeting PID {{ pid }} for perf recording"

    # ===========================================
    # Phase 3：perf 性能分析数据收集
    # ===========================================
    - name: "Combine top and pidstat output into single file"
      ansible.builtin.copy:
        content: |
          === TOP COMMAND OUTPUT  ===
          {{ top_output.stdout }}

          === PIDSTAT COMMAND OUTPUT (Sorted by CPU usage) ===
          {{ pidstat_output.stdout }}

          === TARGET PROCESS FOR PERF ===
          PID: {{ pid }}
        dest: "{{ output_dir.path }}/system_analysis.txt"
        mode: '0644'

    - name: "Record performance data using perf for the process (with debug symbols)"
      ansible.builtin.shell: |
        echo "Starting perf record for PID {{ pid }}"
        perf record -p {{ pid }} -a -g --call-graph dwarf -- sleep {{ perf_duration }} > {{ output_dir.path }}/perf_output.log 2>&1
      args:
        chdir: "{{ output_dir.path }}"
      register: perf_record_output
      ignore_errors: yes

    - name: "Verify perf.data creation"
      ansible.builtin.stat:
        path: "{{ output_dir.path }}/perf.data"
      register: perf_data_check

    - name: "Check for perf.data in other locations if not found"
      ansible.builtin.find:
        paths: "{{ output_dir.path }}"
        patterns: "perf.data*"
        recurse: yes
      register: perf_data_search
      when: not perf_data_check.stat.exists

    - name: "Display perf execution results and debugging info"
      ansible.builtin.debug:
        msg: |
          perf command execution details:
          - Return code: {{ perf_record_output.rc | default('N/A') }}
          - stdout: {{ perf_record_output.stdout | default('N/A') }}
          - stderr: {{ perf_record_output.stderr | default('N/A') }}
          - perf.data exists: {{ perf_data_check.stat.exists }}
          - perf.data search results: {{ perf_data_search.files | default([]) | map(attribute='path') | list }}
          - perf output log: {{ output_dir.path }}/perf_output.log

    - name: "Display collection summary"
      ansible.builtin.debug:
        msg: |
          Performance data collection completed:
          - Output directory: {{ output_dir.path }}
          - System analysis file: {{ output_dir.path }}/system_analysis.txt
          - perf.data created: {{ perf_data_check.stat.exists }}
          - perf.data size: {{ perf_data_check.stat.size | default('N/A') }} bytes
          - perf.data location: {{ output_dir.path }}/perf.data
          - perf output log: {{ output_dir.path }}/perf_output.log

    # ===========================================
    # Phase 4：诊断服务器连接和目录准备
    # ===========================================
    - name: "Set Triage Server variables"
      ansible.builtin.set_fact:
        diagnose_path: "/diagnose/{{ inventory_hostname }}-{{ timestamp_suffix }}"
        diagnose_full_path: "/diagnose/{{ inventory_hostname }}-{{ timestamp_suffix }}/"

    - name: "Test connection to Triage Server"
      ansible.builtin.ping:
      delegate_to: "{{ diagnose_server }}"
      register: diagnose_ping_result
      ignore_errors: yes

    - name: "Display Triage Server connection status"
      ansible.builtin.debug:
        msg: "Triage Server connection: {{ 'SUCCESS' if diagnose_ping_result is succeeded else 'FAILED' }}"

    - name: "Create diagnose directory on Triage Server"
      ansible.builtin.file:
        path: "{{ diagnose_path }}"
        state: directory
        mode: '0755'
        owner: root
        group: root
      delegate_to: "{{ diagnose_server }}"
      register: diagnose_dir_result
      ignore_errors: yes

    - name: "Display diagnose directory creation result"
      ansible.builtin.debug:
        msg: "Diagnose directory creation: {{ 'SUCCESS' if diagnose_dir_result is succeeded else 'FAILED' }} - {{ diagnose_dir_result.msg | default('No message') }}"

    - name: "Verify diagnose directory exists"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}"
      delegate_to: "{{ diagnose_server }}"
      register: diagnose_dir_check
      ignore_errors: yes

    - name: "Display directory verification result"
      ansible.builtin.debug:
        msg: "Directory exists: {{ diagnose_dir_check.stat.exists | default(false) }}"

    - name: "Ensure Triage Server host key is in known_hosts (source host)"
      ansible.builtin.known_hosts:
        name: "{{ diagnose_server }}"
        key: "{{ lookup('pipe', 'ssh-keyscan -T 5 -H ' ~ diagnose_server) }}"
        path: "/root/.ssh/known_hosts"

    # ===========================================
    # Phase 5：性能数据文件传输到诊断服务器
    # ===========================================
    - name: "Verify source directory exists before transfer"
      ansible.builtin.stat:
        path: "{{ output_dir.path }}"
      register: source_dir_check

    - name: "Display source directory verification"
      ansible.builtin.debug:
        msg: |
          Source directory check:
          - Path: {{ output_dir.path }}
          - Exists: {{ source_dir_check.stat.exists }}
          - Is directory: {{ source_dir_check.stat.isdir | default(false) }}

    - name: "List source directory contents"
      ansible.builtin.find:
        paths: "{{ output_dir.path }}"
        patterns: "*"
        recurse: yes
      register: source_files

    - name: "Display source files"
      ansible.builtin.debug:
        msg: |
          Source files to transfer:
          {% for file in source_files.files | default([]) %}
          - {{ file.path }} ({{ file.size | default(0) }} bytes)
          {% endfor %}

    - name: "Verify source directory exists on fault node"
      ansible.builtin.command: "ls -ld {{ output_dir.path }}"
      register: verify_ts
      ignore_errors: yes

    - name: "Display directory verification result"
      ansible.builtin.debug:
        msg: |
          Directory verification on fault node:
          - Command: ls -ld {{ output_dir.path }}
          - Return code: {{ verify_ts.rc | default('N/A') }}
          - Output: {{ verify_ts.stdout | default('No output') }}
          - Error: {{ verify_ts.stderr | default('No error') }}

    - name: "Transfer collected files to Triage Server using synchronize (push)"
      ansible.posix.synchronize:
        src: "{{ output_dir.path }}/"
        dest: "root@{{ diagnose_server }}:{{ diagnose_full_path }}"
        recursive: yes
        mode: push
      delegate_to: "{{ inventory_hostname }}"
      register: sync_result
      ignore_errors: yes
      when: source_dir_check.stat.exists and source_dir_check.stat.isdir

    - name: "Display synchronize transfer result"
      ansible.builtin.debug:
        msg: |
          synchronize transfer result:
          - Status: {{ 'SUCCESS' if sync_result is succeeded else 'FAILED' }}
          - Rc: {{ sync_result.rc | default('N/A') }}
          - Stdout: {{ sync_result.stdout | default('') }}
          - Stderr: {{ sync_result.stderr | default('') }}

    - name: "Verify files were transferred successfully"
      ansible.builtin.find:
        paths: "{{ diagnose_path }}"
        patterns: "*"
        recurse: yes
      delegate_to: "{{ diagnose_server }}"
      register: transferred_files
      ignore_errors: yes

    - name: "Display transferred files"
      ansible.builtin.debug:
        msg: |
          Transferred files on Triage Server:
          {% for file in transferred_files.files | default([]) %}
          - {{ file.path }} ({{ file.size | default(0) }} bytes)
          {% endfor %}

    # ===========================================
    # Phase 6：perf 报告生成和分析
    # ===========================================
    - name: "Generate perf report on Triage Server"
      ansible.builtin.shell: |
        cd {{ diagnose_path }}
        perf report -g --stdio -i perf.data | head -100 > {{ inventory_hostname }}-{{ ansible_date_time.date }}_perf_01.txt
      args:
        executable: /bin/bash
      delegate_to: "{{ diagnose_server }}"
      register: perf_report_result
      ignore_errors: yes

    - name: "Display perf report generation result"
      ansible.builtin.debug:
        msg: |
          Perf report executed on Triage Server:
          - Command return code: {{ perf_report_result.rc | default('N/A') }}
          - Stdout: {{ perf_report_result.stdout | default('') }}
          - Stderr: {{ perf_report_result.stderr | default('') }}
          - Output file: {{ diagnose_path }}/{{ inventory_hostname }}-{{ ansible_date_time.date }}_perf_01.txt

    - name: "Check if perf report output file exists"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}/{{ inventory_hostname }}-{{ ansible_date_time.date }}_perf_01.txt"
      delegate_to: "{{ diagnose_server }}"
      register: perf_report_file

    - name: "Remove original perf.data if report exists"
      ansible.builtin.file:
        path: "{{ diagnose_path }}/perf.data"
        state: absent
      delegate_to: "{{ diagnose_server }}"
      when: perf_report_file.stat.exists
      register: remove_perf_data

    - name: "Display perf.data cleanup result"
      ansible.builtin.debug:
        msg: |
          perf.data cleanup:
          - Report file exists: {{ perf_report_file.stat.exists }}
          - perf.data removed: {{ remove_perf_data is succeeded }}

    # ===========================================
    # Phase 7：Alert 信息文件处理
    # -------------------------------------------------
    # 需求 4：本阶段查找的就是 Phase 0 中新生成的
    #         /tmp/<host>-<alertname>-alert_info.txt 文件。
    # ===========================================
    - name: "Find alert info files for this host IP"
      ansible.builtin.find:
        paths: "/tmp"
        patterns: "{{ inventory_hostname }}-*-alert_info.txt"
        recurse: no
      register: alert_files

    - name: "Display found alert files"
      ansible.builtin.debug:
        msg: |
          Found alert files:
          {% for file in alert_files.files %}
          - {{ file.path }} ({{ file.mtime }})
          {% endfor %}

    - name: "Get the latest alert file"
      ansible.builtin.set_fact:
        latest_alert_file: "{{ alert_files.files | sort(attribute='mtime') | last }}"
      when: alert_files.files | length > 0

    - name: "Display latest alert file"
      ansible.builtin.debug:
        msg: "Latest alert file: {{ latest_alert_file.path }} ({{ latest_alert_file.mtime }})"
      when: latest_alert_file is defined

    - name: "Transfer latest alert file to Triage Server"
      ansible.posix.synchronize:
        src: "{{ latest_alert_file.path }}"
        dest: "root@{{ diagnose_server }}:{{ diagnose_path }}/"
        mode: push
      delegate_to: "{{ inventory_hostname }}"
      register: alert_transfer_result
      ignore_errors: yes
      when: latest_alert_file is defined

    - name: "Display alert file transfer result"
      ansible.builtin.debug:
        msg: |
          Alert file transfer result:
          - Status: {{ 'SUCCESS' if alert_transfer_result is succeeded else 'FAILED' }}
          - Rc: {{ alert_transfer_result.rc | default('N/A') }}
          - Stdout: {{ alert_transfer_result.stdout | default('') }}
          - Stderr: {{ alert_transfer_result.stderr | default('') }}
          - Transferred file: {{ latest_alert_file.path | default('N/A') }}
      when: latest_alert_file is defined

    - name: "Verify alert file was transferred successfully"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}/{{ latest_alert_file.path | basename }}"
      delegate_to: "{{ diagnose_server }}"
      register: alert_file_check
      ignore_errors: yes
      when: latest_alert_file is defined

    - name: "Display alert file verification result"
      ansible.builtin.debug:
        msg: |
          Alert file verification:
          - File exists on Triage Server: {{ alert_file_check.stat.exists | default(false) }}
          - File size: {{ alert_file_check.stat.size | default('N/A') }} bytes
          - File path: {{ diagnose_path }}/{{ latest_alert_file.path | basename | default('N/A') }}
      when: latest_alert_file is defined

    # ===========================================
    # Phase 8：文件合并和最终报告生成
    # ===========================================
    - name: "Merge all files into issues_latest.txt on Triage Server"
      ansible.builtin.shell: |
        cd {{ diagnose_path }}
        ALERT_FILE="{{ latest_alert_file.path | basename | default('') }}"

        if [ -n "$ALERT_FILE" ] && [ -f "$ALERT_FILE" ]; then
          echo "=== ALERT INFO ===" > issues_latest.txt
          cat "$ALERT_FILE" >> issues_latest.txt
          echo "" >> issues_latest.txt
          echo "=== OTHER DIAGNOSTIC FILES ===" >> issues_latest.txt
          echo "" >> issues_latest.txt
        else
          echo "=== DIAGNOSTIC FILES ===" > issues_latest.txt
          echo "" >> issues_latest.txt
        fi

        find . -maxdepth 1 -type f \( -name "*.txt" -o -name "*.log" -o -name "*.data" \) \
          ! -name "issues_latest.txt" ! -name "$ALERT_FILE" -print0 |
        while IFS= read -r -d '' file; do
          echo "=== ${file#./} ===" >> issues_latest.txt
          cat "$file" >> issues_latest.txt
          echo "" >> issues_latest.txt
          echo "" >> issues_latest.txt
        done

        echo "Files merged into issues_latest.txt:"
        echo "Alert file used: $ALERT_FILE"
        ls -la issues_latest.txt
      args:
        executable: /bin/bash
      delegate_to: "{{ diagnose_server }}"
      register: merge_result
      ignore_errors: yes

    - name: "Display merge result"
      ansible.builtin.debug:
        msg: |
          File merge completed:
          - Return code: {{ merge_result.rc | default('N/A') }}
          - Stdout: {{ merge_result.stdout | default('') }}
          - Stderr: {{ merge_result.stderr | default('') }}
          - Merged file: {{ diagnose_path }}/issues_latest.txt

    - name: "Verify merged file exists"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}/issues_latest.txt"
      delegate_to: "{{ diagnose_server }}"
      register: merged_file_stat

    - name: "Display merged file information"
      ansible.builtin.debug:
        msg: |
          Merged file verification:
          - File exists: {{ merged_file_stat.stat.exists }}
          - File size: {{ merged_file_stat.stat.size | default(0) }} bytes
          - File path: {{ diagnose_path }}/issues_latest.txt

    - name: "Notify Triage Server directory location"
      ansible.builtin.debug:
        msg: "Data transferred to Triage Server: {{ diagnose_path }}"

    - name: "Pass diagnose context to next play via dummy host"
      ansible.builtin.add_host:
        name: "_eda_n8n_ctx"
        groups: "eda_n8n_ctx"
        diagnose_path: "{{ diagnose_path }}"
        target_host: "{{ inventory_hostname }}"
        alert_name: "{{ alert_name }}"
        alert_instance: "{{ alert_instance }}"
        alert_severity: "{{ alert_severity }}"
        alert_status: "{{ alert_status }}"
        alert_receiver: "{{ alert_receiver }}"
        webhook_payload: "{{ webhook_payload }}"


# =============================================================================
# 最后一个 Play：调用 n8n_webhook_url 激活 n8n 工作流
# -----------------------------------------------------------------------------
# 需求 1：CMD 版本 rulebook 里的 run_module: uri 在 AAP UI 下因为没有
# inventory context 会失败，所以把这一动作迁移到本 Playbook 最后一个 play。
# 该 play 在 AAP controller (localhost) 上执行，不依赖故障节点。
# =============================================================================
- name: "Notify n8n workflow to start fault-diagnosis pipeline"
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    n8n_webhook_url: "http://10.210.65.103:8080/webhook/9a1d6d25-2a9b-4906-afd7-21c3b4a49860"
    ctx: "{{ hostvars['_eda_n8n_ctx'] | default({}) }}"

  tasks:
    - name: "Display n8n notification context"
      ansible.builtin.debug:
        msg: |
          About to notify n8n workflow:
          - URL            : {{ n8n_webhook_url }}
          - Target host    : {{ ctx.target_host | default('Unknown') }}
          - Alert name     : {{ ctx.alert_name | default('Unknown') }}
          - Alert instance : {{ ctx.alert_instance | default('Unknown') }}
          - Severity       : {{ ctx.alert_severity | default('Unknown') }}
          - Diagnose path  : {{ ctx.diagnose_path | default('Unknown') }}

    - name: "POST to n8n webhook to trigger fault-diagnosis workflow"
      ansible.builtin.uri:
        url: "{{ n8n_webhook_url }}"
        method: POST
        body_format: json
        headers:
          Content-Type: "application/json"
        body:
          alert_instance: "{{ ctx.alert_instance | default('Unknown') }}"
          alert_name: "{{ ctx.alert_name | default('Unknown') }}"
          severity: "{{ ctx.alert_severity | default('Unknown') }}"
          status: "{{ ctx.alert_status | default('Unknown') }}"
          receiver: "{{ ctx.alert_receiver | default('Unknown') }}"
          target_host: "{{ ctx.target_host | default('Unknown') }}"
          diagnose_path: "{{ ctx.diagnose_path | default('Unknown') }}"
          issues_latest: "{{ ctx.diagnose_path | default('') }}/issues_latest.txt"
          webhook_payload: "{{ ctx.webhook_payload | default({}) }}"
        status_code: [200, 201, 202, 204]
        timeout: 15
      register: n8n_result
      ignore_errors: yes

    - name: "Display n8n webhook invocation result"
      ansible.builtin.debug:
        msg: |
          n8n webhook invocation result:
          - Status     : {{ 'SUCCESS' if n8n_result is succeeded else 'FAILED' }}
          - HTTP code  : {{ n8n_result.status | default('N/A') }}
          - Response   : {{ n8n_result.json | default(n8n_result.content | default('')) }}
          - Error msg  : {{ n8n_result.msg | default('') }}

```

### 3.2 Use Case 2 Action Playbook


```text
[admin@aap26 AIOPS]$ more 02_EDA_action_network_down_UI_OK.yml
```
```yaml
---
# =============================================================================
# Playbook (AAP UI / Job Template 版本)
# -----------------------------------------------------------------------------
# 用途：作为 EDA Rulebook (02_eda_rule_linux_network_down.yml)
#       触发的 Job Template 对应的 Playbook，针对「网络接口 DOWN」类告警，
#       仅完成「故障数据采集」工作，不做任何自动修复。
# =============================================================================

- name: "Linux Network Interface Down — Diagnose & Collect Only"
  hosts: all
  become: true

  vars:
    output_base: "/TS"
    diagnose_server: "10.210.65.14"
    n8n_webhook_url: "http://10.210.65.103:8080/webhook/9a1d6d25-2a9b-4906-afd7-21c3b4a49860"

    # -------------------------------------------------------------------------
    # EDA rulebook 通过 job_args.extra_vars.webhook_payload 透传 event.payload。
    # 这里给出空默认值，使本 Playbook 在手工运行（未带 extra_vars）时也不会报错。
    # extra_vars 中同名变量优先级最高，会覆盖此处默认值。
    # -------------------------------------------------------------------------
    webhook_payload: {}

    alert_name: >-
      {{ (webhook_payload.groupLabels | default({})).alertname
         | default((webhook_payload.commonLabels | default({})).alertname
         | default('NetworkInterfaceDown')) }}
    alert_instance: >-
      {{ (webhook_payload.commonLabels | default({})).instance
         | default((webhook_payload.groupLabels | default({})).instance
         | default('UnknownInstance')) }}
    alert_severity: >-
      {{ (webhook_payload.commonLabels | default({})).severity | default('critical') }}
    alert_status: "{{ webhook_payload.status | default('firing') }}"
    alert_receiver: "{{ webhook_payload.receiver | default('Unknown') }}"

    # 故障网卡名（首选 commonLabels.device，例如 ens37）；
    # 若 payload 未提供，则在 Phase 1 自动扫描 state DOWN 的物理网卡。
    alert_device_from_payload: >-
      {{ (webhook_payload.commonLabels | default({})).device | default('') }}

    timestamp_suffix: "{{ ansible_date_time.date }}-{{ ansible_date_time.hour }}-{{ ansible_date_time.minute }}"
    alert_info_file: "/tmp/{{ inventory_hostname }}-{{ alert_name }}-alert_info.txt"

  tasks:
    # ===========================================
    # Phase 0：从 webhook_payload 生成 alert_info 文件
    # ===========================================
    - name: "Show received webhook_payload (from EDA extra_vars)"
      ansible.builtin.debug:
        msg: |
          ===== Webhook payload received from EDA Rulebook =====
          alert_name             : {{ alert_name }}
          alert_instance         : {{ alert_instance }}
          alert_device (payload) : {{ alert_device_from_payload | default('(none)') }}
          severity               : {{ alert_severity }}
          status                 : {{ alert_status }}
          receiver               : {{ alert_receiver }}
          ------ full payload ------
          {{ webhook_payload | to_nice_yaml }}

    - name: "Write alert_info file on target host from webhook_payload"
      ansible.builtin.copy:
        dest: "{{ alert_info_file }}"
        mode: "0644"
        content: |
          === ALERT INFORMATION (from AAP Event Stream) ===
          Alert Name : {{ alert_name }}
          Instance   : {{ alert_instance }}
          Device     : {{ alert_device_from_payload | default('(none)') }}
          Severity   : {{ alert_severity }}
          Status     : {{ alert_status }}
          Receiver   : {{ alert_receiver }}

          === FULL EVENT PAYLOAD ===
          {{ webhook_payload | to_nice_yaml }}

    # ===========================================
    # Phase 1：识别故障网卡
    # ===========================================
    - name: "Create timestamped output directory"
      ansible.builtin.file:
        path: "{{ output_base }}/{{ inventory_hostname }}-{{ timestamp_suffix }}"
        state: directory
        mode: '0755'
      register: output_dir

    - name: "Scan for any DOWN physical interfaces (fallback when payload has no device)"
      ansible.builtin.shell: |
        set -o pipefail
        ip -o link show \
          | awk -F': ' '{print $2" "$0}' \
          | grep -v -E '^(lo|veth|docker|br-|virbr|cni|cilium|kube)' \
          | awk '/state DOWN/ {print $1}' \
          | head -n 1
      args:
        executable: /bin/bash
      register: down_iface_scan
      changed_when: false

    - name: "Decide target device"
      ansible.builtin.set_fact:
        target_device: >-
          {{ alert_device_from_payload
             if (alert_device_from_payload | length > 0)
             else (down_iface_scan.stdout | trim) }}

    - name: "Fail if no target device can be determined"
      ansible.builtin.fail:
        msg: >-
          Neither webhook_payload.commonLabels.device nor ip link scan
          could determine a DOWN interface on {{ inventory_hostname }}.
      when: target_device | length == 0

    - name: "Display target device"
      ansible.builtin.debug:
        msg: "Target down interface = {{ target_device }}"

    # ===========================================
    # Phase 2：收集网络诊断数据（仅采集，不修复）
    # ===========================================
    - name: "Collect basic IP / link / route information"
      ansible.builtin.shell: |
        echo "===== ip a =====";          ip a
        echo "===== ip link =====";       ip link
        echo "===== ip route =====";      ip route
        echo "===== ip -s link show {{ target_device }} ====="; ip -s link show {{ target_device }} || true
      args:
        executable: /bin/bash
      register: net_basic
      changed_when: false

    - name: "Check NetworkManager presence"
      ansible.builtin.command: "which nmcli"
      register: nmcli_check
      ignore_errors: yes
      changed_when: false

    - name: "Collect nmcli status (if available)"
      ansible.builtin.shell: |
        echo "===== nmcli general status ====="; nmcli general status || true
        echo "===== nmcli device status =====";  nmcli device status || true
        echo "===== nmcli connection show ====="; nmcli connection show || true
        echo "===== nmcli device show {{ target_device }} ====="; nmcli device show {{ target_device }} || true
      args:
        executable: /bin/bash
      register: nmcli_info
      when: nmcli_check.rc == 0
      changed_when: false

    - name: "Collect ethtool / driver / link info"
      ansible.builtin.shell: |
        echo "===== ethtool {{ target_device }} ====="; ethtool {{ target_device }} 2>/dev/null || echo "(ethtool not available)"
        echo "===== ethtool -i {{ target_device }} ====="; ethtool -i {{ target_device }} 2>/dev/null || true
        echo "===== sysfs carrier =====";   cat /sys/class/net/{{ target_device }}/carrier 2>/dev/null || true
        echo "===== sysfs operstate ====="; cat /sys/class/net/{{ target_device }}/operstate 2>/dev/null || true
      args:
        executable: /bin/bash
      register: ethtool_info
      changed_when: false

    - name: "Collect NetworkManager / kernel recent logs"
      ansible.builtin.shell: |
        echo "===== journalctl -u NetworkManager (tail 80) ====="
        journalctl -u NetworkManager -n 80 --no-pager 2>/dev/null || true
        echo
        echo "===== dmesg | grep {{ target_device }} (tail 40) ====="
        dmesg --ctime 2>/dev/null | grep -i "{{ target_device }}" | tail -n 40 || true
      args:
        executable: /bin/bash
      register: net_logs
      changed_when: false

    - name: "Write net_diag.txt"
      ansible.builtin.copy:
        dest: "{{ output_dir.path }}/net_diag.txt"
        mode: "0644"
        content: |
          === NETWORK DIAGNOSTIC — {{ target_device }} on {{ inventory_hostname }} ===
          collected at : {{ ansible_date_time.iso8601 }}

          {{ net_basic.stdout }}

          {{ nmcli_info.stdout | default('(nmcli not available)') }}

          {{ ethtool_info.stdout }}

          {{ net_logs.stdout }}

    - name: "Display collection summary"
      ansible.builtin.debug:
        msg: |
          Network data collection completed:
          - Output directory : {{ output_dir.path }}
          - net_diag.txt     : {{ output_dir.path }}/net_diag.txt
          - target_device    : {{ target_device }}

    # ===========================================
    # Phase 3：诊断服务器连接和目录准备
    # ===========================================
    - name: "Set Triage Server variables"
      ansible.builtin.set_fact:
        diagnose_path: "/diagnose/{{ inventory_hostname }}-{{ timestamp_suffix }}"
        diagnose_full_path: "/diagnose/{{ inventory_hostname }}-{{ timestamp_suffix }}/"

    - name: "Test connection to Triage Server"
      ansible.builtin.ping:
      delegate_to: "{{ diagnose_server }}"
      register: diagnose_ping_result
      ignore_errors: yes

    - name: "Display Triage Server connection status"
      ansible.builtin.debug:
        msg: "Triage Server connection: {{ 'SUCCESS' if diagnose_ping_result is succeeded else 'FAILED' }}"

    - name: "Create diagnose directory on Triage Server"
      ansible.builtin.file:
        path: "{{ diagnose_path }}"
        state: directory
        mode: '0755'
        owner: root
        group: root
      delegate_to: "{{ diagnose_server }}"
      register: diagnose_dir_result
      ignore_errors: yes

    - name: "Display diagnose directory creation result"
      ansible.builtin.debug:
        msg: "Diagnose directory creation: {{ 'SUCCESS' if diagnose_dir_result is succeeded else 'FAILED' }} - {{ diagnose_dir_result.msg | default('No message') }}"

    - name: "Verify diagnose directory exists"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}"
      delegate_to: "{{ diagnose_server }}"
      register: diagnose_dir_check
      ignore_errors: yes

    - name: "Display directory verification result"
      ansible.builtin.debug:
        msg: "Directory exists: {{ diagnose_dir_check.stat.exists | default(false) }}"

    - name: "Ensure Triage Server host key is in known_hosts (source host)"
      ansible.builtin.known_hosts:
        name: "{{ diagnose_server }}"
        key: "{{ lookup('pipe', 'ssh-keyscan -T 5 -H ' ~ diagnose_server) }}"
        path: "/root/.ssh/known_hosts"

    # ===========================================
    # Phase 4：诊断文件传输到诊断服务器
    # ===========================================
    - name: "Verify source directory exists before transfer"
      ansible.builtin.stat:
        path: "{{ output_dir.path }}"
      register: source_dir_check

    - name: "Display source directory verification"
      ansible.builtin.debug:
        msg: |
          Source directory check:
          - Path: {{ output_dir.path }}
          - Exists: {{ source_dir_check.stat.exists }}
          - Is directory: {{ source_dir_check.stat.isdir | default(false) }}

    - name: "List source directory contents"
      ansible.builtin.find:
        paths: "{{ output_dir.path }}"
        patterns: "*"
        recurse: yes
      register: source_files

    - name: "Display source files"
      ansible.builtin.debug:
        msg: |
          Source files to transfer:
          {% for file in source_files.files | default([]) %}
          - {{ file.path }} ({{ file.size | default(0) }} bytes)
          {% endfor %}

    - name: "Transfer collected files to Triage Server using synchronize (push)"
      ansible.posix.synchronize:
        src: "{{ output_dir.path }}/"
        dest: "root@{{ diagnose_server }}:{{ diagnose_full_path }}"
        recursive: yes
        mode: push
      delegate_to: "{{ inventory_hostname }}"
      register: sync_result
      ignore_errors: yes
      when: source_dir_check.stat.exists and source_dir_check.stat.isdir

    - name: "Display synchronize transfer result"
      ansible.builtin.debug:
        msg: |
          synchronize transfer result:
          - Status: {{ 'SUCCESS' if sync_result is succeeded else 'FAILED' }}
          - Rc: {{ sync_result.rc | default('N/A') }}
          - Stdout: {{ sync_result.stdout | default('') }}
          - Stderr: {{ sync_result.stderr | default('') }}

    - name: "Verify files were transferred successfully"
      ansible.builtin.find:
        paths: "{{ diagnose_path }}"
        patterns: "*"
        recurse: yes
      delegate_to: "{{ diagnose_server }}"
      register: transferred_files
      ignore_errors: yes

    - name: "Display transferred files"
      ansible.builtin.debug:
        msg: |
          Transferred files on Triage Server:
          {% for file in transferred_files.files | default([]) %}
          - {{ file.path }} ({{ file.size | default(0) }} bytes)
          {% endfor %}

    # ===========================================
    # Phase 5：Alert 信息文件处理
    # -------------------------------------------
    # 查找的就是 Phase 0 中新生成的
    # /tmp/<host>-<alertname>-alert_info.txt 文件
    # ===========================================
    - name: "Find alert info files for this host IP"
      ansible.builtin.find:
        paths: "/tmp"
        patterns: "{{ inventory_hostname }}-*-alert_info.txt"
        recurse: no
      register: alert_files

    - name: "Display found alert files"
      ansible.builtin.debug:
        msg: |
          Found alert files:
          {% for file in alert_files.files %}
          - {{ file.path }} ({{ file.mtime }})
          {% endfor %}

    - name: "Get the latest alert file"
      ansible.builtin.set_fact:
        latest_alert_file: "{{ alert_files.files | sort(attribute='mtime') | last }}"
      when: alert_files.files | length > 0

    - name: "Display latest alert file"
      ansible.builtin.debug:
        msg: "Latest alert file: {{ latest_alert_file.path }} ({{ latest_alert_file.mtime }})"
      when: latest_alert_file is defined

    - name: "Transfer latest alert file to Triage Server"
      ansible.posix.synchronize:
        src: "{{ latest_alert_file.path }}"
        dest: "root@{{ diagnose_server }}:{{ diagnose_path }}/"
        mode: push
      delegate_to: "{{ inventory_hostname }}"
      register: alert_transfer_result
      ignore_errors: yes
      when: latest_alert_file is defined

    - name: "Display alert file transfer result"
      ansible.builtin.debug:
        msg: |
          Alert file transfer result:
          - Status: {{ 'SUCCESS' if alert_transfer_result is succeeded else 'FAILED' }}
          - Rc: {{ alert_transfer_result.rc | default('N/A') }}
          - Stdout: {{ alert_transfer_result.stdout | default('') }}
          - Stderr: {{ alert_transfer_result.stderr | default('') }}
          - Transferred file: {{ latest_alert_file.path | default('N/A') }}
      when: latest_alert_file is defined

    - name: "Verify alert file was transferred successfully"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}/{{ latest_alert_file.path | basename }}"
      delegate_to: "{{ diagnose_server }}"
      register: alert_file_check
      ignore_errors: yes
      when: latest_alert_file is defined

    - name: "Display alert file verification result"
      ansible.builtin.debug:
        msg: |
          Alert file verification:
          - File exists on Triage Server: {{ alert_file_check.stat.exists | default(false) }}
          - File size: {{ alert_file_check.stat.size | default('N/A') }} bytes
          - File path: {{ diagnose_path }}/{{ latest_alert_file.path | basename | default('N/A') }}
      when: latest_alert_file is defined

    # ===========================================
    # Phase 6：文件合并和最终报告生成
    # ===========================================
    - name: "Merge all files into issues_latest.txt on Triage Server"
      ansible.builtin.shell: |
        cd {{ diagnose_path }}
        ALERT_FILE="{{ latest_alert_file.path | basename | default('') }}"

        if [ -n "$ALERT_FILE" ] && [ -f "$ALERT_FILE" ]; then
          echo "=== ALERT INFO ===" > issues_latest.txt
          cat "$ALERT_FILE" >> issues_latest.txt
          echo "" >> issues_latest.txt
          echo "=== OTHER DIAGNOSTIC FILES ===" >> issues_latest.txt
          echo "" >> issues_latest.txt
        else
          echo "=== DIAGNOSTIC FILES ===" > issues_latest.txt
          echo "" >> issues_latest.txt
        fi

        find . -maxdepth 1 -type f \( -name "*.txt" -o -name "*.log" \) \
          ! -name "issues_latest.txt" ! -name "$ALERT_FILE" -print0 |
        while IFS= read -r -d '' file; do
          echo "=== ${file#./} ===" >> issues_latest.txt
          cat "$file" >> issues_latest.txt
          echo "" >> issues_latest.txt
          echo "" >> issues_latest.txt
        done

        echo "Files merged into issues_latest.txt:"
        echo "Alert file used: $ALERT_FILE"
        ls -la issues_latest.txt
      args:
        executable: /bin/bash
      delegate_to: "{{ diagnose_server }}"
      register: merge_result
      ignore_errors: yes

    - name: "Display merge result"
      ansible.builtin.debug:
        msg: |
          File merge completed:
          - Return code: {{ merge_result.rc | default('N/A') }}
          - Stdout: {{ merge_result.stdout | default('') }}
          - Stderr: {{ merge_result.stderr | default('') }}
          - Merged file: {{ diagnose_path }}/issues_latest.txt

    - name: "Verify merged file exists"
      ansible.builtin.stat:
        path: "{{ diagnose_path }}/issues_latest.txt"
      delegate_to: "{{ diagnose_server }}"
      register: merged_file_stat

    - name: "Display merged file information"
      ansible.builtin.debug:
        msg: |
          Merged file verification:
          - File exists: {{ merged_file_stat.stat.exists }}
          - File size: {{ merged_file_stat.stat.size | default(0) }} bytes
          - File path: {{ diagnose_path }}/issues_latest.txt

    - name: "Notify Triage Server directory location"
      ansible.builtin.debug:
        msg: "Data transferred to Triage Server: {{ diagnose_path }}"

    - name: "Pass diagnose context to next play via dummy host"
      ansible.builtin.add_host:
        name: "_eda_n8n_ctx"
        groups: "eda_n8n_ctx"
        diagnose_path: "{{ diagnose_path }}"
        target_host: "{{ inventory_hostname }}"
        target_device: "{{ target_device }}"
        alert_name: "{{ alert_name }}"
        alert_instance: "{{ alert_instance }}"
        alert_severity: "{{ alert_severity }}"
        alert_status: "{{ alert_status }}"
        alert_receiver: "{{ alert_receiver }}"
        webhook_payload: "{{ webhook_payload }}"


# =============================================================================
# 最后一个 Play：调用 n8n_webhook_url 激活 n8n 工作流
# -----------------------------------------------------------------------------
# 与 CPU 版本对齐：在 AAP controller (localhost) 上执行，不依赖故障节点。
# =============================================================================
- name: "Notify n8n workflow to start network-fault-diagnosis pipeline"
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    n8n_webhook_url: "http://10.210.65.103:8080/webhook/9a1d6d25-2a9b-4906-afd7-21c3b4a49860"
    ctx: "{{ hostvars['_eda_n8n_ctx'] | default({}) }}"

  tasks:
    - name: "Display n8n notification context"
      ansible.builtin.debug:
        msg: |
          About to notify n8n workflow:
          - URL            : {{ n8n_webhook_url }}
          - Target host    : {{ ctx.target_host    | default('Unknown') }}
          - Target device  : {{ ctx.target_device  | default('Unknown') }}
          - Alert name     : {{ ctx.alert_name     | default('Unknown') }}
          - Alert instance : {{ ctx.alert_instance | default('Unknown') }}
          - Severity       : {{ ctx.alert_severity | default('Unknown') }}
          - Diagnose path  : {{ ctx.diagnose_path  | default('Unknown') }}

    - name: "POST to n8n webhook to trigger network-fault-diagnosis workflow"
      ansible.builtin.uri:
        url: "{{ n8n_webhook_url }}"
        method: POST
        body_format: json
        headers:
          Content-Type: "application/json"
        body:
          alert_instance: "{{ ctx.alert_instance | default('Unknown') }}"
          alert_name: "{{ ctx.alert_name | default('Unknown') }}"
          severity: "{{ ctx.alert_severity | default('Unknown') }}"
          status: "{{ ctx.alert_status | default('Unknown') }}"
          receiver: "{{ ctx.alert_receiver | default('Unknown') }}"
          target_host: "{{ ctx.target_host | default('Unknown') }}"
          target_device: "{{ ctx.target_device | default('Unknown') }}"
          diagnose_path: "{{ ctx.diagnose_path | default('Unknown') }}"
          issues_latest: "{{ ctx.diagnose_path | default('') }}/issues_latest.txt"
          webhook_payload: "{{ ctx.webhook_payload | default({}) }}"
        status_code: [200, 201, 202, 204]
        timeout: 15
      register: n8n_result
      ignore_errors: yes

    - name: "Display n8n webhook invocation result"
      ansible.builtin.debug:
        msg: |
          n8n webhook invocation result:
          - Status     : {{ 'SUCCESS' if n8n_result is succeeded else 'FAILED' }}
          - HTTP code  : {{ n8n_result.status | default('N/A') }}
          - Response   : {{ n8n_result.json | default(n8n_result.content | default('')) }}
          - Error msg  : {{ n8n_result.msg | default('') }}
```

---

## 4. EDA Rulebook Action Job Templates Configuration

### 4.1 Use Case 1 Action JT

<put image here>

Important: set Extra variables to: `webhook_payload: '{{ event.payload }}'`

### 4.2 Use Case 2 Action JT

<put image here>

Important: set Extra variables to: `webhook_payload: '{{ event.payload }}'`



---

## 5. Other AIOPS Playbooks and Job Templates

### 5.1 Use Case 1 Self-Healing Playbook


```text
[admin@aap26 AIOPS]$ more 04_proceed_selfhealing.yml
```
```yaml
# Self-Healing：清理高 CPU 进程 → 重新巡检 → Markdown 报告 → 上传 Mattermost
#
# 流程：
#   1) 在故障服务器执行 ps -ef，挑出 CPU > 98% 的进程并 kill
#   2) 等待几秒后重新做一次性能巡检（uptime / top / free / df / ps top10）
#   3) 生成 .md 格式报告，先 debug 打印，再保存到
#      /tmp/<故障服务器hostname>-<时间戳>.md
#   4) 经控制节点中转上传到
#      Mattermost (10.210.65.14):/performance_report/<故障服务器hostname>-<时间戳>.md

---
- name: "Self-Healing: kill >98% CPU processes and re-inspect"
  hosts: "{{ target_group | default('Issue_servers') }}"
  become: true
  gather_facts: true

  vars:
    cpu_threshold: 98
    settle_seconds: 5
    mattermost_host: "10.210.65.14"
    mattermost_report_dir: "/performance_report"
    controller_staging_dir: "/tmp/selfhealing_staging"

    timestamp_suffix: "{{ ansible_date_time.date }}-{{ ansible_date_time.hour }}{{ ansible_date_time.minute }}"
    report_filename: "{{ ansible_hostname }}-{{ timestamp_suffix }}.md"
    local_report_path: "/tmp/{{ report_filename }}"

  tasks:

    # ===========================================
    # Phase 1：识别 CPU > 98% 的进程
    # -------------------------------------------
    # 注意：ps 的 %CPU 是「进程生命周期平均值」(cputime/walltime)，
    # 长时间运行的进程即使当前 99%，ps 显示可能很低。
    # 必须用 top -b -n 2 -d 1 取第二次迭代的「瞬时 %CPU」。
    # ===========================================
    - name: "Sample real-time CPU usage via top (2 iterations, 1s apart)"
      ansible.builtin.shell: |
        set -o pipefail
        # 抓第二次迭代后的进程表（top 第一次迭代的 %CPU 与 ps 类似不准）
        top -b -n 2 -d 1 -w 512 \
          | awk '/^top - /{i++; next} i==2 && NF>=12 && $1 ~ /^[0-9]+$/ {print}'
      args:
        executable: /bin/bash
      register: top_second_iter
      changed_when: false

    - name: "Filter processes with %CPU > {{ cpu_threshold }} (column 9 of top)"
      ansible.builtin.shell: |
        set -o pipefail
        echo "{{ top_second_iter.stdout }}" \
          | awk -v thr={{ cpu_threshold }} '$9+0 > thr {print}'
      args:
        executable: /bin/bash
      register: high_cpu_table
      changed_when: false

    - name: "Extract PIDs from filtered top output"
      ansible.builtin.shell: |
        set -o pipefail
        echo "{{ top_second_iter.stdout }}" \
          | awk -v thr={{ cpu_threshold }} '$9+0 > thr {print $1}'
      args:
        executable: /bin/bash
      register: high_cpu_pids
      changed_when: false

    - name: "Show pre-kill snapshot"
      ansible.builtin.debug:
        msg:
          - "----- top (iter 2) raw lines -----"
          - "{{ top_second_iter.stdout_lines | default([]) }}"
          - "----- Processes with %CPU > {{ cpu_threshold }} -----"
          - "{{ high_cpu_table.stdout_lines | default([]) }}"
          - "----- PIDs to kill -----"
          - "{{ high_cpu_pids.stdout_lines | default([]) }}"

    # ===========================================
    # Phase 2：沿 PPID 链 kill 整条进程组
    # -------------------------------------------
    # 适配 multiprocessing / wrapper 启动器场景：
    #   仅 kill worker，父进程下一秒可能 fork 出新 worker → 看着没生效
    # 策略：
    #   对每个高 CPU PID，沿 PPID 一路向上收集祖先，直到：
    #     - PPID == 1（systemd/init），或
    #     - 父进程的 comm 落在 SAFE_COMM 白名单（bash/sshd/login/...）
    #   把整条链上的进程一次性 SIGTERM → 等 2s → 残留 SIGKILL
    # ===========================================
    - name: "Kill high-CPU processes AND their non-system ancestors"
      ansible.builtin.shell: |
        set +e
        # 系统进程白名单：遇到这些祖先就停手，避免误杀 shell / sshd / systemd
        SAFE_COMM="systemd init bash sh zsh fish dash tmux screen sshd login agetty cron crond sudo su nohup"

        # 收集每个高 CPU PID 的整条进程链
        declare -A KILL_SET
        for pid in {{ high_cpu_pids.stdout_lines | join(' ') }}; do
          current="$pid"
          depth=0
          while [ -n "$current" ] && [ "$current" != "1" ] && [ "$current" != "0" ] && [ $depth -lt 10 ]; do
            KILL_SET[$current]=1
            parent=$(ps -o ppid= -p "$current" 2>/dev/null | tr -d ' ')
            [ -z "$parent" ] && break
            parent_comm=$(ps -o comm= -p "$parent" 2>/dev/null | tr -d ' ')
            # 父进程是系统/shell 进程 → 不再向上走
            stop=0
            for safe in $SAFE_COMM; do
              if [ "$parent_comm" = "$safe" ]; then stop=1; break; fi
            done
            [ $stop -eq 1 ] && break
            current="$parent"
            depth=$((depth + 1))
          done
        done

        ALL_PIDS=$(echo "${!KILL_SET[@]}" | tr ' ' '\n' | sort -un | tr '\n' ' ')
        echo "Process chain to kill: $ALL_PIDS"
        echo "-----  Snapshot before kill -----"
        ps -o pid,ppid,user,%cpu,comm,args -p $ALL_PIDS 2>/dev/null || true

        echo "-----  SIGTERM (15) -----"
        for p in $ALL_PIDS; do kill -15 "$p" 2>/dev/null && echo "TERM $p"; done
        sleep 2

        echo "-----  SIGKILL (9) for holdouts -----"
        for p in $ALL_PIDS; do
          if kill -0 "$p" 2>/dev/null; then
            kill -9 "$p" 2>/dev/null && echo "KILL $p"
          fi
        done

        echo "kill cycle completed"
      args:
        executable: /bin/bash
      register: kill_result
      when: high_cpu_pids.stdout_lines | length > 0
      changed_when: high_cpu_pids.stdout_lines | length > 0

    - name: "No high-CPU processes found — skip kill"
      ansible.builtin.debug:
        msg: "No process exceeded {{ cpu_threshold }}% CPU; nothing to kill."
      when: high_cpu_pids.stdout_lines | length == 0

    - name: "Show kill result"
      ansible.builtin.debug:
        msg: "{{ kill_result.stdout_lines | default(['(no kill action taken)']) }}"

    # ===========================================
    # Phase 2.5：等待并验证是否真的清干净
    # ===========================================
    - name: "Wait {{ settle_seconds }}s for system to settle"
      ansible.builtin.pause:
        seconds: "{{ settle_seconds }}"

    - name: "Re-sample top to verify no high-CPU survivor"
      ansible.builtin.shell: |
        set -o pipefail
        top -b -n 2 -d 1 -w 512 \
          | awk '/^top - /{i++; next} i==2 && NF>=12 && $1 ~ /^[0-9]+$/ {print}' \
          | awk -v thr={{ cpu_threshold }} '$9+0 > thr {print}'
      args:
        executable: /bin/bash
      register: post_kill_check
      changed_when: false

    - name: "Warn if a high-CPU process re-emerged (likely auto-restarted by parent/systemd)"
      ansible.builtin.debug:
        msg:
          - "Post-kill check: a process is STILL > {{ cpu_threshold }}% CPU."
          - "It may be re-spawned by a wrapper script or systemd unit."
          - "{{ post_kill_check.stdout_lines }}"
      when: post_kill_check.stdout_lines | length > 0

    # ===========================================
    # Phase 3：重新性能巡检
    # ===========================================
    - name: "Collect uptime / load"
      ansible.builtin.command: uptime
      register: post_uptime
      changed_when: false

    - name: "Collect top 20 lines"
      ansible.builtin.shell: "top -b -n 1 | head -n 20"
      register: post_top
      changed_when: false

    - name: "Collect free -h"
      ansible.builtin.command: free -h
      register: post_free
      changed_when: false

    - name: "Collect df -h"
      ansible.builtin.command: df -h
      register: post_df
      changed_when: false

    - name: "Collect top-10 CPU processes after kill"
      ansible.builtin.shell: |
        set -o pipefail
        ps -eo pid,user,%cpu,%mem,comm,args --sort=-%cpu | head -n 11
      args:
        executable: /bin/bash
      register: post_ps_top
      changed_when: false

    # ===========================================
    # Phase 4：生成 Markdown 报告
    # ===========================================
    - name: "Build Markdown report content"
      ansible.builtin.set_fact:
        report_content: |
          # Self-Healing Performance Report

          | Field | Value |
          | --- | --- |
          | Host | {{ ansible_hostname }} ({{ inventory_hostname }}) |
          | OS | {{ ansible_distribution }} {{ ansible_distribution_version }} |
          | Kernel | {{ ansible_kernel }} |
          | Generated | {{ ansible_date_time.iso8601 }} |
          | CPU threshold | > {{ cpu_threshold }} % |
          | PIDs killed | {{ high_cpu_pids.stdout_lines | default([]) | join(', ') if (high_cpu_pids.stdout_lines | default([]) | length > 0) else 'None' }} |

          ## 1. Pre-Kill Snapshot (CPU > {{ cpu_threshold }}%)

          ```text
          {{ high_cpu_table.stdout | trim }}
          ```

          ## 2. Kill Action (worker + ancestor chain)

          ```text
          {{ kill_result.stdout | default('No processes killed.') | trim }}
          ```

          {% if post_kill_check.stdout_lines | default([]) | length > 0 %}
          > Warning: a process is STILL above {{ cpu_threshold }}% CPU after kill.
          > It is likely re-spawned by a wrapper script or a systemd unit.
          > Survivor list:
          >
          > ```text
          > {{ post_kill_check.stdout | trim }}
          > ```
          {% else %}
          > Post-kill verification passed: no process exceeds {{ cpu_threshold }}% CPU.
          {% endif %}

          ## 3. Post-Kill Inspection

          ### 3.1 uptime

          ```text
          {{ post_uptime.stdout | trim }}
          ```

          ### 3.2 top (first 20 lines)

          ```text
          {{ post_top.stdout | trim }}
          ```

          ### 3.3 free -h

          ```text
          {{ post_free.stdout | trim }}
          ```

          ### 3.4 df -h

          ```text
          {{ post_df.stdout | trim }}
          ```

          ### 3.5 Top 10 processes by CPU

          ```text
          {{ post_ps_top.stdout | trim }}
          ```

          ---

          _Report auto-generated by AAP playbook `04_proceed_selfhealing.yml`._

    - name: "Print Markdown report"
      ansible.builtin.debug:
        msg: "{{ report_content.split('\n') }}"

    - name: "Save report to /tmp/{{ report_filename }}"
      ansible.builtin.copy:
        content: "{{ report_content }}"
        dest: "{{ local_report_path }}"
        owner: root
        group: root
        mode: "0644"

    - name: "Stat saved report"
      ansible.builtin.stat:
        path: "{{ local_report_path }}"
      register: report_stat

    - name: "Fail if local report is empty"
      ansible.builtin.fail:
        msg: "Local report {{ local_report_path }} missing or too small."
      when: not report_stat.stat.exists or report_stat.stat.size | int < 200

    # ===========================================
    # Phase 5：经控制节点中转 → Mattermost
    # ===========================================
    - name: "Prepare staging dir on controller"
      ansible.builtin.file:
        path: "{{ controller_staging_dir }}"
        state: directory
        mode: "0755"
      delegate_to: localhost
      become: false

    - name: "Fetch report from fault host to controller"
      ansible.builtin.fetch:
        src: "{{ local_report_path }}"
        dest: "{{ controller_staging_dir }}/{{ report_filename }}"
        flat: true

    - name: "Ensure {{ mattermost_report_dir }} exists on Mattermost"
      ansible.builtin.file:
        path: "{{ mattermost_report_dir }}"
        state: directory
        mode: "0755"
        owner: root
        group: root
      delegate_to: "{{ mattermost_host }}"

    - name: "Upload report to Mattermost:{{ mattermost_report_dir }}"
      ansible.builtin.copy:
        src: "{{ controller_staging_dir }}/{{ report_filename }}"
        dest: "{{ mattermost_report_dir }}/{{ report_filename }}"
        mode: "0644"
        owner: root
        group: root
      delegate_to: "{{ mattermost_host }}"

    - name: "Verify uploaded file on Mattermost"
      ansible.builtin.stat:
        path: "{{ mattermost_report_dir }}/{{ report_filename }}"
      delegate_to: "{{ mattermost_host }}"
      register: remote_stat

    - name: "Fail if upload size mismatch"
      ansible.builtin.fail:
        msg: >-
          Upload verification failed:
          local={{ report_stat.stat.size }} bytes,
          remote={{ remote_stat.stat.size | default('N/A') }} bytes
      when:
        - not remote_stat.stat.exists or
          (remote_stat.stat.size | default(0) | int) != (report_stat.stat.size | int)

    - name: "Cleanup controller staging file"
      ansible.builtin.file:
        path: "{{ controller_staging_dir }}/{{ report_filename }}"
        state: absent
      delegate_to: localhost
      become: false

    - name: "Summary"
      ansible.builtin.debug:
        msg:
          - "Self-Healing completed"
          - "Host        : {{ ansible_hostname }} ({{ inventory_hostname }})"
          - "PIDs killed : {{ high_cpu_pids.stdout_lines | default([]) | join(', ') if (high_cpu_pids.stdout_lines | default([]) | length > 0) else 'None' }}"
          - "Local file  : {{ local_report_path }} ({{ report_stat.stat.size }} B)"
          - "Remote file : {{ mattermost_host }}:{{ mattermost_report_dir }}/{{ report_filename }}"
          - "View        : ssh root@{{ mattermost_host }} less {{ mattermost_report_dir }}/{{ report_filename }}"

```

### 5.2 Use Case 2 Self-Healing Playbook


```text
[admin@aap26 AIOPS]$ more 05_network_interface_selfhealing.yml
```
```yaml
# Self-Healing：找出 DOWN 网卡 → 自动修复 → 健康检查 → Markdown 报告 → 上传 Mattermost
#
# 流程：
#   Phase 1  扫描所有物理网卡，挑出 state DOWN 的（排除 lo / docker / br / virbr / veth / cni 等）
#   Phase 2  逐个尝试 `ip link set <dev> up`；3s 后若仍未 UP，再用 `nmcli connection up` 兜底
#   Phase 3  修复后健康检查：ip a / ip -br link / 各网卡 carrier+operstate / 默认路由连通
#   Phase 4  生成 .md 报告，debug 打印 + 保存到 /tmp/<hostname>-<时间戳>.md
#   Phase 5  经控制节点中转上传到 Mattermost (10.210.65.14):/network_report/
#
# 安全约束：
#   PROTECTED_DEVS（默认 ens34）—— 管理网卡永不允许被关闭，即使被识别为 DOWN 也跳过。
#
# AAP Survey（可选）：
#   Answer variable name : target_group
#   示例                 : Chaos.example.com / Issue_servers
#
# 手动运行：
#   ansible-playbook 05_network_interface_selfhealing.yml -e target_group=Chaos.example.com
---
- name: "Self-Healing: bring DOWN network interfaces back up"
  hosts: "{{ target_group | default('Issue_servers') }}"
  become: true
  gather_facts: true

  vars:
    # 受保护网卡：永不操作（避免远程失联）
    protected_devices:
      - ens34
    # 修复后等待网卡 settle 的秒数
    settle_seconds: 3
    nm_fallback_wait: 5

    mattermost_host: "10.210.65.14"
    mattermost_report_dir: "/network_report"
    controller_staging_dir: "/tmp/network_selfhealing_staging"

    timestamp_suffix: "{{ ansible_date_time.date }}-{{ ansible_date_time.hour }}{{ ansible_date_time.minute }}"
    report_filename: "{{ ansible_hostname }}-{{ timestamp_suffix }}.md"
    local_report_path: "/tmp/{{ report_filename }}"

  tasks:

    # ===========================================
    # Phase 1：发现故障网卡
    # ===========================================
    - name: "Scan for DOWN physical interfaces (excluding lo / docker / br / virbr / veth / cni / kube)"
      ansible.builtin.shell: |
        set -o pipefail
        ip -o link show \
          | awk -F': ' '{print $2" "$0}' \
          | grep -v -E '^(lo|veth|docker|br-|virbr|cni|cilium|kube|vnet|tun|tap)' \
          | awk '/state DOWN/ {print $1}'
      args:
        executable: /bin/bash
      register: down_scan
      changed_when: false

    - name: "Filter out protected devices from target list"
      ansible.builtin.set_fact:
        target_devices: >-
          {{ down_scan.stdout_lines
             | default([])
             | difference(protected_devices) }}

    - name: "Show pre-heal snapshot"
      ansible.builtin.debug:
        msg:
          - "----- DOWN interfaces detected -----"
          - "{{ down_scan.stdout_lines | default([]) }}"
          - "----- Protected (skipped) -----"
          - "{{ protected_devices }}"
          - "----- Will try to bring UP -----"
          - "{{ target_devices }}"

    - name: "Capture pre-heal ip -br link"
      ansible.builtin.command: ip -br link
      register: pre_brlink
      changed_when: false

    # ===========================================
    # Phase 2：自动修复
    # ===========================================
    - name: "Detect NetworkManager presence"
      ansible.builtin.command: which nmcli
      register: nmcli_check
      changed_when: false
      ignore_errors: yes

    - name: "Attempt 'ip link set <dev> up' for each target device"
      ansible.builtin.command: "ip link set {{ item }} up"
      loop: "{{ target_devices }}"
      register: ip_link_results
      ignore_errors: yes

    - name: "Wait {{ settle_seconds }}s for links to settle"
      ansible.builtin.pause:
        seconds: "{{ settle_seconds }}"
      when: target_devices | length > 0

    - name: "Re-check operstate after ip link up"
      ansible.builtin.shell: |
        cat /sys/class/net/{{ item }}/operstate 2>/dev/null || echo unknown
      args:
        executable: /bin/bash
      loop: "{{ target_devices }}"
      register: operstate_mid
      changed_when: false

    - name: "Build list of devices still not UP after ip link up"
      ansible.builtin.set_fact:
        still_down_devices: >-
          {{ operstate_mid.results
             | rejectattr('stdout', 'equalto', 'up')
             | rejectattr('stdout', 'equalto', 'unknown')
             | map(attribute='item')
             | list }}

    - name: "Find NetworkManager connection names for still-down devices"
      ansible.builtin.shell: |
        nmcli -t -f NAME,DEVICE connection show \
          | awk -F: -v dev="{{ item }}" '$2==dev {print $1; exit}'
      args:
        executable: /bin/bash
      loop: "{{ still_down_devices }}"
      register: nm_conn_names
      when:
        - nmcli_check.rc == 0
        - still_down_devices | length > 0
      changed_when: false
      ignore_errors: yes

    - name: "Fallback: nmcli connection up for still-down devices"
      ansible.builtin.command: "nmcli connection up '{{ item.stdout | trim }}'"
      loop: "{{ nm_conn_names.results | default([]) }}"
      loop_control:
        label: "{{ item.item | default('') }}"
      when:
        - nmcli_check.rc == 0
        - still_down_devices | length > 0
        - item.stdout is defined
        - (item.stdout | trim) | length > 0
      register: nmcli_up_results
      ignore_errors: yes

    - name: "Wait {{ nm_fallback_wait }}s for NetworkManager to apply changes"
      ansible.builtin.pause:
        seconds: "{{ nm_fallback_wait }}"
      when:
        - nmcli_check.rc == 0
        - still_down_devices | length > 0

    # ===========================================
    # Phase 3：修复后健康检查
    # ===========================================
    - name: "Post-heal ip -br link"
      ansible.builtin.command: ip -br link
      register: post_brlink
      changed_when: false

    - name: "Post-heal ip -br addr"
      ansible.builtin.command: ip -br addr
      register: post_braddr
      changed_when: false

    - name: "Post-heal ip route"
      ansible.builtin.command: ip route
      register: post_route
      changed_when: false

    - name: "Collect carrier + operstate for each target device"
      ansible.builtin.shell: |
        for d in {{ target_devices | join(' ') }}; do
          c=$(cat /sys/class/net/$d/carrier   2>/dev/null || echo n/a)
          s=$(cat /sys/class/net/$d/operstate 2>/dev/null || echo n/a)
          printf '%-12s carrier=%s  operstate=%s\n' "$d" "$c" "$s"
        done
      args:
        executable: /bin/bash
      register: post_dev_state
      changed_when: false
      when: target_devices | length > 0

    - name: "Ping default gateway (best-effort)"
      ansible.builtin.shell: |
        set -o pipefail
        GW=$(ip route show default 0/0 | awk '/default/ {print $3; exit}')
        if [ -n "$GW" ]; then
          echo "Default gateway: $GW"
          ping -c 2 -W 2 "$GW" 2>&1 || true
        else
          echo "No default gateway in routing table."
        fi
      args:
        executable: /bin/bash
      register: gw_ping
      changed_when: false

    - name: "Determine per-device recovery status"
      ansible.builtin.set_fact:
        per_device_status: >-
          {{ per_device_status | default([]) +
             [{
               'device': item,
               'operstate': (lookup('pipe', 'cat /sys/class/net/' ~ item ~ '/operstate 2>/dev/null || echo unknown') | trim),
               'carrier':   (lookup('pipe', 'cat /sys/class/net/' ~ item ~ '/carrier   2>/dev/null || echo n/a') | trim)
             }] }}
      loop: "{{ target_devices }}"

    - name: "Determine overall recovery_ok"
      ansible.builtin.set_fact:
        recovery_ok: >-
          {{ target_devices | length == 0 or
             (per_device_status | default([])
               | rejectattr('operstate', 'equalto', 'up')
               | rejectattr('operstate', 'equalto', 'unknown')
               | list | length == 0) }}

    # ===========================================
    # Phase 4：生成 Markdown 报告
    # ===========================================
    - name: "Build Markdown report content"
      ansible.builtin.set_fact:
        report_content: |
          # Network Interface Self-Healing Report

          | Field | Value |
          | --- | --- |
          | Host | {{ ansible_hostname }} ({{ inventory_hostname }}) |
          | OS | {{ ansible_distribution }} {{ ansible_distribution_version }} |
          | Kernel | {{ ansible_kernel }} |
          | Generated | {{ ansible_date_time.iso8601 }} |
          | Protected devices | {{ protected_devices | join(', ') }} |
          | Detected DOWN | {{ down_scan.stdout_lines | default([]) | join(', ') if (down_scan.stdout_lines | default([]) | length > 0) else 'None' }} |
          | Targets | {{ target_devices | join(', ') if (target_devices | length > 0) else 'None' }} |
          | Recovery OK | {{ recovery_ok }} |

          ## 1. Pre-Heal Snapshot

          ```text
          {{ pre_brlink.stdout | trim }}
          ```

          ## 2. Heal Actions

          ### 2.1 ip link set up

          ```text
          {% for r in ip_link_results.results | default([]) %}
          {{ r.item }}: rc={{ r.rc | default('N/A') }}  {{ ('stderr=' ~ (r.stderr | default('') | trim)) if r.stderr | default('') | trim | length > 0 else '' }}
          {% endfor %}
          {% if (ip_link_results.results | default([])) | length == 0 %}
          (no device needed `ip link set up`)
          {% endif %}
          ```

          {% if still_down_devices | length > 0 %}
          ### 2.2 nmcli connection up (fallback)

          ```text
          {% for r in nmcli_up_results.results | default([]) %}
          {{ r.item.item | default('') }}: rc={{ r.rc | default('N/A') }}  {{ ('stderr=' ~ (r.stderr | default('') | trim)) if r.stderr | default('') | trim | length > 0 else '' }}
          {% endfor %}
          {% if (nmcli_up_results.results | default([])) | length == 0 %}
          (nmcli fallback was triggered but no connection name was resolved)
          {% endif %}
          ```
          {% else %}
          ### 2.2 nmcli fallback
          Not needed — every target device came up after `ip link set up`.
          {% endif %}

          ## 3. Post-Heal Health Check

          ### 3.1 Per-device state

          ```text
          {{ post_dev_state.stdout | default('(no target devices)') | trim }}
          ```

          ### 3.2 ip -br link

          ```text
          {{ post_brlink.stdout | trim }}
          ```

          ### 3.3 ip -br addr

          ```text
          {{ post_braddr.stdout | trim }}
          ```

          ### 3.4 ip route

          ```text
          {{ post_route.stdout | trim }}
          ```

          ### 3.5 Default gateway connectivity

          ```text
          {{ gw_ping.stdout | trim }}
          ```

          ---

          {% if recovery_ok %}
          ✅ **All targeted interfaces are UP. Self-heal succeeded.**
          {% else %}
          ⚠️ **Some interfaces remain DOWN — likely a physical / link-layer problem (no cable, peer down, driver issue).**
          {% endif %}

          _Report auto-generated by AAP playbook `05_network_interface_selfhealing.yml`._

    - name: "Print Markdown report"
      ansible.builtin.debug:
        msg: "{{ report_content.split('\n') }}"

    - name: "Save report to /tmp/{{ report_filename }}"
      ansible.builtin.copy:
        content: "{{ report_content }}"
        dest: "{{ local_report_path }}"
        owner: root
        group: root
        mode: "0644"

    - name: "Stat saved report"
      ansible.builtin.stat:
        path: "{{ local_report_path }}"
      register: report_stat

    - name: "Fail if local report is empty"
      ansible.builtin.fail:
        msg: "Local report {{ local_report_path }} missing or too small."
      when: not report_stat.stat.exists or report_stat.stat.size | int < 200

    # ===========================================
    # Phase 5：经控制节点中转 → Mattermost
    # ===========================================
    - name: "Prepare staging dir on controller"
      ansible.builtin.file:
        path: "{{ controller_staging_dir }}"
        state: directory
        mode: "0755"
      delegate_to: localhost
      become: false

    - name: "Fetch report from fault host to controller"
      ansible.builtin.fetch:
        src: "{{ local_report_path }}"
        dest: "{{ controller_staging_dir }}/{{ report_filename }}"
        flat: true

    - name: "Ensure {{ mattermost_report_dir }} exists on Mattermost"
      ansible.builtin.file:
        path: "{{ mattermost_report_dir }}"
        state: directory
        mode: "0755"
        owner: root
        group: root
      delegate_to: "{{ mattermost_host }}"

    - name: "Upload report to Mattermost:{{ mattermost_report_dir }}"
      ansible.builtin.copy:
        src: "{{ controller_staging_dir }}/{{ report_filename }}"
        dest: "{{ mattermost_report_dir }}/{{ report_filename }}"
        mode: "0644"
        owner: root
        group: root
      delegate_to: "{{ mattermost_host }}"

    - name: "Verify uploaded file on Mattermost"
      ansible.builtin.stat:
        path: "{{ mattermost_report_dir }}/{{ report_filename }}"
      delegate_to: "{{ mattermost_host }}"
      register: remote_stat

    - name: "Fail if upload size mismatch"
      ansible.builtin.fail:
        msg: >-
          Upload verification failed:
          local={{ report_stat.stat.size }} bytes,
          remote={{ remote_stat.stat.size | default('N/A') }} bytes
      when:
        - not remote_stat.stat.exists or
          (remote_stat.stat.size | default(0) | int) != (report_stat.stat.size | int)

    - name: "Cleanup controller staging file"
      ansible.builtin.file:
        path: "{{ controller_staging_dir }}/{{ report_filename }}"
        state: absent
      delegate_to: localhost
      become: false

    - name: "Summary"
      ansible.builtin.debug:
        msg:
          - "Network interface self-healing completed"
          - "Host           : {{ ansible_hostname }} ({{ inventory_hostname }})"
          - "Targets healed : {{ target_devices | join(', ') if (target_devices | length > 0) else 'None' }}"
          - "Recovery OK    : {{ recovery_ok }}"
          - "Local file     : {{ local_report_path }} ({{ report_stat.stat.size }} B)"
          - "Remote file    : {{ mattermost_host }}:{{ mattermost_report_dir }}/{{ report_filename }}"
          - "View           : ssh root@{{ mattermost_host }} less {{ mattermost_report_dir }}/{{ report_filename }}"
```

### 5.3 Use Case 3: Unexpected Outage xsos Diagnostic Playbook


```text
[admin@aap26 AIOPS]$ more 02_xsos_report_to_mattermost.yml
```
```yaml
# 使用 xsos 对故障服务器做快速诊断，并把 .md 报告上传到 Mattermost
#
# 流程：
#   1) 在故障服务器上执行 /usr/local/bin/xsos -ya
#      → 输出保存到 /tmp/<inventory_hostname>-<timestamp>.md
#   2) 通过控制节点中转，把该 .md 文件推送到
#      Mattermost (10.210.65.14):/xsos_report/<inventory_hostname>-<timestamp>.md
#
# 前置条件：
#   - 故障服务器与 Mattermost 都已安装 xsos（脚本位于 /usr/local/bin/xsos）
#   - AAP 控制节点能 SSH 到故障服务器与 Mattermost（两者均已纳入 inventory）
#
# 手动运行：
#   ansible-playbook 02_xsos_report_to_mattermost.yml -l chaos.example.com
---
- name: "xsos quick triage → Mattermost"
  hosts: "{{ target_group | default('Issue_servers') }}"
  gather_facts: true
  become: true

  vars:
    xsos_bin: "/usr/local/bin/xsos"
    mattermost_host: "10.210.65.14"
    mattermost_report_dir: "/xsos_report"
    timestamp_suffix: "{{ ansible_date_time.date }}-{{ ansible_date_time.hour }}{{ ansible_date_time.minute }}"
    report_filename: "{{ inventory_hostname }}-{{ timestamp_suffix }}.md"
    local_report_path: "/tmp/{{ report_filename }}"
    controller_staging_dir: "/tmp/xsos_staging"

  tasks:

    # ===========================================
    # 第一阶段：环境检查
    # ===========================================
    - name: "Verify xsos is installed on the fault host"
      ansible.builtin.stat:
        path: "{{ xsos_bin }}"
      register: xsos_stat

    - name: "Fail early if xsos is missing"
      ansible.builtin.fail:
        msg: "{{ xsos_bin }} not found on {{ inventory_hostname }}. Install xsos first."
      when: not xsos_stat.stat.exists or not xsos_stat.stat.executable

    # ===========================================
    # 第二阶段：在故障服务器生成 xsos 报告
    # ===========================================
    - name: "Run xsos -ya and write Markdown report to /tmp"
      ansible.builtin.shell: |
        set -o pipefail
        {
          echo "# xsos report — {{ inventory_hostname }}"
          echo ""
          echo "- Host       : {{ inventory_hostname }}"
          echo "- Generated  : $(date -Iseconds)"
          echo "- Command    : {{ xsos_bin }} -ya"
          echo ""
          echo '```text'
          {{ xsos_bin }} -ya 2>&1 || true
          echo '```'
        } > {{ local_report_path }}
      args:
        executable: /bin/bash
      register: xsos_run
      changed_when: true

    - name: "Stat the generated report on fault host"
      ansible.builtin.stat:
        path: "{{ local_report_path }}"
      register: report_stat

    - name: "Fail if report not generated"
      ansible.builtin.fail:
        msg: "Failed to generate {{ local_report_path }} (size={{ report_stat.stat.size | default(0) }})"
      when: not report_stat.stat.exists or report_stat.stat.size | default(0) | int < 100

    - name: "Show local report info"
      ansible.builtin.debug:
        msg:
          - "Local report : {{ local_report_path }}"
          - "Size (bytes) : {{ report_stat.stat.size }}"

    # ===========================================
    # 第三阶段：经控制节点中转到 Mattermost
    # ===========================================
    - name: "Prepare staging dir on controller"
      ansible.builtin.file:
        path: "{{ controller_staging_dir }}"
        state: directory
        mode: '0755'
      delegate_to: localhost
      become: false
      run_once: false

    - name: "Fetch report from fault host to controller"
      ansible.builtin.fetch:
        src: "{{ local_report_path }}"
        dest: "{{ controller_staging_dir }}/{{ report_filename }}"
        flat: yes

    - name: "Ensure /xsos_report exists on Mattermost"
      ansible.builtin.file:
        path: "{{ mattermost_report_dir }}"
        state: directory
        mode: '0755'
        owner: root
        group: root
      delegate_to: "{{ mattermost_host }}"

    - name: "Upload report to Mattermost:{{ mattermost_report_dir }}"
      ansible.builtin.copy:
        src: "{{ controller_staging_dir }}/{{ report_filename }}"
        dest: "{{ mattermost_report_dir }}/{{ report_filename }}"
        mode: '0644'
        owner: root
        group: root
      delegate_to: "{{ mattermost_host }}"

    # ===========================================
    # 第四阶段：验证 & 汇总
    # ===========================================
    - name: "Verify uploaded file on Mattermost"
      ansible.builtin.stat:
        path: "{{ mattermost_report_dir }}/{{ report_filename }}"
      delegate_to: "{{ mattermost_host }}"
      register: remote_stat

    - name: "Fail if upload size mismatch"
      ansible.builtin.fail:
        msg: >-
          Upload verification failed:
          local={{ report_stat.stat.size }} bytes,
          remote={{ remote_stat.stat.size | default('N/A') }} bytes
      when:
        - not remote_stat.stat.exists or
          (remote_stat.stat.size | default(0) | int) != (report_stat.stat.size | int)

    - name: "Cleanup controller staging file"
      ansible.builtin.file:
        path: "{{ controller_staging_dir }}/{{ report_filename }}"
        state: absent
      delegate_to: localhost
      become: false

    - name: "Summary"
      ansible.builtin.debug:
        msg:
          - "✓ xsos report uploaded successfully"
          - "Source host : {{ inventory_hostname }}"
          - "Local file  : {{ local_report_path }}  ({{ report_stat.stat.size }} B)"
          - "Remote file : {{ mattermost_host }}:{{ mattermost_report_dir }}/{{ report_filename }}"
          - "Open with   : ssh root@{{ mattermost_host }} less {{ mattermost_report_dir }}/{{ report_filename }}"
```

