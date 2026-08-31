# 动态计算适配

Chloros 1.2.0 采用硬件检测和自动处理策略选择机制。处理引擎可自动适配您的硬件——从 Jetson Orin Nano 到多 GPU 工作站——无需任何手动配置。

***

## 工作原理

Chloros 启动时，会分析您的系统：

1. **检测操作系统** — Windows 或 Linux
2. **识别 CPU 核心数和总内存**

3.**检测 GPU 是否存在** — NVIDIA CUDA 能力、显存、型号
4. **识别 Jetson 型号**（如适用）— 通过 `/proc/device-tree/model`
5. **检查温度传感器**（Jetson）—— 用于温度感知处理
6. **选择计算策略** —— 基于所有检测到的硬件
7. **自动配置工作线程数、管道类型和内存分配**

检测到的配置文件会缓存到内存和磁盘中，以便后续运行更快启动：

| 平台 | 缓存配置文件 |
| --- | --- |
| **Linux / Jetson** | `~/.config/chloros/system_config.json`（优先采用 `XDG_CONFIG_HOME`）|
| **Windows** | `%LOCALAPPDATA%\Chloros\config\system_config.json` |

删除该文件可强制进行重新检测——在添加 GPU 或更多内存后此操作非常有用。当缓存由不兼容的旧版本写入时，Chloros 也会自动重新检测。

***

## 计算策略

Chloros 会根据您的硬件从三种计算策略中选择一种：

| 策略 | 选择条件 | 工作进程 | 执行器 | 管道 |
| --- | --- | --- | --- | --- |
| **`GPU_PARALLEL`**| CUDA GPU 报告**12GB+ VRAM**（在 Jetson 统一内存环境下，还需 12GB+ 总共享内存） | `min(4, VRAM ÷ 4GB)`，至少 2 —**在 Jetson 上上限为 2** | `ProcessPoolExecutor`（生成） | `fused_gpu` |
| **`GPU_SINGLE`**| 配备**2-12GB 显存**的 CUDA GPU | 3（I/O 重叠；GPU 访问由信号量串行化）。**内存小于 12GB 的 Jetson 上为 1（顺序）** | `ProcessPoolExecutor`（生成）； 在内存较少的 Jetson 上为进程内顺序模式 | `fused_gpu` / `tiled_gpu` |
| **`CPU_PARALLEL`** | 无 CUDA GPU，或 VRAM 低于 2GB | `max(2, physical cores − 1)` | `ThreadPoolExecutor` | `cpu_fallback` |

`GPU_PARALLEL` 工作进程公式的实际示例：12GB 显存 → 3 个工作进程，16GB 及以上 → 4 个工作进程，任何 Jetson 设备 → 2 个工作进程。

并行性通过 Python 的标准 `concurrent.futures` 实现：GPU 策略使用带有 **spawn** 启动方法的 `ProcessPoolExecutor` （每个工作线程都是一个独立进程，拥有自己的 CUDA 上下文——`fork` 会复制已初始化的 CUDA 状态并导致子进程数据损坏），而 CPU 策略则使用 `ThreadPoolExecutor`。 Chloros 不使用任何第三方分布式框架（如 Ray）。

### 管道类型

* **`fused_gpu`** — 全GPU处理路径。去拜耳化、校正和索引操作在GPU上通过单次融合处理完成。吞吐量最高，但需要最多的VRAM。
* **`tiled_gpu`** — 内存高效型 GPU 路径。将图像按瓦片处理，以适应有限的 GPU 内存。吞吐量较低，但可在内存受限的设备上运行。
* **`cpu_fallback`** — 仅使用 CPU 并通过多线程并行处理。当没有 NVIDIA GPU 可用时使用，并在两个 GPU 处理路径均失败时作为最后的备用方案。

运行时的备用处理链始终为 `fused_gpu` → `tiled_gpu` → `cpu_fallback`。

***

## 手动策略覆盖

设置 `CHLOROS_STRATEGY` 环境变量以强制使用特定策略——这是一种专家级应急方案，适用于自动检测选定的策略不适合当前情况的情形（例如，为保留 GPU 资源供其他任务使用）：

```bash
# Valid values: CPU_PARALLEL, GPU_SINGLE, GPU_PARALLEL
CHLOROS_STRATEGY=CPU_PARALLEL chloros-cli process ~/datasets/flight001
```

该变量匹配时不区分大小写；除这三个名称以外的任何内容都将被忽略，自动检测将继续正常进行。在覆盖设置下，Chloros 仍会为您选择工作进程数量：

| 覆盖值 | 使用的工 作 进 程 数 量 |
| --- | --- |
| `CPU_PARALLEL` | `max(2, physical cores − 1)` |
| `GPU_SINGLE` | 3 |
| `GPU_PARALLEL` | `min(4, physical cores)` |

建议按命令进行设置，而非永久设置，以便常规运行能持续自动适应。

***

