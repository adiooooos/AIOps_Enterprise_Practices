# 分诊台 — 诊断数据配置

> **状态**：20260609 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 说明 |
| --- | --- | --- |
| ① | **角色** | 存储诊断数据；对高阶诊断源数据降噪解码 |
| ② | **同机组件** | Mattermost ChatTool · MCP for SSH |
| ③ | **目录** | `/diagnose/` · `/network_report` · `/performance_report` · `/xsos_report/` |
| ④ | **测试结果** | Use Case 1～3 降噪诊断数据与自愈 / xsos 报告样例 |

---

## 一、概要

此Server主要是承担存储诊断数据，并就高阶诊断源数据进行降噪解码的作用，另外，AIOPS Agent Solution中的chattool mattermost、 MCP for ssh也会部署在这台server上。

---

## 二、诊断数据配置


```bash
# 配置诊断数据目录
[root@aap26EDA rulebook]# mkdir /diagnose/
# 配置诊断报告目录
[root@aap26EDA rulebook]# mkdir /network_report
[root@aap26EDA rulebook]# mkdir /performance_report
[root@aap26EDA rulebook]# mkdir /xsos_report/

# 安装perf 软件
sudo dnf install perf -y
```

---

## 三、测试结果

### 3.1 Use Case 1：降噪后诊断数据

（1）use case 1 的 降噪后诊断数据（用来发送给AI Agent的LLM）

```text
[root@mattermost ~]# more /diagnose/chaos.example.com-2026-05-19-12-16/issues_latest.txt
```

