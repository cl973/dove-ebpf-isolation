# DOVE: Dynamic Overhead Virtualization and Enforcement

**DOVE** 是一个针对 Serverless 和容器化环境设计的轻量级资源隔离系统。它旨在解决 I/O 密集型容器通过软中断过度占用宿主机 CPU 资源，导致服务计费不公和邻居干扰（Noisy Neighbor）的问题。

通过结合 **eBPF** 技术与 **Linux CFS (Completely Fair Scheduler)** 的深度修改，DOVE 能够实时监控网络 I/O 产生的软中断开销，将其精确归属到对应的 Cgroup，并从该 Cgroup 的 CPU 配额中扣除，从而实现真正的精确计费和公平调度。

## 📖 项目背景

在标准的 Linux 容器环境（Cgroups v2）中，网络包处理通常发生在软中断上下文中。这部分 CPU 开销往往被统计为系统全局开销，或者难以精确归因到具体的容器（Cgroup）。

这导致了两个主要问题：
1.  **计费逃逸**：I/O 密集型应用消耗了大量 CPU 处理网络包，但这部分时间未被计入其 Cgroup 使用量，导致云厂商计费偏低。
2.  **资源争抢**：高网络吞吐的容器会占用大量软中断时间，挤占同一宿主机上其他容器的 CPU 时间片，破坏了隔离性。

DOVE 通过建立网络连接与 Cgroup 的映射关系，利用 eBPF 追踪内核网络栈路径，并将统计到的开销反馈给修改后的内核调度器，强制执行带宽控制。

## ✨ 系统架构

DOVE 系统由三个核心组件构成：

1.  **Kernel Patch (Enforcer)**:
    *   修改了 Linux 内核的 CFS 调度器 (`fair.c`)。
    *   在 `__refill_cfs_bandwidth_runtime` 周期性重置带宽时，读取 BPF Map 中的累积开销。
    *   直接从 Cgroup 的 `runtime` 中扣除相应的纳秒数。

2.  **eBPF Monitor (Observer)**:
    *   运行在内核态，挂载于关键网络函数 (`netif_receive_skb`, `dev_queue_xmit` 等)。
    *   **RX 路径**: 计算接收包处理耗时，通过五元组反查 Cgroup。
    *   **TX 路径**: 计算发送包处理耗时，通过 Socket 指针反查 Cgroup。
    *   维护 `pid_to_cgroupid` 和 `connection_to_cgroupid` 映射表。

3.  **User Space Agent (Controller)**:
    *   负责加载 eBPF 程序。
    *   扫描 `/proc` 维护 PID 到 Cgroup ID 的映射。
    *   定期聚合 eBPF Per-CPU Map 中的开销数据，写入全局 Map 供内核调度器读取。

### 项目结构

```
DOVE/
└── network/
    ├── kernel_patch/
    │   ├── 0001-network.patch    # 核心内核补丁 (修改 fair.c)
    ├── linux/kernel/sched/       # 参考内核源码目录
    │   ├── fair.c                # Linux 内核中 CFS (完全公平调度器) 的实现文件
    ├── networking/
    │   ├── networking.bpf.c      # eBPF 内核态程序
    │   ├── networking.c          # 用户态加载器与聚合器
    │   ├── networking.h          # 共享头文件与数据结构
    │   └── networking.skel.h     # libbpf 脚手架 (自动生成)
    ├── scripts/
    │   └── setup_env.sh          # 环境依赖安装脚本
    └── Makefile                  # 项目编译文件
```

## 🚀 快速开始

### 1. 前置条件

*   **操作系统**: Linux (推荐 Ubuntu 20.04/22.04)
*   **内核**: 需要重新编译内核以应用补丁。
*   **权限**: Root 权限。

### 2. 内核补丁应用与编译

DOVE 依赖于对内核调度器的修改。请按照以下步骤操作：

1.  下载与您的系统版本相近的 Linux 内核源码（建议 5.x 或 6.x 系列）。
2.  应用补丁：
    ```bash
    cd /path/to/linux-source
    patch -p1 < /path/to/dove/network/kernel_patch/0001-network.patch
    ```
3.  确保内核配置开启了 BPF 和 Cgroups 相关选项（如 `CONFIG_BPF_SYSCALL`, `CONFIG_CGROUPS`, `CONFIG_CFS_BANDWIDTH`）。
4.  编译并安装新内核，重启系统。

### 3. 环境设置与编译工具

在应用了补丁的新内核环境中，运行脚本安装构建依赖：

```bash
cd ebpf/network/scripts
chmod +x setup_env.sh
sudo ./setup_env.sh
```

该脚本将安装 `clang`, `llvm`, `libbpf`, `linux-tools` 等必要工具。

### 4. 编译 DOVE 用户态程序

回到 `network` 目录进行编译：

```bash
cd ebpf/network
make
```

编译成功后，将在 `networking/` 目录下生成可执行文件 `networking`。

## 💻 使用方法

1.  **启动 DOVE Agent**:
    必须以 Root 权限运行，以便加载 eBPF 程序并修改 sysctl。

    ```bash
    sudo ./networking/networking
    ```

2.  **运行日志**:
    程序启动后，你会看到如下输出：
    *   `BPF program attached successfully.`
    *   `DOVE: cgroup_global_cost map FD set to <fd>` (内核日志，需通过 `dmesg` 查看)

3.  **验证效果**:
    *   创建一个设置了 CPU Quota 的 Cgroup (例如使用 Docker `--cpus` 参数)。
    *   在该容器内运行高强度的网络 I/O 测试（如 `iperf3` 或 `netperf`）。
    *   观察 `dmesg`，你将看到 DOVE 正在从该 Cgroup 扣除运行时的日志（如果在补丁中开启了 `pr_info` 调试）。
    *   该容器的实际 CPU 使用率（包括系统态）将被限制在 Quota 范围内，而不仅仅是用户态 CPU。

## 🔧 技术细节

### 带宽扣除机制
DOVE 通过 sysctl 接口 `dove_global_cost_map_fd` 将 eBPF Map 的文件描述符传递给内核。在 `fair.c` 中：
```c
// 伪代码逻辑
if (dove_global_cost_map_fd >= 0) {
    cost = lookup_bpf_map(cgroup_id);
    if (cost > 0) {
        cfs_b->runtime -= cost; // 核心逻辑：扣除时间片
        clear_bpf_map(cgroup_id);
    }
}
```

### 归因逻辑
*   **TCP**: 通过 Hook `inet_csk_accept` (Server) 和 `tcp_v4_connect` (Client)，结合 `pid_to_cgroupid` Map，建立连接五元组到 Cgroup ID 的映射。
*   **UDP**: 通过 Hook `udp_sendmsg` 建立映射（假设 UDP 服务端在接收前通常会先发送或建立某种状态，或者作为客户端发送）。
*   **PID 映射**: 用户态线程定期扫描 `/proc`，处理新生成的进程，确保短生命周期的容器进程也能被追踪。

## ⚠️ 注意事项

*   **内核版本**: 该补丁基于特定的内核版本开发（请参考 patch 文件头部的内核版本信息），在其他版本上可能需要微调 `fair.c` 的上下文。
*   **性能开销**: 虽然 eBPF 高效，但在极高吞吐量（例如 10Gbps+ 线速小包）场景下，每个包触发 BPF 程序可能会带来一定的 CPU 开销。DOVE 旨在 Serverless 场景下平衡精确度与性能。
*   **Cgroup v2**: 本项目主要设计用于 Cgroup v2 环境 (`/sys/fs/cgroup/...`)。