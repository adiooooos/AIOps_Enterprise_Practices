# Chaos — Use Case 3: Unexpected Outage

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Description |
| --- | --- | --- |
| ① | **Use case** | Use Case 3: Unexpected Outage fault simulation |
| ② | **Target node** | Chaos Server · app directory `/Fault_Simulation` |
| ③ | **Simulation script** | `simulate_chaos_for_xsos.sh` — one-shot inject / restore for faults detectable by `xsos -ya` |
| ④ | **Detection tool** | `/usr/local/bin/xsos -ya` (Load Average · Filesystem Usage) |
| ⑤ | **Safety** | Side effects limited to `/var/tmp/.chaos_*` and `yes` processes; no `/etc` or systemd changes; disk fill capped at `DISK_FILL_PERCENT` (default 92%) |
| ⑥ | **Clear fault** | `restore [cpu|disk|all]` one-shot revert · `status` to confirm |
| ⑦ | **Sample output** | See `inject cpu` / `status` at end of chapter |

## Fault Scenario

On the Chaos Server, `simulate_chaos_for_xsos.sh` simulates **abnormal CPU load** and **high root filesystem usage**—faults that `xsos -ya` can detect immediately—for Unexpected Outage DEMO scenarios. All changes can be reverted with one `restore`; no system config files are modified.

## Supported Fault Types

| Type | Mechanism | Visible in `xsos -ya` |
| --- | --- | --- |
| **cpu** | Start N `yes > /dev/null` processes to push loadavg above CPU core count | **Load Average** section shows anomaly immediately |
| **disk** | `fallocate` a large file under `/var/tmp` to fill `/` to ≥ `${DISK_FILL_PERCENT}%` (default 92%) | **Filesystem Usage** highlights the mount point |

## Usage

Built-in usage (from script header comments):

```bash
sudo ./simulate_chaos_for_xsos.sh inject  [cpu|disk|all]   # default all
sudo ./simulate_chaos_for_xsos.sh restore [cpu|disk|all]   # default all
sudo ./simulate_chaos_for_xsos.sh status                   # show current status
```

Built-in verification notes:

```text
/usr/local/bin/xsos -ya
  - If Load Average line > CPU core count → CPU fault is active
  - If Use% for / in Filesystem Usage >= 92% → disk fault is active
```

---

## Application Files on Chaos Server

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

## simulate_chaos_for_xsos.sh Source Code

> **Note:** The script below is the English reference version for this guide. The deployed script on the Chaos Server may use Chinese log messages; behavior is identical.

