# Chaos — Use Case 1：Process Hung

> **状态**：20260608 Updated  
> **系列**：AIOps DEMO Center 部署与配置分步指南  

## 本章概要

| # | 维度 | 说明 |
| --- | --- | --- |
| ① | **用例** | Use Case 1：Process Hung 故障仿真 |
| ② | **目标节点** | Chaos Server · 应用目录 `/Fault_Simulation` |
| ③ | **仿真应用** | `cpu_stress_webapp.py` — 模拟 Web 应用峰值 CPU 负载，触发 Process Hung 相关告警场景 |
| ④ | **模拟故障** | 启动 `python3 cpu_stress_webapp.py --duration 600` · 结束 `Ctrl+C` |

## 概要：Usecase1:Process Hung的故障仿真

我们会在Chaos Server上部署一个Python应用、模拟webapp峰值造成Process Hung问题的一个业务应用程序。cpu_stress_webapp.py应用源代码如下所示：

---

## Chaos Server 上的应用文件

```text
[root@chaos Fault_Simulation]# pwd
/Fault_Simulation
[root@chaos Fault_Simulation]# ll cpu_stress_webapp.py
-rw-r--r--. 1 root root 9502 May 14 09:01 cpu_stress_webapp.py
[root@chaos Fault_Simulation]# more cpu_stress_webapp.py
```