```text
=== ALERT INFO ===
=== ALERT INFORMATION (from AAP Event Stream) ===
Alert Name : CriticalSystemCpuUsage
Instance   : 10.210.65.148:9100
Severity   : critical
Status     : firing
Receiver   : EDA

=== FULL EVENT PAYLOAD ===
alerts:
-   annotations:
        description: '实例 10.210.65.148:9100 的 CPU 使用率持续 1 分钟高于 95%，当前值: 199.2%。系统可能无响应！'
        summary: '严重 CPU 使用率: 10.210.65.148:9100'
    endsAt: '0001-01-01T00:00:00Z'
    fingerprint: 16aaafed494d0318
    generatorURL: http://prometheus.example.com:9090/graph?g0.expr=%28sum+without+%28cpu%29+%28irate%28node_cpu_seconds_total%7Bmode%21%3D%22idle%22%7D%5B1m%5D%29%29%29+%3E+0.95&g0.tab=1
    labels:
        alertname: CriticalSystemCpuUsage
        cluster: aiops-demo
        environment: production
        instance: 10.210.65.148:9100
        instance_name: chaos.example.com
        instance_type: managed_server
        job: node_exporter_chaos
        mode: user
        os: RHEL 9.4
        prometheus_instance: prometheus.example.com
        role: application
        severity: critical
    startsAt: '2026-05-19T04:16:16.497Z'
    status: firing
commonAnnotations:
    description: '实例 10.210.65.148:9100 的 CPU 使用率持续 1 分钟高于 95%，当前值: 199.2%。系统可能无响应！'
    summary: '严重 CPU 使用率: 10.210.65.148:9100'
commonLabels:
    alertname: CriticalSystemCpuUsage
    cluster: aiops-demo
    environment: production
    instance: 10.210.65.148:9100
    instance_name: chaos.example.com
    instance_type: managed_server
    job: node_exporter_chaos
    mode: user
    os: RHEL 9.4
    prometheus_instance: prometheus.example.com
    role: application
    severity: critical
externalURL: http://10.210.65.174:9093
groupKey: '{}:{alertname="CriticalSystemCpuUsage", cluster="aiops-demo", instance="10.210.65.148:9100"}'
groupLabels:
    alertname: CriticalSystemCpuUsage
    cluster: aiops-demo
    instance: 10.210.65.148:9100
receiver: EDA
status: firing
truncatedAlerts: 0
version: '4'

=== OTHER DIAGNOSTIC FILES ===

=== perf_output.log ===
Warning:
PID/TID switch overriding SYSTEM
[ perf record: Woken up 1887 times to write data ]
[ perf record: Captured and wrote 471.737 MB perf.data (58602 samples) ]


=== system_analysis.txt ===
=== TOP COMMAND OUTPUT  ===
top - 12:16:44 up 22:07,  2 users,  load average: 1.73, 0.63, 0.34
Tasks: 246 total,   3 running, 243 sleeping,   0 stopped,   0 zombie
%Cpu(s): 94.4 us,  5.6 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1774.9 total,    179.3 free,    757.3 used,    993.4 buff/cache
MiB Swap:   2096.0 total,   2079.5 free,     16.5 used.   1017.6 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  20541 root      20   0  307140  10476   3456 R  83.3   0.6   1:44.38 python3
  20542 root      20   0  307140  10476   3456 R  77.8   0.6   1:44.20 python3
    810 root      20   0  526152   9536   6112 S  11.1   0.5   0:00.94 account+
      1 root      20   0  175780  16652  10116 S   0.0   0.9   0:03.59 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.03 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par+
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 slub_fl+
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 netns
     10 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 mm_perc+
     12 root      20   0       0      0      0 I   0.0   0.0   0:00.00 rcu_tas+
     13 root      20   0       0      0      0 I   0.0   0.0   0:00.00 rcu_tas+
     14 root      20   0       0      0      0 I   0.0   0.0   0:00.00 rcu_tas+
     15 root      20   0       0      0      0 S   0.0   0.0   0:00.05 ksoftir+
     16 root      20   0       0      0      0 I   0.0   0.0   0:00.96 rcu_pre+
     17 root      rt   0       0      0      0 S   0.0   0.0   0:00.08 migrati+
     18 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 idle_in+
     20 root      20   0       0      0      0 S   0.0   0.0   0:00.00 cpuhp/0
     21 root      20   0       0      0      0 S   0.0   0.0   0:00.00 cpuhp/1
     22 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 idle_in+
     23 root      rt   0       0      0      0 S   0.0   0.0   0:00.20 migrati+
     24 root      20   0       0      0      0 S   0.0   0.0   0:00.04 ksoftir+
     26 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+

=== PIDSTAT COMMAND OUTPUT (Sorted by CPU usage) ===
Average:        0     20541   99.20    0.00    0.00    0.40   99.20     -  python3
Average:        0     20542   96.62    0.00    0.00    2.78   96.62     -  python3
12:16:50 PM     0     21656    1.00    1.00    0.00    4.00    2.00     1  pidstat
12:16:49 PM     0     20542   95.00    0.00    0.00    4.00   95.00     1  python3
12:16:50 PM     0     20542   97.00    0.00    0.00    3.00   97.00     1  python3
12:16:48 PM     0     20542   97.00    0.00    0.00    3.00   97.00     1  python3
12:16:47 PM     0     21656    1.00    2.00    0.00    3.00    3.00     1  pidstat
12:16:47 PM     0     20542   97.00    0.00    0.00    3.00   97.00     1  python3
12:16:49 PM     0     21656    1.00    2.00    0.00    2.00    3.00     1  pidstat
12:16:48 PM     0     21656    0.00    1.00    0.00    2.00    1.00     1  pidstat
Average:        0     21656    0.60    1.19    0.00    2.39    1.79     -  pidstat
12:16:50 PM     0     20541  100.00    0.00    0.00    1.00  100.00     0  python3
12:16:46 PM     0     21656    0.00    0.00    0.00    0.97    0.00     1  pidstat
12:16:46 PM     0     20542   97.09    0.00    0.00    0.97   97.09     1  python3
12:16:46 PM     0     20541   97.09    0.00    0.00    0.97   97.09     0  python3
Average:        0     21657    0.20    0.40    0.00    0.00    0.60     -  awk
Average:        0       817    0.20    0.20    0.00    0.00    0.40     -  vmtoolsd
Linux 5.14.0-427.42.1.el9_4.x86_64 (chaos.example.com)  05/19/2026      _x86_64_        (2 CPU)
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12:16:49 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12:16:48 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12:16:47 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12:16:46 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12:16:45 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command

=== TARGET PROCESS FOR PERF ===
PID: 20541


=== chaos.example.com-2026-05-19_perf_01.txt ===
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 58K of event 'task-clock:ppp'
# Event count (approx.): 14650500000
#
# Children      Self  Command  Shared Object                        Symbol
# ........  ........  .......  ...................................  .................................
#
    21.43%     0.00%  python3  [unknown]                            [.] 0x3fdfffffffffffff
            |
            ---0x3fdfffffffffffff
               |
                --18.13%--0x7f61ce3c0aa3
                          |
                           --18.01%--pow
                                     |
                                     |--0.69%--0x7f61ce14c779
                                     |
                                     |--0.68%--0x7f61ce14c89f
                                     |
                                     |--0.66%--0x7f61ce14c856
                                     |
                                     |--0.55%--0x7f61ce14c827
                                     |
                                     |--0.55%--0x7f61ce14c6e1
                                     |
                                      --0.52%--0x7f61ce14c759

    18.13%     0.00%  python3  libpython3.9.so.1.0                  [.] 0x00007f61ce3c0aa3
            |
            ---0x7f61ce3c0aa3
               |
                --18.01%--pow
                          |
                          |--0.69%--0x7f61ce14c779
                          |
                          |--0.68%--0x7f61ce14c89f
                          |
                          |--0.66%--0x7f61ce14c856
                          |
                          |--0.55%--0x7f61ce14c827
                          |
                          |--0.55%--0x7f61ce14c6e1
                          |
                           --0.52%--0x7f61ce14c759

    18.01%     1.04%  python3  libm.so.6                            [.] pow
            |
            |--16.97%--pow
            |          |
            |          |--0.69%--0x7f61ce14c779
            |          |
            |          |--0.68%--0x7f61ce14c89f
            |          |
            |          |--0.66%--0x7f61ce14c856
            |          |
            |          |--0.55%--0x7f61ce14c827
            |          |
            |          |--0.55%--0x7f61ce14c6e1
            |          |
            |           --0.52%--0x7f61ce14c759
            |
             --1.04%--0x3fdfffffffffffff
                       0x7f61ce3c0aa3
                       pow

     6.49%     0.00%  python3  [unknown]                            [.] 0x00000000fffffffe
            |
            ---0xfffffffe
               |
                --0.86%--0x7f61ce33173c

     2.15%     2.15%  python3  libpython3.9.so.1.0                  [.] 0x00000000001c76bc
            |
            ---0x7f61ce3c76bc

     2.15%     0.00%  python3  libpython3.9.so.1.0                  [.] 0x00007f61ce3c76bc
            |
            ---0x7f61ce3c76bc

     1.73%     1.73%  python3  libpython3.9.so.1.0                  [.] 0x0000000000130114
            |
            ---0x7f61ce330114

     1.73%     0.00%  python3  libpython3.9.so.1.0                  [.] 0x00007f61ce330114
            |
            ---0x7f61ce330114

     1.48%     0.00%  python3  [unknown]                            [.] 0xfb79d8d5878ac1ff
            |
            ---0xfb79d8d5878ac1ff
               |
                --0.58%--0x7f61ce3b3519

     1.41%     0.00%  python3  libpython3.9.so.1.0                  [.] 0x00007f61ce44df61
            |
            ---0x7f61ce44df61

```
### 3.2 Use Case 1：自愈后的 Report

