# Chaos — Use Case 3：Unexpected Outage

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 说明 |
| --- | --- | --- |
| ① | **用例** | Use Case 3：Unexpected Outage（非预期中断）故障仿真 |
| ② | **目标节点** | Chaos Server · 应用目录 `/Fault_Simulation` |
| ③ | **仿真脚本** | `simulate_chaos_for_xsos.sh` — 一键注入 / 一键还原可被 `xsos -ya` 检测到的故障 |
| ④ | **检测工具** | `/usr/local/bin/xsos -ya`（Load Average · Filesystem Usage） |
| ⑤ | **安全约束** | 副作用限于 `/var/tmp/.chaos_*` 与 `yes` 进程；不改 `/etc`、不动 systemd；磁盘最多填至 `DISK_FILL_PERCENT`（默认 92%） |
| ⑥ | **消除故障** | `restore [cpu|disk|all]` 一键还原 · `status` 确认状态 |
| ⑦ | **验证结果** | 见文末 `inject cpu` / `status` 实测输出 |

## 故障场景介绍

在 Chaos Server 上使用 `simulate_chaos_for_xsos.sh`，模拟运维巡检工具 `xsos -ya` 可即时发现的 **CPU 负载异常** 与 **根分区磁盘使用率过高** 两类突发故障，用于 Unexpected Outage 相关 DEMO 场景。所有变更可一键 `restore` 还原，不修改系统配置文件。

## 支持的故障类型

| 类型 | 机制 | `xsos -ya` 可见现象 |
| --- | --- | --- |
| **cpu** | 启动 N 个 `yes > /dev/null` 进程，将 loadavg 拉到 CPU 核数以上 | **Load Average** 区段立刻出现异常 |
| **disk** | 在 `/var/tmp` 下 `fallocate` 大文件，将 `/` 文件系统填到 ≥ `${DISK_FILL_PERCENT}%`（默认 92%） | **Filesystem Usage** 区段高亮该挂载点 |

## 使用说明 / 用法

脚本内置用法（见源码头部注释）：

```bash
sudo ./simulate_chaos_for_xsos.sh inject  [cpu|disk|all]   # 默认 all
sudo ./simulate_chaos_for_xsos.sh restore [cpu|disk|all]   # 默认 all
sudo ./simulate_chaos_for_xsos.sh status                   # 查看当前状态
```

脚本内置验证说明：

```text
/usr/local/bin/xsos -ya
  - Load Average 行若 > CPU 核数 → CPU 故障已生效
  - Filesystem Usage 中 / 的 Use% >= 92% → 磁盘故障已生效
```

---

## Chaos Server 上的应用文件

```text
[root@chaos Fault_Simulation]# ll
total 28
-rw-r--r--. 1 root root 9502 May 14 09:01 cpu_stress_webapp.py
-rw-r--r--. 1 root root  215 May 20 20:57 manual_for_networkdown.txt
-rwxr-xr-x. 1 root root 7210 May 20 22:22 simulate_chaos_for_xsos.sh
-rwxr-xr-x. 1 root root 2583 May 20 20:57 simulate_network_down.sh
[root@chaos Fault_Simulation]#
[root@chaos Fault_Simulation]# more simulate_chaos_for_xsos.sh
```

## simulate_chaos_for_xsos.sh 源代码