## cpu_stress_webapp.py 源代码

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
CPU压力测试Web应用
用于模拟高CPU使用率，触发Prometheus告警
适用于RHEL9.6系统测试
"""

import os
import sys
import time
import signal
import threading
import multiprocessing
import argparse
import json
from datetime import datetime
import psutil
import math

# 添加多进程支持
if __name__ == '__main__':
    multiprocessing.freeze_support()

class CPUStressWebApp:
    def __init__(self, target_cpu_percent=99, duration=300, cores=None):
        """
        初始化CPU压力测试应用

        Args:
            target_cpu_percent (int): 目标CPU使用率百分比 (1-100)
            duration (int): 测试持续时间（秒），0表示无限运行
            cores (int): 使用的CPU核心数，None表示使用所有核心
        """
        self.target_cpu_percent = target_cpu_percent
        self.duration = duration
        self.start_time = time.time()
        self.running = True
        self.threads = []
        self.processes = []

        # 获取系统信息
        self.cpu_count = multiprocessing.cpu_count()
        self.cores_to_use = cores if cores else self.cpu_count

        # 设置信号处理
        signal.signal(signal.SIGINT, self.signal_handler)
        signal.signal(signal.SIGTERM, self.signal_handler)

        print(f"=== CPU压力测试Web应用 ===")
        print(f"系统CPU核心数: {self.cpu_count}")
        print(f"使用核心数: {self.cores_to_use}")
        print(f"目标CPU使用率: {self.target_cpu_percent}%")
        print(f"测试持续时间: {duration}秒" if duration > 0 else "无限运行")
        print(f"开始时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        print("=" * 50)

    def signal_handler(self, signum, frame):
        """信号处理器，用于优雅退出"""
        print(f"\n收到信号 {signum}，正在停止测试...")
        self.running = False
        self.cleanup()

    def cpu_intensive_task(self, thread_id, target_percent):
        """
        模拟CPU密集型任务 - 持续CPU使用模式

        Args:
            thread_id (int): 线程ID
            target_percent (int): 目标CPU使用率
        """
        print(f"进程 {thread_id} 开始运行，目标CPU使用率: {target_percent}%")

        # 使用持续CPU使用模式，类似stress-ng
        # 通过控制工作强度来达到目标CPU使用率
        work_ratio = target_percent / 100.0

        while self.running:
            # 持续进行CPU密集型计算
            self.simulate_continuous_workload(work_ratio)

    def simulate_continuous_workload(self, work_ratio):
        """
        模拟持续CPU工作负载 - 类似stress-ng的持续模式

        Args:
            work_ratio (float): 工作强度比例 (0.0-1.0)
        """
        # 使用更高效的CPU密集型操作，减少函数调用开销
        iterations = int(50000 * work_ratio)  # 增加迭代次数以提升CPU使用率

        # 纯数学计算 - 最高效的CPU密集型操作
        result = 0
        for i in range(iterations):
            # 使用位运算和数学运算的组合，最大化CPU使用
            result += (i * 3.14159) ** 0.5
            result = result % 1000000  # 防止溢出

            # 添加更多计算密集型操作
            if i % 1000 == 0:
                # 每1000次迭代进行一次更复杂的计算
                temp = math.sin(result) * math.cos(result)
                result += int(temp * 1000)

        # 简单的内存操作
        temp_list = [result + i for i in range(100)]
        _ = sum(temp_list)  # 确保计算结果被使用


    def simulate_web_workload(self):
        """
        模拟真实的web应用工作负载（保留用于兼容性）
        包括：数据处理、加密解密、图像处理、数据库查询模拟等
        """
        # 1. 数据处理 - 模拟JSON解析和序列化
        data = {
            "timestamp": time.time(),
            "user_id": int(time.time() * 1000) % 10000,
            "action": "api_call",
            "data": [i for i in range(100)]
        }
        json_str = json.dumps(data)
        parsed_data = json.loads(json_str)

        # 2. 加密解密操作 - 模拟安全处理
        import hashlib
        hash_obj = hashlib.sha256()
        hash_obj.update(json_str.encode())
        hash_value = hash_obj.hexdigest()

        # 3. 数学计算 - 模拟业务逻辑
        result = 0
        for i in range(1000):
            result += math.sqrt(i * 3.14159) * math.sin(i * 0.1)

        # 4. 字符串处理 - 模拟文本分析
        text = "This is a simulated web application workload for CPU stress testing"
        processed_text = text.upper().replace(" ", "_").strip()

        # 5. 列表操作 - 模拟数据处理
        numbers = [i * 2 for i in range(500)]
        filtered_numbers = [x for x in numbers if x % 3 == 0]
        sorted_numbers = sorted(filtered_numbers, reverse=True)

    def monitor_system(self):
        """监控系统资源使用情况"""
        print("\n=== 系统监控信息 ===")
        print("时间\t\tCPU%\t内存%\t负载\t进程数")
        print("-" * 60)

        while self.running:
            try:
                # 获取CPU使用率
                cpu_percent = psutil.cpu_percent(interval=1)

                # 获取内存使用率
                memory = psutil.virtual_memory()
                memory_percent = memory.percent

                # 获取系统负载
                load_avg = os.getloadavg()[0] if hasattr(os, 'getloadavg') else 0

                # 获取进程数
                process_count = len(psutil.pids())

                current_time = datetime.now().strftime('%H:%M:%S')
                print(f"{current_time}\t{cpu_percent:.1f}%\t{memory_percent:.1f}%\t{load_avg:.2f}\t{process_count}")

                # 检查是否达到目标时间
                if self.duration > 0 and (time.time() - self.start_time) >= self.duration:
                    print(f"\n达到设定时间 {self.duration} 秒，停止测试")
                    self.running = False
                    break

            except Exception as e:
                print(f"监控错误: {e}")
                time.sleep(1)

    def start_test(self):
        """开始CPU压力测试"""
        try:
            # 启动监控线程
            monitor_thread = threading.Thread(target=self.monitor_system, daemon=True)
            monitor_thread.start()

            # 使用进程而不是线程来更好地利用多核CPU
            # 这样可以避免Python GIL的限制，实现真正的并行计算
            for i in range(self.cores_to_use):
                process = multiprocessing.Process(
                    target=self.cpu_intensive_task,
                    args=(i, self.target_cpu_percent)
                )
                process.start()
                self.processes.append(process)

            # 等待所有进程完成
            for process in self.processes:
                process.join()

        except KeyboardInterrupt:
            print("\n用户中断测试")
        except Exception as e:
            print(f"测试过程中发生错误: {e}")
        finally:
            self.cleanup()

    def cleanup(self):
        """清理资源"""
        self.running = False
        print("\n正在清理资源...")

        # 终止所有进程
        for process in self.processes:
            if process.is_alive():
                process.terminate()
                process.join(timeout=2)
                if process.is_alive():
                    process.kill()  # 强制终止

        # 等待线程结束
        for thread in self.threads:
            if thread.is_alive():
                thread.join(timeout=2)

        print("测试完成，资源已清理")

def main():
    """主函数"""
    parser = argparse.ArgumentParser(description='CPU压力测试Web应用')
    parser.add_argument('--cpu-percent', type=int, default=99,
                       help='目标CPU使用率百分比 (默认: 99)')
    parser.add_argument('--duration', type=int, default=300,
                       help='测试持续时间（秒），0表示无限运行 (默认: 300)')
    parser.add_argument('--cores', type=int, default=None,
                       help='使用的CPU核心数，默认使用所有核心')
    parser.add_argument('--version', action='version', version='CPU Stress WebApp v1.0')

    args = parser.parse_args()

    # 验证参数
    if not 1 <= args.cpu_percent <= 100:
        print("错误: CPU使用率必须在1-100之间")
        sys.exit(1)

    if args.duration < 0:
        print("错误: 持续时间不能为负数")
        sys.exit(1)

    if args.cores and (args.cores < 1 or args.cores > multiprocessing.cpu_count()):
        print(f"错误: 核心数必须在1-{multiprocessing.cpu_count()}之间")
        sys.exit(1)

    # 创建并启动测试应用
    app = CPUStressWebApp(
        target_cpu_percent=args.cpu_percent,
        duration=args.duration,
        cores=args.cores
    )

    app.start_test()

if __name__ == "__main__":
    main()

```

---

## 模拟故障

在 Chaos Server 的 `/Fault_Simulation` 目录下执行：

**开始：**

```bash
python3 cpu_stress_webapp.py --duration 600
```

**结束：** `Ctrl+C`

> `--duration` 时间（秒）可任意设置。