(2 ) usecase 1 自愈后的report

```text
[root@mattermost ~]# more /performance_report/chaos-2026-05-20-1103.md
```

````text
# Self-Healing Performance Report

| Field | Value |
| --- | --- |
| Host | chaos (Chaos.example.com) |
| OS | RedHat 9.4 |
| Kernel | 5.14.0-427.42.1.el9_4.x86_64 |
| Generated | 2026-05-20T03:03:05Z |
| CPU threshold | > 98 % |
| PIDs killed | 38571 |

## 1. Pre-Kill Snapshot (CPU > 98%)

```text
38571 root      20   0  307140  10332   3328 R  99.0   0.6   3:52.14 python3
```

## 2. Kill Action (worker + ancestor chain)

```text
Process chain to kill: 38568 38571
-----  Snapshot before kill -----
    PID    PPID USER     %CPU COMMAND         COMMAND
  38568   35033 root      0.1 python3         python3 cpu_stress_webapp.py --duration 600
  38571   38568 root     98.9 python3         python3 cpu_stress_webapp.py --duration 600
-----  SIGTERM (15) -----
TERM 38568
TERM 38571
-----  SIGKILL (9) for holdouts -----
kill cycle completed
```

> Post-kill verification passed: no process exceeds 98% CPU.

## 3. Post-Kill Inspection

### 3.1 uptime

```text
11:03:19 up 1 day, 20:53,  3 users,  load average: 1.68, 1.57, 1.05
```

### 3.2 top (first 20 lines)

```text
top - 11:03:19 up 1 day, 20:53,  3 users,  load average: 1.68, 1.57, 1.05
Tasks: 260 total,   1 running, 259 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  6.2 sy,  0.0 ni, 93.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1774.9 total,    599.0 free,    669.4 used,    667.1 buff/cache
MiB Swap:   2096.0 total,   1935.5 free,    160.5 used.   1105.5 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    817 root      20   0  458568   9084   6016 S   6.7   0.5   1:27.85 vmtoolsd
  38885 root      20   0  226024   4096   3328 R   6.7   0.2   0:00.01 top
      1 root      20   0  175780  16140  10244 S   0.0   0.9   0:19.16 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.07 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par+
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 slub_fl+
      6 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 netns
     10 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 mm_perc+
     12 root      20   0       0      0      0 I   0.0   0.0   0:00.00 rcu_tas+
     13 root      20   0       0      0      0 I   0.0   0.0   0:00.00 rcu_tas+
     14 root      20   0       0      0      0 I   0.0   0.0   0:00.00 rcu_tas+
     15 root      20   0       0      0      0 S   0.0   0.0   0:00.17 ksoftir+
```

### 3.3 free -h

```text
total        used        free      shared  buff/cache   available
Mem:           1.7Gi       671Mi       597Mi       6.0Mi       667Mi       1.1Gi
Swap:          2.0Gi       160Mi       1.9Gi
```

### 3.4 df -h

```text
Filesystem             Size  Used Avail Use% Mounted on
devtmpfs               4.0M     0  4.0M   0% /dev
tmpfs                  888M   84K  888M   1% /dev/shm
tmpfs                  355M  8.5M  347M   3% /run
/dev/mapper/rhel-root   37G  7.5G   29G  21% /
/dev/sda2              960M  296M  665M  31% /boot
/dev/sda1              599M  7.1M  592M   2% /boot/efi
tmpfs                  178M   52K  178M   1% /run/user/42
tmpfs                  178M   36K  178M   1% /run/user/0
```

### 3.5 Top 10 processes by CPU

```text
PID USER     %CPU %MEM COMMAND         COMMAND
  37720 root      0.2  0.2 top             top
   2264 prometh+  0.1  1.0 node_exporter   /opt/node_exporter/node_exporter --web.listen-address=:9100 --web.telemetry-path=/metrics --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docke
r/.+|var/lib/kubelet/.+)($|/) --collector.netclass.ignored-devices=^(veth.*|br-.*|docker.*)$ --collector.netdev.device-exclude=^(veth.*|br-.*|docker.*)$ --log.level=info --log.format=logfmt
  38585 root      0.1  0.6 sshd            sshd: root [priv]
  38590 root      0.1  0.4 sshd            sshd: root@notty
      1 root      0.0  0.8 systemd         /usr/lib/systemd/systemd rhgb --switched-root --system --deserialize 31
      2 root      0.0  0.0 kthreadd        [kthreadd]
      3 root      0.0  0.0 rcu_gp          [rcu_gp]
      4 root      0.0  0.0 rcu_par_gp      [rcu_par_gp]
      5 root      0.0  0.0 slub_flushwq    [slub_flushwq]
      6 root      0.0  0.0 netns           [netns]
```

---

_Report auto-generated by AAP playbook `04_proceed_selfhealing.yml`._
````
### 3.3 Use Case 2：Network Error 自愈 Report