```bash
#!/usr/bin/env bash
# =============================================================================
# simulate_chaos_for_xsos.sh
#   在 Chaos 服务器上，一键注入 / 一键还原 1~2 种可被 xsos -ya 检测到的故障。
#
# 支持的故障类型：
#   1) cpu  : 启动 N 个 `yes > /dev/null` 进程把 loadavg 拉到 CPU 核数以上
#             → xsos -ya 的 "Load Average" 区段会立刻看到异常。
#   2) disk : 在 /var/tmp 下 fallocate 一个大文件，把 / 文件系统填到
#             >= ${DISK_FILL_PERCENT}% （默认 92%）
#             → xsos -ya 的 "Filesystem Usage" 区段会高亮该挂载点。
#
# 安全约束：
#   - 必须 root 运行；
#   - 所有副作用只在 /var/tmp/.chaos_*  和 yes 进程内，restore 一键干净；
#   - 不修改任何系统配置文件，不写 /etc，不动 systemd unit；
#   - 不会真把磁盘灌满到 100%，最多到 DISK_FILL_PERCENT。
#
# 用法：
#   sudo ./simulate_chaos_for_xsos.sh inject  [cpu|disk|all]   # 默认 all
#   sudo ./simulate_chaos_for_xsos.sh restore [cpu|disk|all]   # 默认 all
#   sudo ./simulate_chaos_for_xsos.sh status                   # 查看当前状态
#
# 验证：
#   /usr/local/bin/xsos -ya
#     - Load Average 行若 > CPU 核数 → CPU 故障已生效
#     - Filesystem Usage 中 / 的 Use% >= 92% → 磁盘故障已生效
# =============================================================================
set -euo pipefail

# ---------- 可调参数（环境变量覆盖）----------
CPU_WORKERS="${CPU_WORKERS:-$(nproc)}"
CPU_PID_FILE="${CPU_PID_FILE:-/var/run/chaos_cpu.pids}"

DISK_FILL_PATH="${DISK_FILL_PATH:-/var/tmp/.chaos_diskfill.bin}"
DISK_FILL_PERCENT="${DISK_FILL_PERCENT:-92}"   # 目标使用率（百分比）
DISK_FILL_MAX_PERCENT=97                        # 安全上限，绝不允许超过

ACTION="${1:-status}"
TARGET="${2:-all}"

# ---------- 日志 ----------
log()  { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }
fail() { printf '[%s] ERROR: %s\n' "$(date '+%F %T')" "$*" >&2; exit 1; }

[[ $EUID -eq 0 ]] || fail "请使用 root 运行 (sudo $0 $*)"
(( DISK_FILL_PERCENT > 0 && DISK_FILL_PERCENT <= DISK_FILL_MAX_PERCENT )) \
    || fail "DISK_FILL_PERCENT=$DISK_FILL_PERCENT 越界 (1..$DISK_FILL_MAX_PERCENT)"

# =============================================================================
# CPU 故障
# =============================================================================
cpu_inject() {
    if [[ -s "$CPU_PID_FILE" ]] \
       && kill -0 "$(head -n1 "$CPU_PID_FILE")" 2>/dev/null; then
        log "[CPU] 已存在运行中的故障进程 (pidfile=$CPU_PID_FILE)，跳过注入"
        return 0
    fi
    : > "$CPU_PID_FILE"
    log "[CPU] 启动 $CPU_WORKERS 个 'yes' 进程拉高 loadavg"
    for ((i=1; i<=CPU_WORKERS; i++)); do
        ( yes > /dev/null ) </dev/null >/dev/null 2>&1 &
        echo $! >> "$CPU_PID_FILE"
    done
    sleep 1
    log "[CPU] 已注入。loadavg 在 1~5 分钟内会显著升高。"
}

cpu_restore() {
    local killed=0
    if [[ -s "$CPU_PID_FILE" ]]; then
        while read -r pid; do
            [[ -n "$pid" ]] || continue
            if kill -0 "$pid" 2>/dev/null; then
                kill -9 "$pid" 2>/dev/null || true
                killed=$((killed+1))
            fi
        done < "$CPU_PID_FILE"
        rm -f "$CPU_PID_FILE"
    fi
    # 兜底：清理本机所有 yes 进程
    if pgrep -x yes >/dev/null 2>&1; then
        pkill -9 -x yes 2>/dev/null || true
        killed=$((killed+1))
    fi
    log "[CPU] 已恢复 (killed=$killed)。"
}

cpu_status() {
    log "[CPU] ----- status -----"
    log "[CPU] nproc      = $(nproc)"
    log "[CPU] loadavg    = $(cat /proc/loadavg)"
    log "[CPU] yes count  = $(pgrep -c -x yes 2>/dev/null || echo 0)"
    if [[ -s "$CPU_PID_FILE" ]]; then
        log "[CPU] pidfile    = $CPU_PID_FILE ($(wc -l < "$CPU_PID_FILE") pids)"
    else
        log "[CPU] pidfile    = (none)"
    fi
}

# =============================================================================
# 磁盘故障
# =============================================================================
_disk_target_mount() { df -P "$(dirname "$DISK_FILL_PATH")" | awk 'NR==2{print $6}'; }

disk_inject() {
    mkdir -p "$(dirname "$DISK_FILL_PATH")"
    # 读 df -P：跳过表头，单位是 1KB 块
    local size_kb used_kb target_kb fill_kb cur_pct
    read -r size_kb used_kb < <(
        df -P "$(dirname "$DISK_FILL_PATH")" \
            | awk 'NR==2 {print $2, $3}'
    )
    [[ -n "${size_kb:-}" && -n "${used_kb:-}" ]] \
        || fail "无法解析 df 输出 (path=$DISK_FILL_PATH)"
    cur_pct=$(( used_kb * 100 / size_kb ))
    target_kb=$(( size_kb * DISK_FILL_PERCENT / 100 ))
    fill_kb=$(( target_kb - used_kb ))

    if [[ -f "$DISK_FILL_PATH" ]]; then
        log "[DISK] 已存在填充文件 $DISK_FILL_PATH ($(du -h "$DISK_FILL_PATH" | awk '{print $1}'))，跳过注入"
        df -h "$(dirname "$DISK_FILL_PATH")"
        return 0
    fi
    if (( fill_kb <= 0 )); then
        log "[DISK] 当前使用率已 ${cur_pct}% >= 目标 ${DISK_FILL_PERCENT}%，无需注入"
        touch "$DISK_FILL_PATH"
        return 0
    fi

    log "[DISK] 当前 ${cur_pct}% → 目标 ${DISK_FILL_PERCENT}%，需要填充 $((fill_kb/1024)) MB"
    log "[DISK] fallocate -l ${fill_kb}KiB $DISK_FILL_PATH"
    if ! fallocate -l "${fill_kb}KiB" "$DISK_FILL_PATH" 2>/dev/null; then
        # 部分文件系统不支持 fallocate，退化到 dd
        log "[DISK] fallocate 失败，回退到 dd"
        dd if=/dev/zero of="$DISK_FILL_PATH" bs=1M \
           count=$(( (fill_kb + 1023) / 1024 )) status=none
    fi
    sync
    log "[DISK] 已注入"
    df -h "$(dirname "$DISK_FILL_PATH")"
}

disk_restore() {
    if [[ -e "$DISK_FILL_PATH" ]]; then
        log "[DISK] 删除填充文件 $DISK_FILL_PATH"
        rm -f "$DISK_FILL_PATH"
        sync
    else
        log "[DISK] 无填充文件 ($DISK_FILL_PATH)"
    fi
    df -h "$(dirname "$DISK_FILL_PATH")"
    log "[DISK] 已恢复"
}

disk_status() {
    log "[DISK] ----- status -----"
    log "[DISK] mount       = $(_disk_target_mount)"
    if [[ -e "$DISK_FILL_PATH" ]]; then
        log "[DISK] chaos file  = $DISK_FILL_PATH ($(du -h "$DISK_FILL_PATH" | awk '{print $1}'))"
    else
        log "[DISK] chaos file  = (none)"
    fi
    df -h "$(dirname "$DISK_FILL_PATH")"
}

# =============================================================================
# 调度
# =============================================================================
dispatch() {
    local act=$1 tgt=$2
    case "$tgt" in
        cpu)  "cpu_${act}"  ;;
        disk) "disk_${act}" ;;
        all)  "cpu_${act}"; "disk_${act}" ;;
        *)    fail "未知 target: $tgt (可选 cpu | disk | all)" ;;
    esac
}

case "$ACTION" in
    inject|down)   dispatch inject  "$TARGET" ;;
    restore|up)    dispatch restore "$TARGET" ;;
    status)        dispatch status  "$TARGET" ;;
    -h|--help|help)
        sed -n '2,32p' "$0"
        ;;
    *)
        fail "未知动作: $ACTION (可选 inject | restore | status)"
        ;;
esac

log "DONE: action=$ACTION target=$TARGET"
log "验证：/usr/local/bin/xsos -ya"
```

