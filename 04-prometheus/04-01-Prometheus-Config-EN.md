# Prometheus Monitoring System Deployment & Configuration Guide

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

This guide follows Prometheus official best practices: [https://prometheus.io/docs/introduction/overview/](https://prometheus.io/docs/introduction/overview/)

## Architecture Reference

| Role | Description |
| --- | --- |
| **Prometheus Server** | IP: `10.210.65.174` · OS: Red Hat Enterprise Linux release 9.5 (Plow) · Components: Prometheus + Alertmanager + Node Exporter |
| **Managed Server (Chaos Server)** | IP: `10.210.65.148` · OS: Red Hat Enterprise Linux release 9.4 (Plow) · Component: Node Exporter |

Prometheus and Alertmanager configuration files are in the **Attachments** section at the end of this chapter.

## Chapter Objective

On the Prometheus server and managed servers, complete **Steps 1–12**: package download/extract, standard directory and config deployment, systemd services, startup verification, Chaos Node Exporter deployment, and firewall configuration.

| Item | Description |
| --- | --- |
| **Prometheus Server** | `10.210.65.174` |
| **Chaos Server** | `10.210.65.148` |
| **Versions** | Prometheus 3.5.1 · Alertmanager 0.31.1 · Node Exporter 1.10.2 |
| **Install Paths** | `/opt/prometheus` · `/opt/alertmanager` · `/opt/node_exporter` |
| **Config Source** | `04_prometheus.example.com/` |

---

## Executive Summary: Steps at a Glance

| Part | Step | Content |
| --- | --- | --- |
| **Part 1** | 1–4 | Package download and extraction |
| **Part 2** | 5 (5.1–5.4) | Standard configuration deployment |
| **Part 3** | 6–8 | Rules and config deployment |
| **Part 4** | 9 (9.1–9.4) | systemd service configuration |
| **Part 5** | 10–12 | Start and verify services |

---

# Part 1: Package Download and Extraction

## Step 1: Download Prometheus Packages

```bash
# Create temporary download directory
mkdir -p /tmp/prometheus_install && cd /tmp/prometheus_install

# Download Prometheus (version 3.5.1)
wget https://github.com/prometheus/prometheus/releases/download/v3.5.1/prometheus-3.5.1.linux-amd64.tar.gz

# Download Alertmanager (version 0.31.1)
wget https://github.com/prometheus/alertmanager/releases/download/v0.31.1/alertmanager-0.31.1.linux-amd64.tar.gz

# Download Node Exporter (version 1.10.2)
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```

## Step 2: Verify Downloaded Files (Optional but Recommended)

```bash
# Verify file integrity (SHA256 from official release)
sha256sum *.tar.gz
```

## Step 3: Extract to Standard Directories

```bash
# Create standard directories and extract
mkdir -p /opt/prometheus /opt/alertmanager /opt/node_exporter

tar -xzf /tmp/prometheus_install/prometheus-3.5.1.linux-amd64.tar.gz -C /opt/prometheus --strip-components=1
tar -xzf /tmp/prometheus_install/alertmanager-0.31.1.linux-amd64.tar.gz -C /opt/alertmanager --strip-components=1
tar -xzf /tmp/prometheus_install/node_exporter-1.10.2.linux-amd64.tar.gz -C /opt/node_exporter --strip-components=1

# Verify directory structure
ls -l /opt/prometheus
ls -l /opt/alertmanager
ls -l /opt/node_exporter
```

## Step 4: Verify Installation Directory Structure

```bash
# Verify directory
cd /opt
pwd

# View Alertmanager
ls -lh /opt/alertmanager
# Expected:
# -rwxr-xr-x alertmanager    (executable)
# -rw-r--r-- alertmanager.yml (default config, replace later)
# -rwxr-xr-x amtool          (admin tool)

# View Node Exporter
ls -lh /opt/node_exporter
# Expected:
# -rwxr-xr-x node_exporter   (executable)

# View Prometheus
ls -lh /opt/prometheus
# Expected:
# -rwxr-xr-x prometheus       (main binary)
# -rwxr-xr-x promtool         (config validation tool)
# -rw-r--r-- prometheus.yml   (default config, replace later)
```

---

# Part 2: Standard Configuration Deployment

## Step 5: Create Standard Directory Structure

Follow Prometheus best practices to create a standard directory layout.

### 5.1 Create Prometheus Directory Structure

```bash
# Create directories required by Prometheus
cd /opt/prometheus
mkdir -p data rules config consoles console_libraries targets

# Verify directory structure
tree -L 1 /opt/prometheus
# or use ls -l
ls -l /opt/prometheus
```

**Directory descriptions:**

- `data/` - Prometheus TSDB time-series database (requires sufficient disk space)
- `rules/` - Alert and recording rule files
- `config/` - Config backup directory (for rollback)
- `consoles/` - Web console templates (optional)
- `console_libraries/` - Console library files (optional)
- `targets/` - File-based service discovery (dynamic targets)

### 5.2 Create Alertmanager Directory Structure

```bash
# Create directories required by Alertmanager
cd /opt/alertmanager
mkdir -p data config templates

# Verify directory structure
ls -l /opt/alertmanager
```

**Directory descriptions:**

- `data/` - Alertmanager state and silence data
- `config/` - Config backup directory
- `templates/` - Alert notification templates

### 5.3 Create Dedicated User (Recommended for Production)

```bash
# Create prometheus system user and group
groupadd --system prometheus
useradd -s /sbin/nologin --system -g prometheus prometheus

# Verify user creation
id prometheus
# Expected: uid=xxx(prometheus) gid=xxx(prometheus) groups=xxx(prometheus)
uid=981(prometheus) gid=980(prometheus) groups=980(prometheus)
```

### 5.4 Set Directory Permissions and Ownership

```bash
# Set owner to prometheus user
chown -R prometheus:prometheus /opt/prometheus
chown -R prometheus:prometheus /opt/alertmanager
chown -R prometheus:prometheus /opt/node_exporter

# Base directory permissions
chmod 755 /opt/prometheus /opt/alertmanager /opt/node_exporter

# Data directory permissions (stricter)
chmod 750 /opt/prometheus/data
chmod 750 /opt/alertmanager/data

# Config file permissions (prevent unauthorized modification)
chmod 640 /opt/prometheus/prometheus.yml 2>/dev/null || true
chmod 640 /opt/alertmanager/alertmanager.yml 2>/dev/null || true

# Verify permissions
ls -ld /opt/prometheus /opt/alertmanager /opt/node_exporter
ls -ld /opt/prometheus/data /opt/alertmanager/data
```

**Security best practices:**

- Run services as a dedicated system user (least privilege)
- Data directories: mode 750 (prometheus user/group only)
- Config files: mode 640 (prometheus read/write, group read)
- Executables: mode 755 (executable by all users)

---

# Part 3: Rules and Config Deployment

## Step 6: Deploy Prometheus Alert Rules

### 6.1 Copy Alert Rule Files

Copy rules from the config bundle to Prometheus `rules/`:

```bash
# Assume current directory is 04_prometheus.example.com/
cp 10-Prometheus规则文件/linux_filesystem_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_network_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_performance_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_service_alerts.yml /opt/prometheus/rules/

chown prometheus:prometheus /opt/prometheus/rules/*.yml
chmod 774 /opt/prometheus/rules/*.yml

ls -lh /opt/prometheus/rules/
```

**Alert rule files:**

- `linux_filesystem_alerts.yml` - Filesystem alerts (disk usage, inode usage, etc.)
- `linux_network_alerts.yml` - Network alerts (errors, packet loss, etc.)
- `linux_performance_alerts.yml` - Performance alerts (CPU, memory, load, etc.)
- `linux_service_alerts.yml` - Availability alerts (node down, exporter down, etc.)

### 6.2 Deploy Prometheus Main Config

```bash
cp /opt/prometheus/prometheus.yml /opt/prometheus/config/prometheus.yml.original
cp prometheus.yml /opt/prometheus/prometheus.yml

chown prometheus:prometheus /opt/prometheus/prometheus.yml
chmod 770 /opt/prometheus/prometheus.yml
```

**Key `prometheus.yml` settings:**

1. **global**
   - `scrape_interval: 15s`
   - `evaluation_interval: 15s`
   - `scrape_timeout: 10s`
   - `external_labels`
2. **alerting**
   - Alertmanager: `localhost:9093`
   - API version: `v2` (recommended)
3. **rule_files**
   - All alert rules (absolute paths)
4. **scrape_configs**
   - `prometheus` - self-monitoring
   - `alertmanager` - Alertmanager monitoring
   - `node_exporter_prometheus` - Node Exporter on Prometheus server
   - `node_exporter_chaos` - Node Exporter on Chaos server

### 6.3 Validate Prometheus Config

```bash
cd /opt/prometheus
./promtool check config prometheus.yml
```

**Expected:**

```
Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax
```

### 6.4 Validate Alert Rules

```bash
cd /opt/prometheus
./promtool check rules rules/*.yml
```

**Expected example:**

```
Checking rules/linux_filesystem_alerts.yml
 SUCCESS: 7 rules found
...
Checking rules/linux_service_alerts.yml
 SUCCESS: 5 rules found
```

### 6.5 Test a Single Rule File (Optional)

```bash
./promtool check rules rules/linux_performance_alerts.yml
./promtool check rules rules/linux_performance_alerts.yml
```

## Step 7: Deploy Alertmanager Config

### 7.1 Deploy Alertmanager Config File

```bash
cp /opt/alertmanager/alertmanager.yml /opt/alertmanager/config/alertmanager.yml.original
cp alertmanager.yml /opt/alertmanager/alertmanager.yml

chown prometheus:prometheus /opt/alertmanager/alertmanager.yml
chmod 770 /opt/alertmanager/alertmanager.yml
```

**Key `alertmanager.yml` settings:**

1. **global** - `resolve_timeout: 5m`
2. **route** - `group_by`, `group_wait`, `group_interval`, `repeat_interval`; sub-routes for Critical / Warning / Info/Unknown
3. **receivers** - `default`; `EDA` webhook → `http://10.210.65.24:5000/alerts`, `send_resolved: true`
4. **inhibit_rules** - Critical suppresses Warning on same instance; InstanceDown suppresses other alerts on that instance

### 7.2 Validate Alertmanager Config

```bash
cd /opt/alertmanager
./amtool check-config alertmanager.yml
```

**Expected:**

```
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 2 inhibit rules
 - 2 receivers
 - 0 templates
```

### 7.3 Test Alertmanager Config (Optional)

```bash
./amtool config routes --config.file=alertmanager.yml

./amtool config routes test \
  --config.file=alertmanager.yml \
  --tree \
  alertname=HighCPUUsage severity=critical
```

**Expected:**

```
Routing tree:
└── default-route  receiver: default
    └── {severity="critical"}  receiver: EDA
```

## Step 8: Node Exporter Configuration

Node Exporter is configured via **startup flags** — no config file.

### 8.1 Collector Overview

**Default collectors (keep):**

- `cpu`, `diskstats`, `filesystem`, `loadavg`, `meminfo`, `netdev`, `netstat`, `stat`, `time`, `uname`

**Optional collectors:**

- `systemd` (`--collector.systemd`)
- `processes` (`--collector.processes`)
- `tcpstat` (`--collector.tcpstat`)

**Recommended filters:**

- Exclude virtual filesystem mount points (`/proc`, `/sys`, etc.)
- Exclude Docker/container network interfaces

### 8.2 Startup Parameters

Applied in the **systemd unit** (Step 9.3):

```bash
--web.listen-address=:9100
--collector.filesystem.mount-points-exclude="^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)"
--collector.netclass.ignored-devices="^(veth.*|br-.*|docker.*)$"
--collector.netdev.device-exclude="^(veth.*|br-.*|docker.*)$"
```

**Parameter descriptions:**

- `--web.listen-address=:9100` - Listen on port 9100
- `--collector.filesystem.mount-points-exclude` - Exclude virtual FS mounts
- `--collector.netclass.ignored-devices` - Exclude virtual network devices
- `--collector.netdev.device-exclude` - Exclude virtual network interfaces

---

# Part 4: systemd Service Configuration

## Step 9: Create systemd Service Files

Follow systemd best practices for standardized service management.

### 9.1 Create Prometheus Service

Create `/etc/systemd/system/prometheus.service`:

```bash
cat > /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus

# Prometheus 工作目录
WorkingDirectory=/opt/prometheus

# 环境变量
Environment="GOMAXPROCS=2"

# Prometheus 启动命令
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/data \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=10GB \
  --web.console.templates=/opt/prometheus/consoles \
  --web.console.libraries=/opt/prometheus/console_libraries \
  --web.listen-address=:9090 \
  --web.max-connections=512 \
  --web.read-timeout=5m \
  --web.enable-lifecycle \
  --web.enable-admin-api \
  --log.level=info \
  --log.format=logfmt

# 配置热重载支持
ExecReload=/bin/kill -HUP $MAINPID

# 优雅停止
KillMode=process
KillSignal=SIGTERM
TimeoutStopSec=30s

# 重启策略
Restart=on-failure
RestartSec=5s


# 资源限制
LimitNOFILE=65536
LimitNPROC=4096
LimitMEMLOCK=0

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=prometheus

[Install]
WantedBy=multi-user.target
EOF
```

**Prometheus service parameters:**

**Storage:**

- `--storage.tsdb.retention.time=15d`
- `--storage.tsdb.retention.size=10GB`

**Web:**

- `--web.listen-address=:9090`
- `--web.max-connections=512`
- `--web.read-timeout=5m`
- `--web.enable-lifecycle`
- `--web.enable-admin-api`

**Logging:**

- `--log.level=info`
- `--log.format=logfmt`

**Security hardening (documented):**

- `User=prometheus`
- `NoNewPrivileges=true`
- `ProtectHome=true`
- `ProtectSystem=strict`
- `ReadWritePaths=/opt/prometheus/data`
- `PrivateTmp=true`

**Resource limits:**

- `LimitNOFILE=65536`
- `LimitNPROC=4096`
- `GOMAXPROCS=2`

### 9.2 Create Alertmanager Service

Create `/etc/systemd/system/alertmanager.service`:

```bash
cat > /etc/systemd/system/alertmanager.service <<'EOF'
[Unit]
Description=Prometheus Alertmanager
Documentation=https://prometheus.io/docs/alerting/latest/alertmanager/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus

# Alertmanager 工作目录
WorkingDirectory=/opt/alertmanager

# Alertmanager 启动命令
ExecStart=/opt/alertmanager/alertmanager \
  --config.file=/opt/alertmanager/alertmanager.yml \
  --storage.path=/opt/alertmanager/data \
  --data.retention=120h \
  --web.listen-address=:9093 \
  --web.external-url=http://10.210.65.174:9093 \
  --cluster.listen-address= \
  --log.level=info \
  --log.format=logfmt

# 配置热重载支持
ExecReload=/bin/kill -HUP $MAINPID

# 优雅停止
KillMode=process
KillSignal=SIGTERM
TimeoutStopSec=10s

# 重启策略
Restart=on-failure
RestartSec=5s


# 资源限制
LimitNOFILE=65536
LimitNPROC=2048

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=alertmanager

[Install]
WantedBy=multi-user.target
EOF
```

**Alertmanager service parameters:**

- `--storage.path=/opt/alertmanager/data`
- `--data.retention=120h`
- `--web.listen-address=:9093`
- `--web.external-url=http://10.210.65.174:9093`
- `--cluster.listen-address=` (single instance, cluster disabled)
- `--log.level=info` · `--log.format=logfmt`

### 9.3 Create Node Exporter Service

Create `/etc/systemd/system/node_exporter.service`:

```bash
cat > /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Prometheus Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus

# Node Exporter 工作目录
WorkingDirectory=/opt/node_exporter

# Node Exporter 启动命令
ExecStart=/opt/node_exporter/node_exporter \
  --web.listen-address=:9100 \
  --web.telemetry-path=/metrics \
  --collector.filesystem.mount-points-exclude="^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)" \
  --collector.netclass.ignored-devices="^(veth.*|br-.*|docker.*)$" \
  --collector.netdev.device-exclude="^(veth.*|br-.*|docker.*)$" \
  --log.level=info \
  --log.format=logfmt

# 优雅停止
KillMode=process
KillSignal=SIGTERM
TimeoutStopSec=5s

# 重启策略
Restart=on-failure
RestartSec=5s


# 资源限制
LimitNOFILE=65536
LimitNPROC=1024

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

**Node Exporter service parameters:**

- `--web.listen-address=:9100` · `--web.telemetry-path=/metrics`
- Collector filters for filesystem, netclass, netdev
- `--log.level=info` · `--log.format=logfmt`

### 9.4 Set systemd Service File Permissions

```bash
chmod 644 /etc/systemd/system/prometheus.service
chmod 644 /etc/systemd/system/alertmanager.service
chmod 644 /etc/systemd/system/node_exporter.service

ls -l /etc/systemd/system/{prometheus,alertmanager,node_exporter}.service
```

---

# Part 5: Start and Verify Services

## Step 10: Start and Verify Services

### 10.1 Reload systemd Configuration

```bash
systemctl daemon-reload
systemctl list-unit-files | grep -E 'prometheus|alertmanager|node_exporter'
```

**Expected:**

```
alertmanager.service      disabled
node_exporter.service     disabled
prometheus.service        disabled
```

### 10.2 Start All Services on Prometheus Server (10.210.65.174)

```bash
systemctl start node_exporter
systemctl start alertmanager
systemctl start prometheus

sleep 2

systemctl status node_exporter --no-pager
systemctl status alertmanager --no-pager
systemctl status prometheus --no-pager
```

**Verify service state:**

```bash
systemctl is-active prometheus alertmanager node_exporter
# Expected:
# active
# active
# active

systemctl is-failed prometheus alertmanager node_exporter
# Expected:
# active
# active
# active
```

### 10.3 Verify Processes and Ports

```bash
ps aux | grep -E 'prometheus|alertmanager|node_exporter' | grep -v grep

ss -tlnp | grep -E '9090|9093|9100'
# Expected:
# LISTEN ... *:9090 ... prometheus
# LISTEN ... *:9093 ... alertmanager
# LISTEN ... *:9100 ... node_exporter

netstat -tlnp | grep -E '9090|9093|9100'
```

### 10.4 Enable Boot Autostart

```bash
systemctl enable node_exporter
systemctl enable alertmanager
systemctl enable prometheus

systemctl is-enabled prometheus alertmanager node_exporter
# Expected:
# enabled
# enabled
# enabled
```

### 10.5 View Service Logs

```bash
journalctl -u prometheus --no-pager -n 50
journalctl -u alertmanager --no-pager -n 50
journalctl -u node_exporter --no-pager -n 50

# journalctl -u prometheus -f
```

**Key log messages:**

- Prometheus: Server is ready to receive web requests
- Alertmanager: Listening on :9093
- Node Exporter: Listening on :9100

## Step 11: Deploy Node Exporter on Chaos Server (10.210.65.148)

### 11.1 Prepare Deployment Files

On Prometheus server (10.210.65.174):

```bash
mkdir -p /tmp/node_exporter_deploy

cp -r /opt/node_exporter /tmp/node_exporter_deploy/
cp /etc/systemd/system/node_exporter.service /tmp/node_exporter_deploy/

cat > /tmp/node_exporter_deploy/deploy.sh <<'DEPLOY_EOF'
#!/bin/bash
set -e

echo "=== Node Exporter 部署脚本 ==="

# 创建 prometheus 用户
if ! id prometheus &>/dev/null; then
    echo "创建 prometheus 用户..."
    groupadd --system prometheus
    useradd -s /sbin/nologin --system -g prometheus prometheus
fi

# 复制文件
echo "复制 Node Exporter 文件..."
cp -r node_exporter /opt/
cp node_exporter.service /etc/systemd/system/

# 设置权限
echo "设置权限..."
chown -R prometheus:prometheus /opt/node_exporter
chmod 755 /opt/node_exporter
chmod 644 /etc/systemd/system/node_exporter.service

# 重载 systemd
echo "重载 systemd 配置..."
systemctl daemon-reload

# 启动服务
echo "启动 Node Exporter 服务..."
systemctl start node_exporter
systemctl enable node_exporter

# 验证服务
echo "验证服务状态..."
systemctl status node_exporter --no-pager

echo "=== 部署完成 ==="
DEPLOY_EOF

chmod +x /tmp/node_exporter_deploy/deploy.sh
```

### 11.2 Transfer Files to Chaos Server

```bash
scp -r /tmp/node_exporter_deploy root@10.210.65.148:/tmp/

ssh root@10.210.65.148 "ls -l /tmp/node_exporter_deploy/"
```

### 11.3 Run Deployment on Chaos Server

On Chaos server (10.210.65.148):

```bash
ssh root@10.210.65.148

cd /tmp/node_exporter_deploy

./deploy.sh

systemctl status node_exporter
ss -tlnp | grep 9100
curl -s http://localhost:9100/metrics | head -20
```

### 11.4 Verify Chaos Server Node Exporter

```bash
systemctl is-active node_exporter
systemctl is-enabled node_exporter

curl http://localhost:9100/metrics | grep "node_exporter_build_info"

journalctl -u node_exporter --no-pager -n 30
```

### 11.5 Test Connectivity from Prometheus Server

On Prometheus server:

```bash
curl -s http://10.210.65.148:9100/metrics | head -20

# If connection fails, check firewall (next step)
```

## Step 12: Configure Firewall

### 12.1 Check Firewall Status

```bash
systemctl status firewalld
firewall-cmd --list-all
```

### 12.2 Configure Prometheus Server (10.210.65.174) Firewall

```bash
firewall-cmd --permanent --add-port=9090/tcp  # Prometheus Web UI
firewall-cmd --permanent --add-port=9093/tcp  # Alertmanager Web UI
firewall-cmd --permanent --add-port=9100/tcp  # Node Exporter

firewall-cmd --reload

firewall-cmd --list-ports
# Expected: 9090/tcp 9093/tcp 9100/tcp
```

**Production recommendation (source-restricted):**

```bash
firewall-cmd --permanent --remove-port=9090/tcp
firewall-cmd --permanent --remove-port=9093/tcp
firewall-cmd --permanent --remove-port=9100/tcp

firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9090" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9093" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9100" accept'

# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.66.208.0/24" port protocol="tcp" port="9090" accept'

firewall-cmd --reload

firewall-cmd --list-rich-rules
```

---

## Attachments: Prometheus and Alertmanager Config Files

### prometheus.yml

```yaml
# Prometheus 配置文件
# 基于 Prometheus 最佳实践: https://prometheus.io/docs/practices/

global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s

  external_labels:
    cluster: 'aiops-demo'
    environment: 'production'
    prometheus_instance: 'prometheus.example.com'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'
      timeout: 10s
      api_version: v2

rule_files:
  - '/opt/prometheus/rules/linux_filesystem_alerts.yml'
  - '/opt/prometheus/rules/linux_network_alerts.yml'
  - '/opt/prometheus/rules/linux_performance_alerts.yml'
  - '/opt/prometheus/rules/linux_service_alerts.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance_type: 'prometheus_server'
          role: 'monitoring'

  - job_name: 'alertmanager'
    static_configs:
      - targets: ['localhost:9093']
        labels:
          instance_type: 'alertmanager'
          role: 'monitoring'

  - job_name: 'node_exporter_prometheus'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance_name: 'prometheus.example.com'
          instance_type: 'prometheus_server'
          role: 'monitoring'
          os: 'RHEL 9.5'
    scrape_interval: 15s
    scrape_timeout: 10s

  - job_name: 'node_exporter_chaos'
    static_configs:
      - targets: ['10.210.65.148:9100']
        labels:
          instance_name: 'chaos.example.com'
          instance_type: 'managed_server'
          role: 'application'
          os: 'RHEL 9.4'
    scrape_interval: 15s
    scrape_timeout: 10s

  # - job_name: 'node_exporter_file_sd'
  #   file_sd_configs:
  #     - files:
  #         - '/opt/prometheus/targets/nodes_*.yml'
  #       refresh_interval: 5m
```

### alertmanager.yml – Works with AAP UI

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'EDA'
  group_by: ['alertname', 'cluster', 'service', 'instance']
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 5m

receivers:
  - name: 'EDA'
    webhook_configs:
    - url: 'https://aap26.example.com:443/eda-event-streams/api/eda/v1/external_event_stream/a8e18ec4-ede4-43be-a3d9-dbc83701b811/post/'
      send_resolved: false
      http_config:
        basic_auth:
          username: "event_stream"
          password: "redhat"
        tls_config:
          insecure_skip_verify: true
      max_alerts: 0
```

### alertmanager.yml – Works from CLI / CMD

```yaml
# Alertmanager 配置文件
# 基于 Prometheus 最佳实践: https://prometheus.io/docs/alerting/latest/configuration/

global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service', 'instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

  routes:
    - match:
        severity: critical
      receiver: 'EDA'
      group_wait: 5s
      group_interval: 5s
      repeat_interval: 3h
      continue: false

    - match:
        severity: warning
      receiver: 'EDA'
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      continue: false

    - match_re:
        severity: '^(info|unknown)$'
      receiver: 'default'
      repeat_interval: 24h

receivers:
  - name: 'default'

  - name: 'EDA'
    webhook_configs:
      - url: 'http://10.210.65.24:5000/alerts'
        send_resolved: true
        http_config:
          follow_redirects: true
        max_alerts: 0

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']

  - source_match:
      alertname: 'InstanceDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']

# time_intervals:
#   - name: 'maintenance_window'
#     time_intervals:
#       - weekdays: ['saturday', 'sunday']
#         times:
#           - start_time: '00:00'
#             end_time: '06:00'
```
### 0814 updated Prometheus rules

#### linux_performance_alerts.yml
```yaml
# Linux 性能告警规则
# 监控 CPU、内存、磁盘 I/O 等系统性能指标

groups:
  - name: linux.performance.alerts
    interval: 30s
    rules:
      # 高系统 CPU 使用率 - Warning 级别
      - alert: HighSystemCpuUsage
        expr: |
          (
            sum without (cpu) (
              irate(node_cpu_seconds_total{mode!="idle"}[5m])
            )
          ) > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高系统 CPU 使用率: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 CPU 使用率持续 5 分钟高于 80%，当前值: {{ $value | humanizePercentage }}。"

      # 高系统 CPU 使用率 - Critical 级别
      - alert: CriticalSystemCpuUsage
        expr: |
          (
            sum without (cpu) (
              irate(node_cpu_seconds_total{mode!="idle"}[1m])
            )
          ) > 0.95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "严重 CPU 使用率: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 CPU 使用率持续 1 分钟高于 95%，当前值: {{ $value | humanizePercentage }}。系统可能无响应！"

      # 高 CPU 负载 - Warning 级别
      - alert: HighCpuLoad
        expr: |
          (
            node_load5 / count without (cpu, mode) (node_cpu_seconds_total{mode="idle"}) > 1.5
          )
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "高 CPU 负载: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 5 分钟平均负载超过 CPU 核心数的 1.5 倍，当前值: {{ $value | humanize }}。"

      # 高 CPU 负载 - Critical 级别
      - alert: CriticalCpuLoad
        expr: |
          (
            node_load5 / count without (cpu, mode) (node_cpu_seconds_total{mode="idle"}) > 2.0
          )
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "严重 CPU 负载: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 5 分钟平均负载超过 CPU 核心数的 2 倍，当前值: {{ $value | humanize }}。"

      # 高内存使用率 - Warning 级别
      - alert: HighMemoryUsage
        expr: |
          (
            1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
          ) > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高内存使用率: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的内存使用率持续 5 分钟高于 80%，当前值: {{ $value | humanizePercentage }}。"

      # 高内存使用率 - Critical 级别
      - alert: CriticalMemoryUsage
        expr: |
          (
            1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
          ) > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "严重内存使用率: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的内存使用率持续 2 分钟高于 95%，当前值: {{ $value | humanizePercentage }}。系统可能出现 OOM！"

      # Swap 使用率过高
      - alert: HighSwapUsage
        expr: |
          (
            (node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / node_memory_SwapTotal_bytes
          ) > 0.50
          and
          node_memory_SwapTotal_bytes > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Swap 使用率过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 Swap 使用率持续 5 分钟高于 50%，当前值: {{ $value | humanizePercentage }}。可能存在内存不足问题。"

      # 磁盘 I/O 延迟过高
      - alert: HighDiskIOLatency
        expr: |
          (
            rate(node_disk_io_time_seconds_total{device!~"dm-.*"}[5m]) > 0.8
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "磁盘 I/O 延迟过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的磁盘 {{ $labels.device }} I/O 延迟过高，当前 I/O 利用率: {{ $value | humanizePercentage }}。"

      # 磁盘读写速率异常高
      - alert: HighDiskIOPS
        expr: |
          (
            rate(node_disk_reads_completed_total{device!~"dm-.*"}[5m]) +
            rate(node_disk_writes_completed_total{device!~"dm-.*"}[5m])
          ) > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "磁盘 IOPS 异常高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的磁盘 {{ $labels.device }} IOPS 异常高，当前值: {{ $value | humanize }} ops/s。"

      # 上下文切换率过高
      - alert: HighContextSwitching
        expr: |
          (
            rate(node_context_switches_total[5m]) > 100000
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "上下文切换率过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的上下文切换率过高，当前值: {{ $value | humanize }} 次/秒。可能存在大量线程竞争。"
```

#### linux_network_alerts.yml
```yaml
# Linux 网络告警规则
# 监控网络错误、丢包和连接状态

groups:
  - name: linux.network.alerts
    interval: 30s
    rules:
      # 网络接收丢包率过高
      - alert: HighNetworkReceiveDrop
        expr: |
          (
            rate(node_network_receive_drop_total{device!~"lo|veth.*|docker.*|br-.*"}[5m]) > 10
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "网络接收丢包率过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的网卡 {{ $labels.device }} 接收丢包率过高，当前速率: {{ $value | humanize }} 包/秒。"

      # 网络发送丢包率过高
      - alert: HighNetworkTransmitDrop
        expr: |
          (
            rate(node_network_transmit_drop_total{device!~"lo|veth.*|docker.*|br-.*"}[5m]) > 10
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "网络发送丢包率过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的网卡 {{ $labels.device }} 发送丢包率过高，当前速率: {{ $value | humanize }} 包/秒。"

      # 网络接收错误率过高
      - alert: HighNetworkReceiveErrors
        expr: |
          (
            rate(node_network_receive_errs_total{device!~"lo|veth.*|docker.*|br-.*"}[5m]) > 10
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "网络接收错误率过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的网卡 {{ $labels.device }} 接收错误率过高，当前速率: {{ $value | humanize }} 包/秒。"

      # 网络发送错误率过高
      - alert: HighNetworkTransmitErrors
        expr: |
          (
            rate(node_network_transmit_errs_total{device!~"lo|veth.*|docker.*|br-.*"}[5m]) > 10
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "网络发送错误率过高: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的网卡 {{ $labels.device }} 发送错误率过高，当前速率: {{ $value | humanize }} 包/秒。"

      # 网络接口状态异常
      - alert: NetworkInterfaceDown
        expr: |
          (
            node_network_up{device!~"lo|veth.*|docker.*|br-.*"} == 0
          )
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "网络接口异常: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的网卡 {{ $labels.device }} 处于 DOWN 状态。"

      # TCP 连接处于 TIME_WAIT 状态过多
      - alert: HighTCPTimeWaitConnections
        expr: |
          (
            node_netstat_Tcp_CurrEstab > 0
            and
            node_sockstat_TCP_tw / node_netstat_Tcp_CurrEstab > 2
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "TCP TIME_WAIT 连接过多: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 TCP TIME_WAIT 连接数是当前建立连接数的 {{ $value | humanize }} 倍。可能存在连接泄漏或短连接过多的情况。"

      # TCP 连接数接近上限
      - alert: TCPConnectionsNearLimit
        expr: |
          (
            node_netstat_Tcp_CurrEstab / node_sockstat_sockets_used > 0.8
          )
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "TCP 连接数接近上限: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 TCP 连接数占总 socket 使用数的 {{ $value | humanizePercentage }}，接近上限。"
```

#### linux_filesystem_alerts.yml
```yaml
# Linux 文件系统告警规则
# 监控磁盘空间使用和 inode 使用情况

groups:
  - name: linux.filesystem.alerts
    interval: 30s
    rules:
      # 文件系统空间不足告警 - Warning 级别
      - alert: FilesystemSpaceFillingUp
        expr: |
          (
            node_filesystem_avail_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
            /
            node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
          ) * 100 < 20
          and
          node_filesystem_readonly{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "文件系统空间不足: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的挂载点 {{ $labels.mountpoint }} (设备: {{ $labels.device }}) 剩余空间不足 20%，当前可用: {{ $value | humanizePercent
age }}。"

      # 文件系统空间严重不足告警 - Critical 级别
      - alert: FilesystemSpaceCritical
        expr: |
          (
            node_filesystem_avail_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
            /
            node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
          ) * 100 < 10
          and
          node_filesystem_readonly{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "文件系统空间严重不足: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的挂载点 {{ $labels.mountpoint }} (设备: {{ $labels.device }}) 剩余空间不足 10%，当前可用: {{ $value | humanizePercent
age }}。请立即处理！"

      # 文件系统预测将在 24 小时内耗尽
      - alert: FilesystemWillFillIn24h
        expr: |
          (
            predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}[6h], 24 * 3600) < 0
            and
            node_filesystem_avail_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} < 0.4
0
          )
          and
          node_filesystem_readonly{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} == 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "文件系统空间预测告警: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的挂载点 {{ $labels.mountpoint }} (设备: {{ $labels.device }}) 基于过去 6 小时的趋势，预计在 24 小时内空间将耗尽。当前
可用: {{ $value | humanize }}B。"

      # Inode 使用率过高 - Warning 级别
      - alert: FilesystemInodesFillingUp
        expr: |
          (
            node_filesystem_files_free{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
            /
            node_filesystem_files{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
          ) * 100 < 20
          and
          node_filesystem_readonly{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "文件系统 Inode 不足: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的挂载点 {{ $labels.mountpoint }} (设备: {{ $labels.device }}) 剩余 inode 不足 20%，当前可用: {{ $value | humanizePerc
entage }}。"

      # Inode 使用率严重过高 - Critical 级别
      - alert: FilesystemInodesCritical
        expr: |
          (
            node_filesystem_files_free{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
            /
            node_filesystem_files{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"}
          ) * 100 < 5
          and
          node_filesystem_readonly{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "文件系统 Inode 严重不足: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的挂载点 {{ $labels.mountpoint }} (设备: {{ $labels.device }}) 剩余 inode 不足 5%，当前可用: {{ $value | humanizePerce
ntage }}。请立即处理！"

      # 文件系统只读
      - alert: FilesystemReadOnly
        expr: |
          node_filesystem_readonly{fstype!~"tmpfs|devtmpfs|vfat|overlay|squashfs"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "文件系统变为只读: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的挂载点 {{ $labels.mountpoint }} (设备: {{ $labels.device }}) 已变为只读状态。可能存在磁盘故障或文件系统错误。"

```

#### linux_service_alerts.yml

```yaml
# Linux 服务和实例状态告警规则
# 监控关键系统服务和实例可用性

groups:
  - name: linux.service.alerts
    interval: 30s
    rules:
      # 实例宕机告警 - 最高优先级
      - alert: InstanceDown
        expr: |
          up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "实例宕机: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} (job: {{ $labels.job }}) 已宕机超过 1 分钟，Prometheus 无法抓取其指标。"

      # Node Exporter 不可用
      - alert: NodeExporterDown
        expr: |
          up{job=~"node_exporter.*"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node Exporter 不可用: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的 Node Exporter 不可用，无法收集系统指标。"

      # 系统服务状态异常
      - alert: SystemdServiceFailed
        expr: |
          node_systemd_unit_state{state="failed"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Systemd 服务失败: {{ $labels.name }}"
          description: "实例 {{ $labels.instance }} 上的服务 {{ $labels.name }} 处于 failed 状态。"

      # 关键系统服务未运行
      - alert: CriticalSystemdServiceInactive
        expr: |
          node_systemd_unit_state{
            state="active",
            name=~"sshd.service|NetworkManager.service|chronyd.service|rsyslog.service"
          } == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "关键服务未运行: {{ $labels.name }}"
          description: "实例 {{ $labels.instance }} 上的关键服务 {{ $labels.name }} 未处于 active 状态。"


      # 系统时钟偏移过大
      - alert: SystemClockSkewDetected
        expr: |
          abs(node_timex_offset_seconds) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "系统时钟偏移: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的系统时钟偏移超过 50ms，当前偏移: {{ $value | humanize }}s。"

      # 系统重启
      - alert: SystemRebooted
        expr: |
          node_boot_time_seconds > (time() - 600)
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "系统已重启: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 在最近 10 分钟内重启过。启动时间: {{ $value | humanizeTimestamp }}。"

      # 内核版本变更
      - alert: KernelVersionChanged
        expr: |
          changes(node_uname_info[30m]) > 0
        for: 0m
        labels:
          severity: info
        annotations:
          summary: "内核版本变更: {{ $labels.instance }}"
          description: "实例 {{ $labels.instance }} 的内核版本在最近 30 分钟内发生了变化。当前内核: {{ $labels.release }}。"

```