（3）use case 2：Network error自愈的report

```text
[root@mattermost ~]# more /network_report/chaos-2026-05-20-2326.md
```

````text
# Network Interface Self-Healing Report

| Field | Value |
| --- | --- |
| Host | chaos (10.210.65.148) |
| OS | RedHat 9.4 |
| Kernel | 5.14.0-427.42.1.el9_4.x86_64 |
| Generated | 2026-05-20T15:26:14Z |
| Protected devices | ens34 |
| Detected DOWN | ens37 |
| Targets | ens37 |
| Recovery OK | True |

## 1. Pre-Heal Snapshot

```text
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
ens34            UP             00:0c:29:06:89:45 <BROADCAST,MULTICAST,UP,LOWER_UP>
ens37            DOWN           00:0c:29:06:89:4f <BROADCAST,MULTICAST>
```

## 2. Heal Actions

### 2.1 ip link set up

```text
ens37: rc=0
```

### 2.2 nmcli fallback
Not needed — every target device came up after `ip link set up`.

## 3. Post-Heal Health Check

### 3.1 Per-device state

```text
ens37        carrier=1  operstate=up
```

### 3.2 ip -br link

```text
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
ens34            UP             00:0c:29:06:89:45 <BROADCAST,MULTICAST,UP,LOWER_UP>
ens37            UP             00:0c:29:06:89:4f <BROADCAST,MULTICAST,UP,LOWER_UP>
```

### 3.3 ip -br addr

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens34            UP             10.210.65.148/24 fe80::20c:29ff:fe06:8945/64
ens37            UP
```

### 3.4 ip route

```text
default via 10.210.65.254 dev ens34 proto dhcp src 10.210.65.148 metric 100
10.210.65.0/24 dev ens34 proto kernel scope link src 10.210.65.148 metric 100
```

### 3.5 Default gateway connectivity

```text
Default gateway: 10.210.65.254
PING 10.210.65.254 (10.210.65.254) 56(84) bytes of data.
64 bytes from 10.210.65.254: icmp_seq=1 ttl=64 time=0.275 ms
64 bytes from 10.210.65.254: icmp_seq=2 ttl=64 time=0.252 ms

--- 10.210.65.254 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1154ms
rtt min/avg/max/mdev = 0.252/0.263/0.275/0.011 ms
```

---

✅ **All targeted interfaces are UP. Self-heal succeeded.**

_Report auto-generated by AAP playbook `05_network_interface_selfhealing.yml`._
````
### 3.4 Use Case 3：xsos Report

（4） use case 3 -遇到未知故障的xsos report

```text
[root@mattermost ~]# more /xsos_report/10.210.65.148-2026-05-20-2336.md
```

````text
# xsos report — 10.210.65.148

- Host       : 10.210.65.148
- Generated  : 2026-05-20T23:36:24+08:00
- Command    : /usr/local/bin/xsos -ya

```text
DMIDECODE
  BIOS:
    Vend: VMware, Inc.
    Vers: VMW201.00V.20192059.B64.2207280713
    Date: 07/28/2022
    BIOS Rev:
    FW Rev:
  System:
    Mfr:  VMware, Inc.
    Prod: VMware20,1
    Vers: None
    Ser:  VMware-56 4d 55 00 2e 73 83 da-e6 3b d5 48 75 06 89 45
    UUID: 00554d56-732e-da83-e63b-d54875068945
  CPU:
    2 of 2 CPU sockets populated, 2 cores/0 threads per CPU
    4 total cores, 0 total threads
    Mfr:  GenuineIntel
    Fam:  Unknown
    Freq: 2670 MHz
    Vers: Intel(R) Xeon(R) CPU E5-2680 v2 @ 2.80GHz
  Memory:
    Total: 2048 MiB (2 GiB)
    DIMMs: 1 of 130 populated
    MaxCapacity: 2048 MiB (2 GiB / 0.00 TiB)

OS
  Hostname: chaos.example.com
  Distro:   [redhat-release] Red Hat Enterprise Linux release 9.4 (Plow)
            [os-release] Red Hat Enterprise Linux 9.4 (Plow) 9.4 (Plow)
  RHN:      (missing)
  RHSM:     hostname = subscription.rhsm.redhat.com
            proxy_hostname =
  YUM:      3 enabled plugins: debuginfo-install, product-id, subscription-manager
  Runlevel: N 5  (default )
  SELinux:  enforcing  (default enforcing)
  Arch:     mach=x86_64  cpu=x86_64  platform=x86_64
  Kernel:
    Booted kernel:  5.14.0-427.42.1.el9_4.x86_64
    GRUB default:
    Build version:
      Linux version 5.14.0-427.42.1.el9_4.x86_64
      (mockbuild@x86-64-03.build.eng.rdu2.redhat.com) (gcc (GCC) 11.4.1 20231218 (Red
      Hat 11.4.1-3), GNU ld version 2.35.2-43.el9) #1 SMP PREEMPT_DYNAMIC Fri Oct 18
      14:35:40 EDT 2024
    Booted kernel cmdline:
      BOOT_IMAGE=(hd0,gpt2)/vmlinuz-5.14.0-427.42.1.el9_4.x86_64
      root=/dev/mapper/rhel-root ro crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M
      resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet
    GRUB default kernel cmdline:

    Taint-check: 0  (kernel untainted)
    - - - - - - - - - - - - - - - - - - -
  Sys time:  Wed May 20 11:36:25 PM CST 2026
  Boot time: Mon May 18 09:59:20 PM CST 2026
  Time Zone: Asia/Shanghai
  Uptime:    2 days,  1:37,  3 users
  LoadAvg:   [2 CPU] 2.03 (101%), 1.06 (53%), 0.66 (33%)
  /proc/stat:
    procs_running: 3   procs_blocked: 0    processes [Since boot]: 55335
    cpu [Utilization since boot]:
      us 6%, ni 0%, sys 0%, idle 93%, iowait 0%, irq 0%, sftirq 0%, steal 0%

