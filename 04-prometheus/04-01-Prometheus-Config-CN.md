# Prometheus 监控系统部署及配置指南

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

本指南遵循 Prometheus 官方最佳实践：[https://prometheus.io/docs/introduction/overview/](https://prometheus.io/docs/introduction/overview/)

## 架构参考

| 角色 | 说明 |
| --- | --- |
| **Prometheus 服务器** | IP: `10.210.65.174` · 操作系统: Red Hat Enterprise Linux release 9.5 (Plow) · 部署组件: Prometheus + Alertmanager + Node Exporter |
| **被管理服务器（Chaos Server）** | IP: `10.210.65.148` · 操作系统: Red Hat Enterprise Linux release 9.4 (Plow) · 部署组件: Node Exporter |

Prometheus 及 Alertmanager 的配置文件见本章结尾附件。

## 本章目标

在 Prometheus 服务器与被管理服务器上，按 **Step 1～12** 完成软件包下载解压、标准化目录与配置部署、systemd 服务、启动验证、Chaos 节点 Node Exporter 部署及防火墙配置。

| 项目 | 说明 |
| --- | --- |
| **Prometheus 服务器** | `10.210.65.174` |
| **Chaos 服务器** | `10.210.65.148` |
| **软件版本** | Prometheus 3.5.1 · Alertmanager 0.31.1 · Node Exporter 1.10.2 |
| **安装路径** | `/opt/prometheus` · `/opt/alertmanager` · `/opt/node_exporter` |
| **配置源** | `04_prometheus.example.com/` |

---

## Executive Summary: 步骤一览

| 部分 | Step | 内容 |
| --- | --- | --- |
| **第一部分** | 1～4 | 软件包下载及解压 |
| **第二部分** | 5（5.1～5.4） | 标准化配置部署 |
| **第三部分** | 6～8 | 规则文件部署 |
| **第四部分** | 9（9.1～9.4） | systemd 服务配置 |
| **第五部分** | 10～12 | 启动和验证服务 |

---

# 第一部分：软件包下载及解压

## Step 1：下载 Prometheus 软件包

```bash
# 创建临时下载目录
mkdir -p /tmp/prometheus_install && cd /tmp/prometheus_install

# 下载 Prometheus（版本 3.5.1）
wget https://github.com/prometheus/prometheus/releases/download/v3.5.1/prometheus-3.5.1.linux-amd64.tar.gz

# 下载 Alertmanager（版本 0.31.1）
wget https://github.com/prometheus/alertmanager/releases/download/v0.31.1/alertmanager-0.31.1.linux-amd64.tar.gz

# 下载 Node Exporter（版本 1.10.2）
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```

## Step 2：验证下载文件（可选但推荐）

```bash
# 验证文件完整性（需要从官方获取 SHA256）
sha256sum *.tar.gz
```

## Step 3：解压到标准化目录

```bash
# 创建标准化目录并解压到指定路径
mkdir -p /opt/prometheus /opt/alertmanager /opt/node_exporter

tar -xzf /tmp/prometheus_install/prometheus-3.5.1.linux-amd64.tar.gz -C /opt/prometheus --strip-components=1
tar -xzf /tmp/prometheus_install/alertmanager-0.31.1.linux-amd64.tar.gz -C /opt/alertmanager --strip-components=1
tar -xzf /tmp/prometheus_install/node_exporter-1.10.2.linux-amd64.tar.gz -C /opt/node_exporter --strip-components=1

# 验证目录结构
ls -l /opt/prometheus
ls -l /opt/alertmanager
ls -l /opt/node_exporter
```

## Step 4：验证安装目录结构

```bash
# 验证目录
cd /opt
pwd

# 查看 Alertmanager
ls -lh /opt/alertmanager
# 预期输出：
# -rwxr-xr-x alertmanager    (可执行文件)
# -rw-r--r-- alertmanager.yml (默认配置文件，稍后替换)
# -rwxr-xr-x amtool          (管理工具)

# 查看 Node Exporter
ls -lh /opt/node_exporter
# 预期输出：
# -rwxr-xr-x node_exporter   (可执行文件)

# 查看 Prometheus
ls -lh /opt/prometheus
# 预期输出：
# -rwxr-xr-x prometheus       (主程序)
# -rwxr-xr-x promtool         (配置验证工具)
# -rw-r--r-- prometheus.yml   (默认配置文件，稍后替换)
```

---

# 第二部分：标准化配置部署

## Step 5：创建标准化目录结构

遵循 Prometheus 最佳实践，创建标准化的目录结构。

### 5.1 创建 Prometheus 目录结构

```bash
# 创建 Prometheus 运行所需目录
cd /opt/prometheus
mkdir -p data rules config consoles console_libraries targets

# 验证目录结构
tree -L 1 /opt/prometheus
# 或者使用 ls -l
ls -l /opt/prometheus
```

**目录说明：**

- `data/` - Prometheus TSDB 时序数据库存储目录（重要：需要足够的磁盘空间）
- `rules/` - 告警和记录规则文件目录
- `config/` - 配置文件版本备份目录（用于配置回滚）
- `consoles/` - Web 控制台模板目录（可选）
- `console_libraries/` - 控制台库文件目录（可选）
- `targets/` - 基于文件的服务发现配置目录（用于动态目标发现）

### 5.2 创建 Alertmanager 目录结构

```bash
# 创建 Alertmanager 运行所需目录
cd /opt/alertmanager
mkdir -p data config templates

# 验证目录结构
ls -l /opt/alertmanager
```

**目录说明：**

- `data/` - Alertmanager 状态和静默数据存储目录
- `config/` - 配置文件版本备份目录
- `templates/` - 告警通知模板目录（用于自定义告警格式）

### 5.3 创建专用用户（推荐用于生产环境）

```bash
# 创建 prometheus 系统用户和组
groupadd --system prometheus
useradd -s /sbin/nologin --system -g prometheus prometheus

# 验证用户创建
id prometheus
# 预期输出：uid=xxx(prometheus) gid=xxx(prometheus) groups=xxx(prometheus)
uid=981(prometheus) gid=980(prometheus) groups=980(prometheus)
```

### 5.4 设置目录权限和所有者

```bash
# 设置目录所有者为 prometheus 用户
chown -R prometheus:prometheus /opt/prometheus
chown -R prometheus:prometheus /opt/alertmanager
chown -R prometheus:prometheus /opt/node_exporter

# 设置基础目录权限
chmod 755 /opt/prometheus /opt/alertmanager /opt/node_exporter

# 设置数据目录权限（更严格的权限）
chmod 750 /opt/prometheus/data
chmod 750 /opt/alertmanager/data

# 设置配置文件权限（防止未授权修改）
chmod 640 /opt/prometheus/prometheus.yml 2>/dev/null || true
chmod 640 /opt/alertmanager/alertmanager.yml 2>/dev/null || true

# 验证权限
ls -ld /opt/prometheus /opt/alertmanager /opt/node_exporter
ls -ld /opt/prometheus/data /opt/alertmanager/data
```

**安全最佳实践说明：**

- 使用专用系统用户运行服务，遵循最小权限原则
- 数据目录权限 750（仅 prometheus 用户和组可访问）
- 配置文件权限 640（仅 prometheus 用户可读写，组可读）
- 可执行文件权限 755（所有用户可执行）

---

# 第三部分：规则文件部署

## Step 6：部署 Prometheus 告警规则

### 6.1 复制告警规则文件

将本目录下的规则文件复制到 Prometheus 的 rules 目录：

```bash
# 假设当前在配置文件目录（04_prometheus.example.com/）
# 复制所有告警规则文件
cp 10-Prometheus规则文件/linux_filesystem_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_network_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_performance_alerts.yml /opt/prometheus/rules/
cp 10-Prometheus规则文件/linux_service_alerts.yml /opt/prometheus/rules/

# 设置规则文件权限
chown prometheus:prometheus /opt/prometheus/rules/*.yml
chmod 774 /opt/prometheus/rules/*.yml

# 验证规则文件
ls -lh /opt/prometheus/rules/
```

**告警规则文件说明：**

- `linux_filesystem_alerts.yml` - 文件系统相关告警（磁盘使用率、inode 使用率等）
- `linux_network_alerts.yml` - 网络相关告警（网络错误、丢包率等）
- `linux_performance_alerts.yml` - 系统性能告警（CPU、内存、负载等）
- `linux_service_alerts.yml` - 服务可用性告警（节点宕机、Exporter 不可用等）

### 6.2 部署 Prometheus 主配置文件

```bash
# 备份原始配置文件
cp /opt/prometheus/prometheus.yml /opt/prometheus/config/prometheus.yml.original

# 复制优化后的配置文件
cp prometheus.yml /opt/prometheus/prometheus.yml

# 设置配置文件权限
chown prometheus:prometheus /opt/prometheus/prometheus.yml
chmod 770 /opt/prometheus/prometheus.yml
```

**Prometheus 配置文件关键配置说明：**

1. **全局配置 (global)**
   - `scrape_interval: 15s` - 抓取间隔 15 秒（平衡数据精度和存储）
   - `evaluation_interval: 15s` - 告警规则评估间隔 15 秒
   - `scrape_timeout: 10s` - 抓取超时 10 秒
   - `external_labels` - 外部标签（用于联邦和远程存储）
2. **Alertmanager 配置 (alerting)**
   - 配置 Alertmanager 地址：`localhost:9093`
   - API 版本：`v2`（推荐）
3. **规则文件 (rule_files)**
   - 引用所有告警规则文件（使用绝对路径）
4. **抓取配置 (scrape_configs)**
   - `prometheus` - Prometheus 自监控
   - `alertmanager` - Alertmanager 监控
   - `node_exporter_prometheus` - Prometheus 服务器的 Node Exporter
   - `node_exporter_chaos` - Chaos 服务器的 Node Exporter

### 6.3 验证 Prometheus 配置

```bash
# 切换到 prometheus 目录
cd /opt/prometheus

# 验证主配置文件语法
./promtool check config prometheus.yml
```

**预期输出：**

```
Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax
```

### 6.4 验证告警规则文件

```bash
# 验证所有规则文件
cd /opt/prometheus
./promtool check rules rules/*.yml
```

**预期输出示例：**

```
Checking rules/linux_filesystem_alerts.yml
 SUCCESS: 7 rules found

Checking rules/linux_network_alerts.yml
 SUCCESS: 7 rules found

Checking rules/linux_performance_alerts.yml
 SUCCESS: 8 rules found

Checking rules/linux_service_alerts.yml
 SUCCESS: 5 rules found
```

### 6.5 测试单个规则文件（可选）

```bash
# 测试特定规则文件
./promtool check rules rules/linux_performance_alerts.yml

# 查看规则详细信息
./promtool check rules rules/linux_performance_alerts.yml
```

## Step 7：部署 Alertmanager 配置

### 7.1 部署 Alertmanager 配置文件

```bash
# 备份原始配置文件
cp /opt/alertmanager/alertmanager.yml /opt/alertmanager/config/alertmanager.yml.original

# 复制优化后的配置文件
cp alertmanager.yml /opt/alertmanager/alertmanager.yml

# 设置配置文件权限
chown prometheus:prometheus /opt/alertmanager/alertmanager.yml
chmod 770 /opt/alertmanager/alertmanager.yml
```

**Alertmanager 配置文件关键配置说明：**

1. **全局配置 (global)**
   - `resolve_timeout: 5m` - 告警自动解决超时时间
2. **路由配置 (route)**
   - `group_by: ['alertname', 'cluster', 'service', 'instance']` - 告警分组标签
   - `group_wait: 10s` - 首次告警等待时间（聚合同组告警）
   - `group_interval: 10s` - 同组告警发送间隔
   - `repeat_interval: 12h` - 相同告警重复发送间隔
   - **子路由：**
     - Critical 级别：5秒等待，3小时重复间隔
     - Warning 级别：10秒等待，12小时重复间隔
     - Info/Unknown 级别：24小时重复间隔
3. **接收器配置 (receivers)**
   - `default` - 默认接收器（低优先级告警）
   - `EDA` - Event-Driven Ansible Webhook 接收器
     - URL: `http://10.210.65.24:5000/alerts`
     - 发送已解决的告警：`send_resolved: true`
4. **抑制规则 (inhibit_rules)**
   - Critical 告警抑制 Warning 告警（同一实例）
   - InstanceDown 告警抑制该实例的所有其他告警

### 7.2 验证 Alertmanager 配置

```bash
# 切换到 alertmanager 目录
cd /opt/alertmanager

# 验证配置文件
./amtool check-config alertmanager.yml
```

**预期输出：**

```
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 2 inhibit rules
 - 2 receivers
 - 0 templates
```

### 7.3 测试 Alertmanager 配置（可选）

```bash
# 查看路由配置
./amtool config routes --config.file=alertmanager.yml

# 测试告警路由（模拟告警）
./amtool config routes test \
  --config.file=alertmanager.yml \
  --tree \
  alertname=HighCPUUsage severity=critical
```

**预期输出：**

```
Routing tree:
└── default-route  receiver: default
    └── {severity="critical"}  receiver: EDA
```

## Step 8：配置 Node Exporter

Node Exporter 通过启动参数进行配置，无需配置文件。

### 8.1 Node Exporter 采集器说明

**默认启用的采集器（推荐保留）：**

- `cpu` - CPU 统计信息
- `diskstats` - 磁盘 I/O 统计
- `filesystem` - 文件系统使用情况
- `loadavg` - 系统负载
- `meminfo` - 内存使用情况
- `netdev` - 网络设备统计
- `netstat` - 网络连接统计
- `stat` - 系统统计信息
- `time` - 系统时间
- `uname` - 系统信息

**常用可选采集器：**

- `systemd` - systemd 服务状态（使用 `--collector.systemd`）
- `processes` - 进程统计（使用 `--collector.processes`）
- `tcpstat` - TCP 连接状态（使用 `--collector.tcpstat`）

**推荐的过滤配置：**

- 排除虚拟文件系统挂载点（避免采集 `/proc`、`/sys` 等）
- 排除 Docker/容器网络接口（减少无用指标）

### 8.2 Node Exporter 启动参数

Node Exporter 将通过 systemd 服务启动，参数配置在服务文件中：

```bash
--web.listen-address=:9100
--collector.filesystem.mount-points-exclude="^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)"
--collector.netclass.ignored-devices="^(veth.*|br-.*|docker.*)$"
--collector.netdev.device-exclude="^(veth.*|br-.*|docker.*)$"
```

**参数说明：**

- `--web.listen-address=:9100` - 监听端口 9100
- `--collector.filesystem.mount-points-exclude` - 排除虚拟文件系统挂载点
- `--collector.netclass.ignored-devices` - 排除虚拟网络设备（容器网桥）
- `--collector.netdev.device-exclude` - 排除虚拟网络接口

---

# 第四部分：systemd 服务配置

## Step 9：创建 systemd 服务文件

遵循 systemd 最佳实践，配置标准化的服务管理。

### 9.1 创建 Prometheus 服务

创建 `/etc/systemd/system/prometheus.service`：

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

**Prometheus 服务参数说明：**

**存储配置：**

- `--storage.tsdb.retention.time=15d` - 数据保留 15 天
- `--storage.tsdb.retention.size=10GB` - 数据最大 10GB（先达到哪个限制就触发清理）

**Web 配置：**

- `--web.listen-address=:9090` - 监听所有接口的 9090 端口
- `--web.max-connections=512` - 最大并发连接数
- `--web.read-timeout=5m` - HTTP 读取超时
- `--web.enable-lifecycle` - 启用热重载功能（通过 POST `/-/reload`）
- `--web.enable-admin-api` - 启用管理 API（删除时序数据、快照等）

**日志配置：**

- `--log.level=info` - 日志级别（debug/info/warn/error）
- `--log.format=logfmt` - 日志格式（logfmt/json）

**安全加固：**

- `User=prometheus` - 使用专用用户运行
- `NoNewPrivileges=true` - 禁止提升权限
- `ProtectHome=true` - 保护用户主目录
- `ProtectSystem=strict` - 保护系统目录
- `ReadWritePaths=/opt/prometheus/data` - 仅允许写入数据目录
- `PrivateTmp=true` - 使用私有临时目录

**资源限制：**

- `LimitNOFILE=65536` - 最大打开文件数
- `LimitNPROC=4096` - 最大进程数
- `GOMAXPROCS=2` - Go 运行时最大 CPU 核心数

### 9.2 创建 Alertmanager 服务

创建 `/etc/systemd/system/alertmanager.service`：

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

**Alertmanager 服务参数说明：**

**存储配置：**

- `--storage.path=/opt/alertmanager/data` - 数据存储路径
- `--data.retention=120h` - 数据保留时间 120 小时（5 天）

**Web 配置：**

- `--web.listen-address=:9093` - 监听所有接口的 9093 端口
- `--web.external-url=http://10.210.65.174:9093` - 外部访问 URL（用于告警通知中的链接）

**集群配置：**

- `--cluster.listen-address=` - 单实例模式，不启用集群（留空禁用）

**日志配置：**

- `--log.level=info` - 日志级别
- `--log.format=logfmt` - 日志格式

### 9.3 创建 Node Exporter 服务

创建 `/etc/systemd/system/node_exporter.service`：

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

**Node Exporter 服务参数说明：**

**Web 配置：**

- `--web.listen-address=:9100` - 监听端口 9100
- `--web.telemetry-path=/metrics` - 指标暴露路径

**采集器过滤：**

- `--collector.filesystem.mount-points-exclude` - 排除虚拟文件系统（`/proc`、`/sys`、Docker 卷等）
- `--collector.netclass.ignored-devices` - 排除虚拟网络设备类（Docker 网桥）
- `--collector.netdev.device-exclude` - 排除虚拟网络接口（veth、br-*）

**日志配置：**

- `--log.level=info` - 日志级别
- `--log.format=logfmt` - 日志格式

### 9.4 设置 systemd 服务文件权限

```bash
# 设置服务文件权限
chmod 644 /etc/systemd/system/prometheus.service
chmod 644 /etc/systemd/system/alertmanager.service
chmod 644 /etc/systemd/system/node_exporter.service

# 验证服务文件
ls -l /etc/systemd/system/{prometheus,alertmanager,node_exporter}.service
```

---

# 第五部分：启动和验证服务

## Step 10：启动和验证服务

### 10.1 重载 systemd 配置

```bash
# 重载 systemd 管理器配置
systemctl daemon-reload

# 验证服务单元文件加载
systemctl list-unit-files | grep -E 'prometheus|alertmanager|node_exporter'
```

**预期输出：**

```
alertmanager.service      disabled
node_exporter.service     disabled
prometheus.service        disabled
```

### 10.2 在 Prometheus 服务器 (10.210.65.174) 上启动所有服务

```bash
# 启动服务（按依赖顺序启动）
systemctl start node_exporter
systemctl start alertmanager
systemctl start prometheus

# 等待 2 秒让服务完全启动
sleep 2

# 查看服务状态
systemctl status node_exporter --no-pager
systemctl status alertmanager --no-pager
systemctl status prometheus --no-pager
```

**验证服务运行状态：**

```bash
# 检查服务是否处于 active (running) 状态
systemctl is-active prometheus alertmanager node_exporter
# 预期输出：
# active
# active
# active

# 检查服务是否有错误
systemctl is-failed prometheus alertmanager node_exporter
# 预期输出：
# active
# active
# active
```

### 10.3 验证进程和端口

```bash
# 查看进程
ps aux | grep -E 'prometheus|alertmanager|node_exporter' | grep -v grep

# 查看监听端口
ss -tlnp | grep -E '9090|9093|9100'
# 预期输出：
# LISTEN 0  128  *:9090  *:*  users:(("prometheus",pid=xxx,fd=x))
# LISTEN 0  128  *:9093  *:*  users:(("alertmanager",pid=xxx,fd=x))
# LISTEN 0  128  *:9100  *:*  users:(("node_exporter",pid=xxx,fd=x))

# 或者使用 netstat
netstat -tlnp | grep -E '9090|9093|9100'
```

### 10.4 设置开机自启动

```bash
# 启用服务开机自启
systemctl enable node_exporter
systemctl enable alertmanager
systemctl enable prometheus

# 验证开机自启状态
systemctl is-enabled prometheus alertmanager node_exporter
# 预期输出：
# enabled
# enabled
# enabled
```

### 10.5 查看服务日志

```bash
# 查看 Prometheus 启动日志
journalctl -u prometheus --no-pager -n 50

# 查看 Alertmanager 启动日志
journalctl -u alertmanager --no-pager -n 50

# 查看 Node Exporter 启动日志
journalctl -u node_exporter --no-pager -n 50

# 实时查看日志（可选）
# journalctl -u prometheus -f
```

**检查日志中的关键信息：**

- Prometheus: Server is ready to receive web requests
- Alertmanager: Listening on :9093
- Node Exporter: Listening on :9100

## Step 11：在 Chaos 服务器 (10.210.65.148) 上部署 Node Exporter

### 11.1 准备部署文件

在 Prometheus 服务器 (10.210.65.174) 上执行：

```bash
# 创建临时打包目录
mkdir -p /tmp/node_exporter_deploy

# 复制 Node Exporter 二进制文件和服务文件
cp -r /opt/node_exporter /tmp/node_exporter_deploy/
cp /etc/systemd/system/node_exporter.service /tmp/node_exporter_deploy/

# 创建部署脚本
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

### 11.2 传输文件到 Chaos 服务器

```bash
# 在 Prometheus 服务器上执行
# 将部署文件传输到 Chaos 服务器
scp -r /tmp/node_exporter_deploy root@10.210.65.148:/tmp/

# 验证文件传输
ssh root@10.210.65.148 "ls -l /tmp/node_exporter_deploy/"
```

### 11.3 在 Chaos 服务器上执行部署

切换到 Chaos 服务器 (10.210.65.148) 执行：

```bash
# SSH 登录到 Chaos 服务器
ssh root@10.210.65.148

# 切换到部署目录
cd /tmp/node_exporter_deploy

# 执行部署脚本
./deploy.sh

# 验证部署
systemctl status node_exporter
ss -tlnp | grep 9100
curl -s http://localhost:9100/metrics | head -20
```

### 11.4 验证 Chaos 服务器 Node Exporter

```bash
# 在 Chaos 服务器上验证
systemctl is-active node_exporter
systemctl is-enabled node_exporter

# 测试指标端点
curl http://localhost:9100/metrics | grep "node_exporter_build_info"

# 查看服务日志
journalctl -u node_exporter --no-pager -n 30
```

### 11.5 从 Prometheus 服务器测试连接

在 Prometheus 服务器上执行：

```bash
# 测试网络连通性
curl -s http://10.210.65.148:9100/metrics | head -20

# 如果连接失败，检查防火墙（见下一步）
```

## Step 12：配置防火墙

### 12.1 检查防火墙状态

```bash
# 检查 firewalld 是否运行
systemctl status firewalld

# 查看当前防火墙规则
firewall-cmd --list-all
```

### 12.2 配置 Prometheus 服务器 (10.210.65.174) 防火墙

```bash
# 添加端口规则
firewall-cmd --permanent --add-port=9090/tcp  # Prometheus Web UI
firewall-cmd --permanent --add-port=9093/tcp  # Alertmanager Web UI
firewall-cmd --permanent --add-port=9100/tcp  # Node Exporter

# 重载防火墙配置
firewall-cmd --reload

# 验证规则
firewall-cmd --list-ports
# 预期输出：9090/tcp 9093/tcp 9100/tcp
```

**生产环境推荐配置（基于源地址限制）：**

```bash
# 移除简单端口规则
firewall-cmd --permanent --remove-port=9090/tcp
firewall-cmd --permanent --remove-port=9093/tcp
firewall-cmd --permanent --remove-port=9100/tcp

# 添加富规则：仅允许内网访问
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9090" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9093" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.210.65.0/24" port protocol="tcp" port="9100" accept'

# 如果需要从其他网段访问（例如管理网段）
# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.66.208.0/24" port protocol="tcp" port="9090" accept'

# 重载防火墙配置
firewall-cmd --reload

# 验证富规则
firewall-cmd --list-rich-rules
```

---

## 附件：Alertmanager 及 Prometheus 配置文件

### prometheus.yml

```yaml
# Prometheus 配置文件
# 基于 Prometheus 最佳实践: https://prometheus.io/docs/practices/

global:
  # 全局抓取间隔
  scrape_interval: 15s
  # 全局抓取超时
  scrape_timeout: 10s
  # 全局评估间隔（告警规则评估）
  evaluation_interval: 15s

  # 外部标签 - 在联邦或远程写入时使用
  external_labels:
    cluster: 'aiops-demo'
    environment: 'production'
    prometheus_instance: 'prometheus.example.com'

# Alertmanager 配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'
      timeout: 10s
      # API 版本，推荐使用 v2
      api_version: v2

# 规则文件
rule_files:
  - '/opt/prometheus/rules/linux_filesystem_alerts.yml'
  - '/opt/prometheus/rules/linux_network_alerts.yml'
  - '/opt/prometheus/rules/linux_performance_alerts.yml'
  - '/opt/prometheus/rules/linux_service_alerts.yml'

# 抓取配置
scrape_configs:
  # Prometheus 自监控
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance_type: 'prometheus_server'
          role: 'monitoring'

  # Alertmanager 监控
  - job_name: 'alertmanager'
    static_configs:
      - targets: ['localhost:9093']
        labels:
          instance_type: 'alertmanager'
          role: 'monitoring'

  # Node Exporter - Prometheus 服务器
  - job_name: 'node_exporter_prometheus'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance_name: 'prometheus.example.com'
          instance_type: 'prometheus_server'
          role: 'monitoring'
          os: 'RHEL 9.5'
    # 抓取间隔可以针对特定 job 调整
    scrape_interval: 15s
    scrape_timeout: 10s

  # Node Exporter - Chaos 服务器
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

  # 未来扩展：可以添加基于文件的服务发现
  # - job_name: 'node_exporter_file_sd'
  #   file_sd_configs:
  #     - files:
  #         - '/opt/prometheus/targets/nodes_*.yml'
  #       refresh_interval: 5m
```

### alertmanager.yml – 针对 AAP UI 才可以成功

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'EDA'
  group_by: ['alertname', 'cluster', 'service', 'instance']
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 5m


# 接收器配置
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
      max_alerts: 0  # 0 表示不限制，发送所有告警
```

### alertmanager.yml – 针对 CMD 下可以成功

```yaml
# Alertmanager 配置文件
# 基于 Prometheus 最佳实践: https://prometheus.io/docs/alerting/latest/configuration/

global:
  # 告警解决超时时间
  resolve_timeout: 5m

  # HTTP 配置（如果 webhook 需要）
  # http_config:
  #   follow_redirects: true

# 模板配置（可选）
# templates:
#   - '/opt/alertmanager/templates/*.tmpl'

# 路由配置
route:
  # 默认接收器
  receiver: 'default'

  # 分组标签 - 相同标签的告警会被聚合在一起发送
  group_by: ['alertname', 'cluster', 'service', 'instance']

  # 首次告警等待时间 - 等待同一组的其他告警
  group_wait: 10s

  # 同一组告警的发送间隔
  group_interval: 10s

  # 相同告警的重复发送间隔
  repeat_interval: 12h

  # 子路由 - 基于标签路由到不同的接收器
  routes:
    # Critical 级别的告警发送到 EDA
    - match:
        severity: critical
      receiver: 'EDA'
      group_wait: 5s
      group_interval: 5s
      repeat_interval: 3h
      continue: false

    # Warning 级别的告警也发送到 EDA，但间隔更长
    - match:
        severity: warning
      receiver: 'EDA'
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      continue: false

    # 其他告警发送到默认接收器
    - match_re:
        severity: '^(info|unknown)$'
      receiver: 'default'
      repeat_interval: 24h

# 接收器配置
receivers:
  # 默认接收器（可以配置为日志或其他低优先级渠道）
  - name: 'default'
    # webhook_configs:
    #   - url: 'http://localhost:5001/default'
    #     send_resolved: true

  # EDA Webhook 接收器
  - name: 'EDA'
    webhook_configs:
      - url: 'http://10.210.65.24:5000/alerts'
        send_resolved: true
        http_config:
          follow_redirects: true
        # 发送超时
        max_alerts: 0  # 0 表示不限制，发送所有告警

# 抑制规则 - 当某些告警触发时，抑制其他告警
inhibit_rules:
  # Critical 告警会抑制同一实例的 Warning 告警
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']

  # 节点宕机时抑制该节点的所有其他告警
  - source_match:
      alertname: 'InstanceDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']

# 时间抑制（可选 - 维护窗口期）
# time_intervals:
#   - name: 'maintenance_window'
#     time_intervals:
#       - weekdays: ['saturday', 'sunday']
#         times:
#           - start_time: '00:00'
#             end_time: '06:00'
```