---

## 消除故障

故障仿真完成后，在 Chaos Server 的 `/Fault_Simulation` 目录下执行 `restore` 一键还原：

| 目标 | 脚本行为（见源码 `cpu_restore` / `disk_restore`） |
| --- | --- |
| **cpu** | 终止 `yes` 进程，清理 `/var/run/chaos_cpu.pids` |
| **disk** | 删除 `/var/tmp/.chaos_diskfill.bin` 填充文件 |
| **all**（默认） | 同时恢复 CPU 与磁盘 |

```bash
sudo ./simulate_chaos_for_xsos.sh restore [cpu|disk|all]   # 默认 all
sudo ./simulate_chaos_for_xsos.sh status                   # 查看当前状态
```

---

## 验证结果示例

```text
[root@chaos Fault_Simulation]# ./simulate_chaos_for_xsos.sh inject cpu
[2026-06-08 17:47:07] [CPU] 启动 2 个 'yes' 进程拉高 loadavg
[2026-06-08 17:47:08] [CPU] 已注入。loadavg 在 1~5 分钟内会显著升高。
[2026-06-08 17:47:08] DONE: action=inject target=cpu
[2026-06-08 17:47:08] 验证：/usr/local/bin/xsos -ya

[root@chaos Fault_Simulation]# ./simulate_chaos_for_xsos.sh status
[2026-06-08 17:47:25] [CPU] ----- status -----
[2026-06-08 17:47:25] [CPU] nproc      = 2
[2026-06-08 17:47:25] [CPU] loadavg    = 0.44 0.10 0.03 3/387 146475
[2026-06-08 17:47:25] [CPU] yes count  = 2
[2026-06-08 17:47:25] [CPU] pidfile    = /var/run/chaos_cpu.pids (2 pids)
[2026-06-08 17:47:25] [DISK] ----- status -----
[2026-06-08 17:47:25] [DISK] mount       = /
[2026-06-08 17:47:25] [DISK] chaos file  = (none)
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root   37G  7.5G   29G  21% /
[2026-06-08 17:47:25] DONE: action=status target=all
[2026-06-08 17:47:25] 验证：/usr/local/bin/xsos -ya

[root@chaos Fault_Simulation]# ./simulate_chaos_for_xsos.sh status
[2026-06-08 17:48:52] [CPU] ----- status -----
[2026-06-08 17:48:52] [CPU] nproc      = 2
[2026-06-08 17:48:52] [CPU] loadavg    = 1.55 0.58 0.21 1/386 146647
[2026-06-08 17:48:52] [CPU] yes count  = 0
0
[2026-06-08 17:48:52] [CPU] pidfile    = (none)
[2026-06-08 17:48:52] [DISK] ----- status -----
[2026-06-08 17:48:52] [DISK] mount       = /
[2026-06-08 17:48:52] [DISK] chaos file  = (none)
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root   37G  7.5G   29G  21% /
[2026-06-08 17:48:52] DONE: action=status target=all
[2026-06-08 17:48:52] 验证：/usr/local/bin/xsos -ya
```