```bash
#!/usr/bin/env bash
# =============================================================================
# simulate_chaos_for_xsos.sh
#   On the Chaos server, one-shot inject / one-shot restore for 1–2 fault types
#   detectable by xsos -ya.
#
# Supported fault types:
#   1) cpu  : Start N `yes > /dev/null` processes to push loadavg above CPU core count
#             → xsos -ya "Load Average" section shows anomaly immediately.
#   2) disk : fallocate a large file under /var/tmp to fill / filesystem to
#             >= ${DISK_FILL_PERCENT}% (default 92%)
#             → xsos -ya "Filesystem Usage" section highlights that mount point.
#
# Safety constraints:
#   - Must run as root;
#   - All side effects confined to /var/tmp/.chaos_* and yes processes; restore cleans up in one step;
#   - Does not modify system config files, does not write /etc, does not touch systemd units;
#   - Will not fill disk to 100%; capped at DISK_FILL_PERCENT.
#
# Usage:
#   sudo ./simulate_chaos_for_xsos.sh inject  [cpu|disk|all]   # default all
#   sudo ./simulate_chaos_for_xsos.sh restore [cpu|disk|all]   # default all
#   sudo ./simulate_chaos_for_xsos.sh status                   # show current status
#
# Verification:
#   /usr/local/bin/xsos -ya
#     - If Load Average line > CPU core count → CPU fault is active
#     - If Use% for / in Filesystem Usage >= 92% → disk fault is active
# =============================================================================
set -euo pipefail

# ---------- Tunable parameters (override via env vars) ----------
CPU_WORKERS="${CPU_WORKERS:-$(nproc)}"
CPU_PID_FILE="${CPU_PID_FILE:-/var/run/chaos_cpu.pids}"

DISK_FILL_PATH="${DISK_FILL_PATH:-/var/tmp/.chaos_diskfill.bin}"
DISK_FILL_PERCENT="${DISK_FILL_PERCENT:-92}"   # target usage (percent)
DISK_FILL_MAX_PERCENT=97                        # safety cap, must not exceed

ACTION="${1:-status}"
TARGET="${2:-all}"

# ---------- Logging ----------
log()  { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }
fail() { printf '[%s] ERROR: %s\n' "$(date '+%F %T')" "$*" >&2; exit 1; }

[[ $EUID -eq 0 ]] || fail "Run as root (sudo $0 $*)"
(( DISK_FILL_PERCENT > 0 && DISK_FILL_PERCENT <= DISK_FILL_MAX_PERCENT )) \
    || fail "DISK_FILL_PERCENT=$DISK_FILL_PERCENT out of range (1..$DISK_FILL_MAX_PERCENT)"

# =============================================================================
# CPU fault
# =============================================================================
cpu_inject() {
    if [[ -s "$CPU_PID_FILE" ]] \
       && kill -0 "$(head -n1 "$CPU_PID_FILE")" 2>/dev/null; then
        log "[CPU] Fault processes already running (pidfile=$CPU_PID_FILE), skipping inject"
        return 0
    fi
    : > "$CPU_PID_FILE"
    log "[CPU] Starting $CPU_WORKERS 'yes' processes to raise loadavg"
    for ((i=1; i<=CPU_WORKERS; i++)); do
        ( yes > /dev/null ) </dev/null >/dev/null 2>&1 &
        echo $! >> "$CPU_PID_FILE"
    done
    sleep 1
    log "[CPU] Injected. loadavg will rise significantly within 1~5 minutes."
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
    # Fallback: clean up all local yes processes
    if pgrep -x yes >/dev/null 2>&1; then
        pkill -9 -x yes 2>/dev/null || true
        killed=$((killed+1))
    fi
    log "[CPU] Restored (killed=$killed)."
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
# Disk fault
# =============================================================================
_disk_target_mount() { df -P "$(dirname "$DISK_FILL_PATH")" | awk 'NR==2{print $6}'; }

disk_inject() {
    mkdir -p "$(dirname "$DISK_FILL_PATH")"
    # Read df -P: skip header, units are 1KB blocks
    local size_kb used_kb target_kb fill_kb cur_pct
    read -r size_kb used_kb < <(
        df -P "$(dirname "$DISK_FILL_PATH")" \
            | awk 'NR==2 {print $2, $3}'
    )
    [[ -n "${size_kb:-}" && -n "${used_kb:-}" ]] \
        || fail "Cannot parse df output (path=$DISK_FILL_PATH)"
    cur_pct=$(( used_kb * 100 / size_kb ))
    target_kb=$(( size_kb * DISK_FILL_PERCENT / 100 ))
    fill_kb=$(( target_kb - used_kb ))

    if [[ -f "$DISK_FILL_PATH" ]]; then
        log "[DISK] Fill file already exists $DISK_FILL_PATH ($(du -h "$DISK_FILL_PATH" | awk '{print $1}')), skipping inject"
        df -h "$(dirname "$DISK_FILL_PATH")"
        return 0
    fi
    if (( fill_kb <= 0 )); then
        log "[DISK] Current usage ${cur_pct}% >= target ${DISK_FILL_PERCENT}%, no inject needed"
        touch "$DISK_FILL_PATH"
        return 0
    fi

    log "[DISK] Current ${cur_pct}% → target ${DISK_FILL_PERCENT}%, need to fill $((fill_kb/1024)) MB"
    log "[DISK] fallocate -l ${fill_kb}KiB $DISK_FILL_PATH"
    if ! fallocate -l "${fill_kb}KiB" "$DISK_FILL_PATH" 2>/dev/null; then
        # Some filesystems do not support fallocate; fall back to dd
        log "[DISK] fallocate failed, falling back to dd"
        dd if=/dev/zero of="$DISK_FILL_PATH" bs=1M \
           count=$(( (fill_kb + 1023) / 1024 )) status=none
    fi
    sync
    log "[DISK] Injected"
    df -h "$(dirname "$DISK_FILL_PATH")"
}

disk_restore() {
    if [[ -e "$DISK_FILL_PATH" ]]; then
        log "[DISK] Removing fill file $DISK_FILL_PATH"
        rm -f "$DISK_FILL_PATH"
        sync
    else
        log "[DISK] No fill file ($DISK_FILL_PATH)"
    fi
    df -h "$(dirname "$DISK_FILL_PATH")"
    log "[DISK] Restored"
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
# Dispatch
# =============================================================================
dispatch() {
    local act=$1 tgt=$2
    case "$tgt" in
        cpu)  "cpu_${act}"  ;;
        disk) "disk_${act}" ;;
        all)  "cpu_${act}"; "disk_${act}" ;;
        *)    fail "Unknown target: $tgt (options: cpu | disk | all)" ;;
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
        fail "Unknown action: $ACTION (options: inject | restore | status)"
        ;;
esac

log "DONE: action=$ACTION target=$TARGET"
log "Verify: /usr/local/bin/xsos -ya"
```

---

## Clear Fault After Simulation

After fault injection, run `restore` in `/Fault_Simulation` on the Chaos Server:

| Target | Script behavior (see `cpu_restore` / `disk_restore` in source) |
| --- | --- |
| **cpu** | Terminate `yes` processes; clear `/var/run/chaos_cpu.pids` |
| **disk** | Remove fill file `/var/tmp/.chaos_diskfill.bin` |
| **all** (default) | Restore both CPU and disk |

```bash
sudo ./simulate_chaos_for_xsos.sh restore [cpu|disk|all]   # default all
sudo ./simulate_chaos_for_xsos.sh status                   # show current status
```

---

## Sample Verification Output

> **Note:** Sample output below is translated to English for readability. Actual runtime logs on the Chaos Server match the Chinese version in [05-03-Chaos-Other-Outage-CN.md](05-03-Chaos-Other-Outage-CN.md).

```text
[root@chaos Fault_Simulation]# ./simulate_chaos_for_xsos.sh inject cpu
[2026-06-08 17:47:07] [CPU] Starting 2 'yes' processes to raise loadavg
[2026-06-08 17:47:08] [CPU] Injected. loadavg will rise significantly within 1~5 minutes.
[2026-06-08 17:47:08] DONE: action=inject target=cpu
[2026-06-08 17:47:08] Verify: /usr/local/bin/xsos -ya

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
[2026-06-08 17:47:25] Verify: /usr/local/bin/xsos -ya

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
[2026-06-08 17:48:52] Verify: /usr/local/bin/xsos -ya
```