KDUMP CONFIG
  kexec-tools rpm version:
    kexec-tools-2.0.27-8.el9_4.3.x86_64
  Service enablement:
    UNIT           STATE
    kdump.service  enabled  enabled
  kdump initrd/initramfs:
    33722880 May 11 11:58 initramfs-5.14.0-427.42.1.el9_4.x86_64kdump.img
  Memory reservation config:
    /proc/cmdline { crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M }
    GRUB default  { crashkernel param not present }
  Actual memory reservation per /proc/iomem:
      73000000-7effffff : Crash kernel
  kdump.conf:
    auto_reset_crashkernel yes
    path /var/crash
    core_collector makedumpfile -l --message-level 7 -d 31
  kdump.conf "path" available space:
    System MemTotal (uncompressed core size) { 1.73 GiB }
    Available free space on target path's fs { 2.90 GiB }  (fs=/)
  Panic sysctls:
    kernel.sysrq [bitmask] =  "16"  (see proc man page)
    kernel.panic [secs] =  0  (no autoreboot on panic)
    kernel.hung_task_panic =  0
    kernel.panic_on_oops =  1
    kernel.panic_on_io_nmi =  0
    kernel.panic_on_unrecovered_nmi =  0
    kernel.panic_on_stackoverflow =
    kernel.panic_on_warn =  0
    kernel.softlockup_panic =  0
    kernel.unknown_nmi_panic =  0
    kernel.nmi_watchdog =  0
    vm.panic_on_oom [0-2] =  0  (no panic)

CPU
  2 logical processors (2 CPU cores)
  1 Intel Xeon CPU E5-2680 v2 @ 2.80GHz (flags: aes,constant_tsc,ht,lm,nx,pae,rdrand)
  └─2 threads / 2 cores each

INTERRUPTS
    0: ▊. IO-APIC 2-edge timer
    1: .▊ IO-APIC 1-edge i8042
    8: .. IO-APIC 8-edge rtc0
    9: .▊ IO-APIC 9-fasteoi acpi
   12: .▊ IO-APIC 12-edge i8042
   14: .. IO-APIC 14-edge ata_piix
   15: .. IO-APIC 15-edge ata_piix
   16: .. IO-APIC 16-fasteoi ehci_hcd:usb2, vmwgfx
   18: ▊. IO-APIC 18-fasteoi uhci_hcd:usb1
   24: ▊▊ PCI-MSIX-0000:02:00.0 0-edge vmw_pvscsi
   25: ▊▊ PCI-MSIX-0000:02:02.0 0-edge ens34-rxtx-0
   26: ▊▊ PCI-MSIX-0000:02:02.0 1-edge ens34-rxtx-1
   27: .. PCI-MSIX-0000:02:02.0 2-edge ens34-event-2
   28: ▊▊ PCI-MSI-0000:02:04.0 0-edge ahci[0000:02:04.0]
   29: .▊ PCI-MSIX-0000:00:07.7 0-edge vmw_vmci
   30: .. PCI-MSIX-0000:00:07.7 1-edge vmw_vmci
   31: .▊ PCI-MSIX-0000:00:07.7 2-edge vmw_vmci
   32: ▊▊ PCI-MSIX-0000:02:05.0 0-edge ens37-rxtx-0
   33: ▊▊ PCI-MSIX-0000:02:05.0 1-edge ens37-rxtx-1
   34: .. PCI-MSIX-0000:02:05.0 2-edge ens37-event-2
  NMI: .. Non-maskable interrupts
  LOC: ▊▊ Local timer interrupts
  SPU: .. Spurious interrupts
  PMI: .. Performance monitoring interrupts
  IWI: ▊▊ IRQ work interrupts
  RTR: .. APIC ICR read retries
  RES: ▊▊ Rescheduling interrupts
  CAL: ▊▊ Function call interrupts
  TLB: ▊▊ TLB shootdowns
  TRM: .. Thermal event interrupts
  THR: .. Threshold APIC interrupts
  DFR: .. Deferred Error APIC interrupts
  MCE: .. Machine check exceptions
  MCP: ▊▊ Machine check polls
  ERR: .
  MIS: .
  PIN: .. Posted-interrupt notification event
  NPI: .. Nested posted-interrupt event
  PIW: .. Posted-interrupt wakeup event

MEMORY
  Stats graphed as percent of MemTotal:
    MemUsed    ▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊...  93.9%
    Buffers    ..................................................   0.0%
    Cached     ▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊▊.......................  53.1%
    HugePages  ..................................................   0.0%
    Dirty      ..................................................   0.0%
  RAM:
    1.7 GiB total ram
    1.6 GiB (94%) used
    0.7 GiB (41%) used excluding Buffers/Cached
    0 GiB (0%) dirty
  HugePages:
    No ram pre-allocated to HugePages
  THP:
    0.18 GiB allocated to THP
  LowMem/Slab/PageTables/Shmem/Percpu:
    0.19 GiB (11%) of total ram used for Slab
    0.01 GiB (1%) of total ram used for PageTables
    0.01 GiB (0%) of total ram used for Shmem
    0 GiB (0%) of total ram used for Percpu
  Swap:
    0.2 GiB (11%) used of 2 GiB total

