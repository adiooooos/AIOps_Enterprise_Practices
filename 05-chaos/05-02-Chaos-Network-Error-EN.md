# Chaos — Use Case 2: Network Error

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Chapter Overview

| # | Dimension | Description |
| --- | --- | --- |
| ① | **Use case** | Use Case 2: Network Error fault simulation |
| ② | **Target node** | Chaos Server (`10.210.65.148`) · app directory `/Fault_Simulation` |
| ③ | **Simulation script** | `simulate_network_down.sh` — runs `ip link set down/up` on `ens37` to trigger / clear Prometheus `NetworkInterfaceDown`; protected management NIC `ens34` (carries `10.210.65.148`) cannot be touched |
| ④ | **Alert timing** | Prometheus rule `for: 2m` — NIC must stay DOWN for 2+ minutes before firing |
| ⑤ | **Simulate fault** | `down` inject fault → `status` check state → `up` recover |

## Overview: Use Case 2 — Network Error Fault Simulation

On the Chaos Server, use `simulate_network_down.sh` to perform UP/DOWN/STATUS on non-management NIC `ens37`, simulating a link failure and triggering / clearing `NetworkInterfaceDown` alerts. The script does not modify config files; all changes are in-memory only. Source code of `simulate_network_down.sh` is shown below.

---

## Application File on Chaos Server

```text
[root@chaos Fault_Simulation]# ll simulate_network_down.sh
-rwxr-xr-x. 1 root root 2583 May 20 20:57 simulate_network_down.sh
[root@chaos Fault_Simulation]# pwd
/Fault_Simulation
[root@chaos Fault_Simulation]#
[root@chaos Fault_Simulation]# more simulate_network_down.sh
```

## simulate_network_down.sh Source Code

```bash
#!/usr/bin/env bash
  # =============================================================================
  # simulate_network_down.sh
  #   在 Chaos (10.210.65.148) 上对 ens37 进行 UP/DOWN/STATUS 操作，
  #   触发 / 恢复 / 检查 Prometheus 的 NetworkInterfaceDown 告警。
  #
  # 安全约束：
  #   - 只允许操作 TARGET_DEV（默认 ens37）；
  #   - 显式禁止误操作管理网卡 PROTECTED_DEV（默认 ens34，承载 10.210.65.148）；
  #   - 不修改任何配置文件，所有变更都是内存态，重启后回到原状态。
  # 用法：
  #   sudo ./simulate_network_down.sh down      # 模拟故障（默认动作）
  #   sudo ./simulate_network_down.sh up        # 恢复网卡
  #   sudo ./simulate_network_down.sh status    # 查看当前状态
  # 触发告警时长：Prometheus rule for: 2m → 设备 DOWN 持续 2 分钟以上才会 firing。
  # =============================================================================
  set -euo pipefail

  TARGET_DEV="${TARGET_DEV:-ens37}"
  PROTECTED_DEV="${PROTECTED_DEV:-ens34}"
  ACTION="${1:-down}"

  log()  { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }
  fail() { printf '[%s] ERROR: %s\n' "$(date '+%F %T')" "$*" >&2; exit 1; }

  [[ $EUID -eq 0 ]]                       || fail "请使用 root 运行（sudo $0 $*）"
  [[ "$TARGET_DEV" != "$PROTECTED_DEV" ]] || fail "TARGET_DEV 不能等于受保护的 $PROTECTED_DEV"
  ip link show "$TARGET_DEV" >/dev/null 2>&1 \
      || fail "网卡 $TARGET_DEV 不存在"

  show_status() {
      log "----- $TARGET_DEV current status -----"
      ip -br link show "$TARGET_DEV"
      log "carrier   = $(cat /sys/class/net/$TARGET_DEV/carrier   2>/dev/null || echo n/a)"
      log "operstate = $(cat /sys/class/net/$TARGET_DEV/operstate 2>/dev/null || echo n/a)"
      log "管理网卡 $PROTECTED_DEV 状态："
      ip -br link show "$PROTECTED_DEV"
  }

  case "$ACTION" in
      down)
          log "对 $TARGET_DEV 执行 ip link set down（模拟网卡故障）"
          ip link set "$TARGET_DEV" down
          sleep 1
          show_status
          log "故障已注入。Prometheus rule for=2m，预计 ~2 分钟后 NetworkInterfaceDown 进入 firing。"
          ;;
      up)
          log "对 $TARGET_DEV 执行 ip link set up（恢复网卡）"
          ip link set "$TARGET_DEV" up
          sleep 2
          show_status
          log "网卡已恢复。"
          ;;
      status)
          show_status
          ;;
      *)
          fail "未知动作：$ACTION（可选：down | up | status）"
          ;;
  esac
```

---

## Simulate Fault

On the Chaos Server, in `/Fault_Simulation`:

```bash
sudo ./simulate_network_down.sh down       # 拉低 ens37，~2 分钟后 EDA 触发
sudo ./simulate_network_down.sh status     # 验证当前状态
sudo ./simulate_network_down.sh up         # 测试完恢复
```