## 平台特定行为

| 平台 | 策略 | 工作进程 | 管道 | 备注 |
| --- | --- | --- | --- | --- |
| **Jetson Orin Nano 8GB** | `GPU_SINGLE` | 1 | `tiled_gpu`（顺序处理） | 内存高效模式，每次处理一张图像 |
| **Jetson Orin NX 8GB** | `GPU_SINGLE` | 1 | `tiled_gpu`（顺序） | 共享内存小于 12GB 导致必须顺序处理 |
| **Jetson Orin NX 16GB** | `GPU_PARALLEL` | 2 | `fused_gpu`（并行） | 推荐边缘设备——Jetson 限制为 2 个工作线程 |
| **Jetson AGX Orin 32-64GB** | `GPU_PARALLEL` | 2 | `fused_gpu`（并发）| 最高边缘性能 （Jetson 同样限制为 2 个工作线程） |
| **配备 8GB GPU 的台式机** | `GPU_SINGLE` | 3 | `fused_gpu` / `tiled_gpu` | 3 个工作线程在 I/O 操作上重叠，同时信号量对 GPU 访问进行串行化 |
| **配备 12GB+ GPU 的台式机** | `GPU_PARALLEL` | 3-4 | `fused_gpu` （并发） | 桌面系统最佳性能：12GB → 3 个工作线程，16GB+ → 4 |
| **纯CPU系统** | `CPU_PARALLEL` | 物理核心数 - 1（至少2个） | `cpu_fallback` | 无需GPU，使用线程池 |

{% hint style="info" %}
**Jetson 统一内存**：Jetson 设备共享 GPU 和 CPU 内存。一台 16GB 的 Jetson Orin NX 报告的 VRAM 约为 15.3GB，但这实际上是操作系统和 CPU 进程所使用的同一块物理 RAM。 这就是为什么 16GB 及以上的 Jetson 设备与 12GB 及以上的台式机 GPU 一样符合 `GPU_PARALLEL` 的条件，但工作线程数上限为 2 个——GPU、工作线程及其每个工作线程的 CUDA 上下文都从同一个共享池中获取资源。
{% endhint %}

### 基于显存的 GPU 预算（独立 GPU）

在配备独立 NVIDIA GPU 的 x86_64 主机上，检测到的 VRAM 还会决定显卡可占用的处理资源量以及批处理大小的最大值：

| 检测到的 VRAM | GPU 预算上限 | 批处理大小倍数 |
| --- | --- | --- |
| **8GB+** | 90% | ×2.0 |
| **6-8GB** | 85% | ×1.75 |
| **

3.5-6GB** | 80% | ×1.5 |
| **2-3.5GB** | 75% | ×1.25 |
| **低于 2GB** | 70% | ×1.0 |