STORAGE
  Whole Disks from /proc/partitions:
    1 disks, totaling 40 GiB (0.04 TiB)
    - - - - - - - - - - - - - - - - - - - - -
    Disk        Size in GiB
    ----        -----------
    sda         40

  Disk layout from lsblk:
    NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sda             8:0    0   40G  0 disk
    ├─sda1          8:1    0  600M  0 part /boot/efi
    ├─sda2          8:2    0    1G  0 part /boot
    └─sda3          8:3    0 38.4G  0 part
      ├─rhel-root 253:0    0 36.4G  0 lvm  /
      └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
    sr0            11:0    1 1024M  0 rom

  Filesystem usage from df:
    Filesystem            1K-blocks     Used Available Use% Mounted on
    /dev/mapper/rhel-root  38064128 35018984   3045144  92% /
    /dev/sda2                983040   302800    680240  31% /boot
    /dev/sda1                613184     7208    605976   2% /boot/efi

DM-MULTIPATH
  [No paths detected]

LSPCI
  Net:
    1 dual-port (2) VMware VMXNET3 Ethernet Controller (rev 01)
  Storage:
    (1) VMware SATA AHCI controller
    (1) VMware PVSCSI SCSI Controller (rev 02)
  VGA:
    VMware SVGA II Adapter

ETHTOOL
  Interface Status:
    ens34  0000:02:02.0  link=up 10000Mb/s full (autoneg=N)  rx ring 1024/4096  drv vmxnet3 v1.7.0.0-k-NAPI / fw UNKNOWN
    ens37  0000:02:05.0  link=up 10000Mb/s full (autoneg=N)  rx ring 1024/4096  drv vmxnet3 v1.7.0.0-k-NAPI / fw UNKNOWN
  Interface Errors:
    [None]

SOFTIRQ
  Backlog max is sufficient (Current value: net.core.netdev_max_backlog = 1000)
  Budget is sufficient (Current value: net.core.netdev_budget = 300)
  (see https://access.redhat.com/solutions/1241943)

NETDEV
  Interface  RxMiBytes  RxPackets  RxErrs  RxDrop   RxFifo  RxComp  RxFrame  RxMultCast
  =========  =========  =========  ======  ======   ======  ======  =======  ==========
  ens34      1384       1112 k     0       12 (0%)  0       0       0        20767 (2%)
  ens37      0          0 k        0       0        0       0       0        0
  - - - - - - - - - - - - - - - - -
  Interface  TxMiBytes  TxPackets  TxErrs  TxDrop   TxFifo  TxComp  TxColls  TxCarrier
  =========  =========  =========  ======  ======   ======  ======  =======  ==========
  ens34      240        545 k      0       0        0       0       0        0
  ens37      0          0 k        0       0        0       0       0        0

SOCKSTAT
  sockets: used 602
  TCP: inuse 17 orphan 0 tw 0 alloc 24 mem 0
  UDP: inuse 4 mem 0
  UDPLITE: inuse 0
  RAW: inuse 0
  FRAG: inuse 0 memory 0

IP4
  Interface  Master IF  MAC Address        MTU     State  IPv4 Address
  =========  =========  =================  ======  =====  ==================
  lo         -          -                  65536   up     127.0.0.1/8
  ens34      -          00:0c:29:06:89:45  1500    up     10.210.65.148/24
  ens37      -          00:0c:29:06:89:4f  1500    up     -

IP6
  Interface  Master IF  MAC Address        MTU     State  IPv6 Address                                 Scope
  =========  =========  =================  ======  =====  ===========================================  =====
  lo         -          -                  65536   up     ::1/128                                      host
  ens34      -          00:0c:29:06:89:45  1500    up     fe80::20c:29ff:fe06:8945/64                  link
  ens37      -          00:0c:29:06:89:4f  1500    up     -                                            -

SYSCTLS
  kernel.
    hostname =  "chaos.example.com"
    osrelease =  "5.14.0-427.42.1.el9_4.x86_64"
    tainted =  "0"  (kernel untainted)
    random.boot_id =  "94ec8ba1-a92d-465a-a8ff-71a251314f76"
    random.entropy_avail [bits] =  "256"
    hung_task_panic [bool] =  "0"
    hung_task_timeout_secs =  "120"  (secs task must be D-state to trigger)
    hung_task_warnings [num_warnings] =  "10"  (will report 10 [more] warnings)
    msgmax [bytes] =  "8192"
    msgmnb [bytes] =  "16384"
    msgmni [msg queues] =  "32000"
    panic [secs] =  "0"  (no autoreboot on panic)
    panic_on_oops [bool] =  "1"
    nmi_watchdog [bool] =  "0"
    panic_on_io_nmi [bool] =  "0"
    panic_on_unrecovered_nmi [bool] =  "0"
    unknown_nmi_panic [bool] =  "0"
    panic_on_stackoverflow [bool] =  {sysctl not present}
    panic_on_warn [bool] =  "0"
    softlockup_panic [bool] =  "0"
    softlockup_thresh [secs] =  {sysctl not present}
    pid_max =  "4194304"
    threads-max =  "13701"
    sem [array] =  "32000  1024000000  500  32000"
      SEMMSL (max semaphores per array) =  32000
      SEMMNS (max sems system-wide)     =  1024000000
      SEMOPM (max ops per semop call)   =  500
      SEMMNI (max number of sem arrays) =  32000
    shmall [4-KiB pages] =  "18446744073692774399"  (70368744177600.0 GiB max total shared memory)
    shmmax [bytes] =  "18446744073692774399"  (17179869183.98 GiB max segment size)
    shmmni [segments] =  "4096"  (max number of segs)
    sysrq [bitmask] =  "16"  (see proc man page)
    sched_min_granularity_ns [nanosecs] =  {sysctl not present}
    sched_latency_ns [nanosecs] =  {sysctl not present}
  fs.
    file-max [fds] =  "9223372036854775807"  (system-wide limit for num open files [file descriptors])
    nr_open [fds] =  "1073741816"  (per-process limit for num open files [see also RLIMIT_NOFILE])
    file-nr [fds] =  "5248  0  9223372036854775807"  (num allocated fds, N/A, num free fds)
    inode-nr [inodes] =  "119733  3077"  (nr_inodes allocated, nr_free_inodes)
    dentry-state [dentries] =  "120349  90788  45  0  3055  0"  (nr_dentry, nr_unused, age_limit, want_pages, nr_negative, dummy)
  net.
    core.busy_read [microsec] =  "0"  (off)
    core.busy_poll [microsec] =  "0"  (off)
    core.netdev_budget [packets] =  "300"
    core.netdev_max_backlog [packets] =  "1000"
    core.rmem_default [bytes] =  "212992"  (208 KiB)
    core.wmem_default [bytes] =  "212992"  (208 KiB)
    core.rmem_max [bytes] =  "212992"  (208 KiB)
    core.wmem_max [bytes] =  "212992"  (208 KiB)
    ipv4.icmp_echo_ignore_all [bool] =  "0"
    ipv4.ip_forward [bool] =  "0"
    ipv4.ip_local_port_range [ports] =  "32768  60999"  (defines ephemeral port range used by TCP/UDP)
    ipv4.ip_local_reserved_ports [ports] =  ""  (comma-separated ports/ranges to exclude from automatic port assignments)
    ipv4.tcp_max_orphans [sockets] =  "8192"  (512 MiB @ max 64 KiB per orphan)
    ipv4.tcp_mem [4-KiB pages] =  "19758  26347  39516"  (0.08 GiB, 0.10 GiB, 0.15 GiB)
    ipv4.udp_mem [4-KiB pages] =  "39519  52694  79038"  (0.15 GiB, 0.20 GiB, 0.30 GiB)
    ipv4.tcp_window_scaling [bool] =  "1"
    ipv4.tcp_rmem [bytes] =  "4096  131072  6291456"  (4 KiB, 128 KiB, 6144 KiB)
    ipv4.tcp_wmem [bytes] =  "4096  16384  4194304"  (4 KiB, 16 KiB, 4096 KiB)
    ipv4.udp_rmem_min [bytes] =  "4096"  (4 KiB)
    ipv4.udp_wmem_min [bytes] =  "4096"  (4 KiB)
    ipv4.tcp_sack [bool] =  "1"
    ipv4.tcp_timestamps [bool] =  "1"
    ipv4.tcp_fastopen [bitmap] =  "1"  (enable send; see ip-sysctl.txt)
    ipv4.ipfrag_high_thresh [bytes] =  "4194304"  (4096 KiB)
    ipv4.ipfrag_low_thresh [bytes] =  "3145728"  (3072 KiB)
    ipv6.ip6frag_high_thresh [bytes] =  "4194304"  (4096 KiB)
    ipv6.ip6frag_low_thresh [bytes] =  "3145728"  (3072 KiB)
  vm.
    dirty_ratio =  "20"  (% of total system memory)
    dirty_bytes =  "0"  (disabled -- check dirty_ratio)
    dirty_background_ratio =  "10"  (% of total system memory)
    dirty_background_bytes =  "0"  (disabled -- check dirty_background_ratio)
    dirty_expire_centisecs =  "3000"
    dirty_writeback_centisecs =  "500"
    max_map_count =  "65530"
    min_free_kbytes =  "45056"
    nr_hugepages [2-MiB pages] =  "0"
    nr_overcommit_hugepages [2-MiB pages] =  "0"
    overcommit_memory [0-2] =  "0"  (heuristic overcommit)
    overcommit_ratio =  "50"
    oom_kill_allocating_task [bool] =  "0"  (scan tasklist)
    panic_on_oom [0-2] =  "0"  (no panic)
    swappiness [0-100] =  "60"
    force_cgroup_v2_swappiness [bool] =  {sysctl not present}
    vfs_cache_pressure [0-100] =  "100"
    zone_reclaim_mode [bitmask] =  "0"  (zone reclaim disabled)

PS CHECK
  Total number of threads/processes:
    683 / 272
  Top users of CPU & MEM:
    USER      %CPU    %MEM   RSS
    root      219.1%  36.6%  0.68 GiB
    prometh+  0.1%    0.9%   0.02 GiB
    polkitd   0.0%    1.1%   0.02 GiB
    gdm       0.0%    20.7%  0.39 GiB
    dbus      0.0%    0.3%   0.01 GiB
    colord    0.0%    0.4%   0.01 GiB
    avahi     0.0%    0.3%   0.01 GiB
  Uninteruptible sleep threads/processes (0/0):
    [None]
  Defunct zombie threads/processes (0/0):
    [None]
  Top CPU-using processes:
    USER      PID    %CPU  %MEM  VSZ-MiB  RSS-MiB  TTY    STAT  START  TIME  COMMAND
    root      55023  107   0.0   216      2        pts/1  R     23:32  3:56  yes
    root      55022  107   0.0   216      2        pts/1  R     23:32  3:56  yes
    root      55208  2.1   1.3   243      23       ?      S     23:36  0:00  /usr/bin/python3
    root      55217  1.1   0.2   219      4        ?      S     23:36  0:00  /bin/bash /usr/local/bin/xsos -ya
    root      55298  0.8   0.4   19       8        ?      Ss    23:36  0:00  /usr/lib/systemd/systemd-timedated
    root      55010  0.7   10.0  646      179      ?      Ssl   23:32  0:01  /usr/libexec/packagekitd
    root      55043  0.2   0.6   21       12       ?      Ss    23:36  0:00  sshd: root [priv]
    root      55211  0.1   0.2   219      5        ?      S     23:36  0:00  /bin/bash /usr/local/bin/xsos -ya
    root      55046  0.1   0.4   21       8        ?      S     23:36  0:00  sshd: root@notty
    prometh+  2264   0.1   0.9   1215     16       ?      Ssl   May18  3:51  /opt/node_exporter/node_exporter --web.listen-address=:9100 --web.telemetry-path=/metrics
  Top MEM-using processes:
    USER      PID    %CPU  %MEM  VSZ-MiB  RSS-MiB  TTY    STAT  START  TIME  COMMAND
    root      55010  0.7   10.0  646      179      ?      Ssl   23:32  0:01  /usr/libexec/packagekitd
    gdm       1254   0.0   6.7   3794     120      tty1   Sl+   May18  0:52  /usr/bin/gnome-shell
    root      55208  2.1   1.3   243      23       ?      S     23:36  0:00  /usr/bin/python3
    polkitd   808    0.0   1.1   2522     20       ?      Ssl   May18  0:06  /usr/lib/polkit-1/polkitd --no-debug
    root      918    0.0   1.0   466      18       ?      Ssl   May18  0:08  /usr/sbin/NetworkManager --no-daemon
    prometh+  2264   0.1   0.9   1215     16       ?      Ssl   May18  3:51  /opt/node_exporter/node_exporter --web.listen-address=:9100 --web.telemetry-path=/metrics
    root      36934  0.0   0.7   21       13       ?      Ss    18:38  0:00  sshd: root [priv]
    root      35022  0.0   0.7   21       13       ?      Ss    17:59  0:00  sshd: root [priv]
    root      35018  0.0   0.7   21       13       ?      Ss    17:59  0:00  sshd: root [priv]
    root      17633  0.0   0.7   24       13       ?      Ss    May19  0:00  /usr/lib/systemd/systemd --user
  Top thread-spawning processes:
    #   USER      PID    %CPU  %MEM  VSZ-MiB  RSS-MiB  TTY    STAT  START  TIME  COMMAND
    12  gdm       1254   0.0   6.7   3794     120      tty1   -     May18  0:52  /usr/bin/gnome-shell
    6   polkitd   808    0.0   1.1   2522     20       ?      -     May18  0:06  /usr/lib/polkit-1/polkitd --no-debug
    6   gdm       1860   0.0   0.4   589      8        tty1   -     May18  0:10  /usr/libexec/gsd-smartcard
    5   root      814    0.0   0.5   388      10       ?      -     May18  0:00  /usr/libexec/udisks2/udisksd
    5   prometh+  2264   0.1   0.9   1215     16       ?      -     May18  3:51  /opt/node_exporter/node_exporter --web.listen-address=:9100 --web.telemetry-path=/metrics
    5   gdm       2065   0.0   0.6   2731     12       tty1   -     May18  0:00  /usr/bin/gjs /usr/share/gnome-shell/org.gnome.ScreenSaver
    5   gdm       1843   0.0   0.6   2731     12       tty1   -     May18  0:00  /usr/bin/gjs /usr/share/gnome-shell/org.gnome.Shell.Notifications
    5   gdm       1787   0.0   0.4   532      8        ?      -     May18  0:00  /usr/bin/wireplumber
    4   root      851    0.0   0.4   382      8        ?      -     May18  0:00  /usr/sbin/ModemManager
    4   root      810    0.0   0.4   514      7        ?      -     May18  0:01  /usr/libexec/accounts-daemon

SS CHECK
  Sessions:
    Proto Local Peer sock_drop app_limited dsack_dups lost reord_seen back_log retrans_total rq tq %CPU %MEM User Command
    ----- ----- ---- --------- ----------- ---------- ---- ---------- -------- ------------- -- -- ---- ---- ---- -------

FIREWALL
  Services enabled:
    No firewall services enabled.

  Rules loaded:
    No rules loaded.
Warning: '/proc/net/bonding/' files unreadable; skipping bonding check

cat: /ps: No such file or directory
cat: /ps: No such file or directory
cat: /ps: No such file or directory
cat: /ps: No such file or directory
cat: /sos_commands/networking/ss_-peaonmi: No such file or directory
/usr/local/bin/xsos: line 3787: /sos_commands/systemd/systemctl_status_--all: No such file or directory
/usr/local/bin/xsos: line 3787: /sos_commands/systemd/systemctl_status_--all: No such file or directory
/usr/local/bin/xsos: line 3787: /sos_commands/systemd/systemctl_status_--all: No such file or directory
````