独立显卡仅为系统预留 0.5GB 内存，因为它们不与系统内存共享。Jetson 配置文件预留的内存要多得多，且上限更低——请参阅 [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md#per-model-gpu-budget)。

***

## 动态 GPU 内存分配

Chloros 采用 [4 线程处理管道](processing-pipeline.md)：

* **线程 1**（检测）—— 图像加载、EXIF 解析、目标检测
* **线程 2**（校准）—— 反射率校准计算
* **线程 3**（处理）—— GPU 去拜耳化、暗角校正、指数计算
* **线程 4**（导出）—— 文件写入、元数据嵌入

线程 1、2 和 4 对 GPU 的消耗较小；线程 3 则是 GPU 消耗大户。 随着前期管道线程的完成，其GPU资源将**重新分配给剩余的活跃线程**，因此随着运行进程的推进，线程 3 将获得越来越多的内存。

### 分配阶段

| 阶段 | 活跃线程 | GPU 内存分配 |
| --- | --- | --- |
| **早期** | 1、2、3、4 | 分配给所有线程，其中大部分分配给线程 3 |
| **中期-早期** | 2、3、4 | 重新分配线程 1 的份额 |
| **中后期** | 3, 4 | 线程 1 和 2 的份额分配给 3 和 4 |
| **后期** | 3 或 4 | 最后一个活跃线程获得其最大分配量 |

这些数值遵循两条规则：

* 若某个线程是**唯一**处于活动状态的线程，则将其配置文件中的最大分配量授予该线程。
* 当有多个*高负载*的 GPU 任务处于活动状态时，每个高负载任务的基础分配量将在它们之间进行分配（但绝不会低于其配置的最小值）。

运行时实际使用的值是平台配置文件分配量与 GPU 内存监视器实时建议值中的**较低者**，因此繁忙的显卡始终优先于过于乐观的配置文件。***

## 纹理感知处理

纹理感知去拜耳算法（**仅限 Chloros+** — `--debayer texture-aware`）运行一个 AI/ML 去噪模型，该模型每次复制大约需要 1.75GB 的 FP16 格式 VRAM，因此其 GPU 内存占用远高于标准方法：

* **VRAM 低于 7GB**的系统会以**同步循环方式，每次处理一张图像** 来执行“纹理感知”处理——无法同时容纳多个模型副本，且工作线程池只会增加资源竞争
* **VRAM 7GB 以上** 的系统可以并行处理“纹理感知”模式，但与“标准”模式相比，工作线程数量会减少
* 在 **Jetson** 上， “Texture Aware”始终被固定在单个工作线程上，而在低功耗机型（Nano、Orin Nano）上，系统还会自动应用 GPU 频率限制——详见 [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md#gpu-frequency-cap-for-texture-aware-on-nano-and-orin-nano)***

## 热管理（Jetson）

Jetson 设备存在热限制，尤其是在封闭或机载部署环境中。Chloros 会监控 Jetson 的板载温度传感器，并自动调整批处理大小：

| 温度 | 响应 |
| --- | --- |
| **&lt; 70°C** | 正常运行 — 全速 |
| **70°C**（警告）| 批处理大小逐步缩减（70°C 至 80°C 区间内从 100% → 50%）|
| **80°C**（危急） | 激进限速（80°C 至 90°C 区间内从 50% 降至 0%） |
| **90°C**（关机） | 完全停止 GPU 处理 |

在散热条件良好的台式机系统上，极少触发热节流。

***

## 内存压力处理

Chloros 在处理过程中持续监控 GPU 内存，并根据三种级别采取相应措施。

**批处理大小。** 批处理的起始大小为 8 张图像乘以上表中的平台系数。 Chloros 随后检查可用 VRAM，预留其中 20% 用于 PyTorch 自身的开销，并假设每张 12MP 图像约占用 100MB GPU 内存——批处理大小取内存限制值与平台基准值中的较小者。 该值绝不会低于 1。**预先缩减。**当**VRAM 利用率超过 85%** 时，系统会在任何任务失败之前缩减批量大小。**按线程分配削减。** 随着实时利用率攀升，每个线程的 GPU 配额将被缩减：利用率超过 80% 时缩减至 0.75 倍，超过 90% 时缩减至 0.5 倍。 监控阈值分别为 70%（保守值）、85%（正常运行阈值）和 95%（内存不足风险）。**内存不足的退避与恢复机制。** 若仍发生内存不足事件：

* 批处理大小将**减半**，且在每次连续发生内存不足时再次减半——此后每次成功的批处理都会使该惩罚措施后退一步
* 活动线程的内存分配将削减至当前值的70%，且分配器切换至保守策略，在连续成功分配后再次放宽限制
* 在严重压力下，管道会从 `fused_gpu` 降级至 `tiled_gpu`，作为最后手段则降级至 `cpu_fallback`

**主机内存（Jetson）。** 在处理之前，CLI会根据您的图像数量和去拜耳模式估算主机内存峰值，如果内存加上基于文件的交换空间可能不足，则会发出警告，并打印出添加交换空间的确切命令——请参阅[NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md#swap-warning-and-recommendations)。***

## 监控计算适配情况

### 系统诊断

`chloros-cli selftest` 是确认计算层当前状态的最快捷方式：

```bash
chloros-cli selftest
```

其 7 项检查涵盖版本、端口可用性、后端启动、`/api/test`、系统信息、去噪器模型存在性以及 CUDA + 去噪器就绪状态。检查 5 会直接输出硬件行：

```
      GPU: NVIDIA RTX A4000, CUDA: True, PyTorch: 2.7.0
```

检查 7 会输出 `CUDA: <bool>, Denoiser: <bool>` —— 这两项都必须为真，Texture Aware 功能才能被使用。

### 后端日志

策略和工作进程数量在每次运行开始时由后端内部确定——没有 CLI 标语来宣布这些信息。 当出现异常行为（如 GPU 路径回退、内存不足（OOM）、降噪器无法加载等）时，相关信息会显示在该会话的后端日志中：

| 平台 | 日志位置 |
| --- | --- |
| **Linux / Jetson** | `~/.cache/chloros/logs/backend_<YYYYMMDD_HHMMSS>.log`（每次启动生成一个文件） |
| **Linux、CLI-started 后端** | 另见 `~/.chloros/backend.log` |
| **Windows** | `%LOCALAPPDATA%\Chloros\logs\` |

### 实时进度

运行期间，CLI 会通过服务器发送事件（Server-Sent Events）实时流式传输各线程的进度（检测、分析、处理、导出）——这是判断线程 3 是否为瓶颈的实用依据。参见 [处理管道](processing-pipeline.md)。

***

## 后续步骤

* [处理管道](processing-pipeline.md) — 了解 4 线程管道架构
* [NVIDIA Jetson 指南](../linux/nvidia-jetson-guide.md) — 针对 Jetson 的部署与优化
* [CLI：命令行](../CLI.md) — CLI 指南
* [CLI 参考](../reference/cli-reference.md) — 1.2.0 版本的完整命令列表
