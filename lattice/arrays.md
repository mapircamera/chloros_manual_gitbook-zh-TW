# 多摄像头阵列

LATTICE **阵列**是指两个或多个 LATTICE 摄像头连接成一个同步单元。其中一台摄像头作为**主摄像头**： 它会在共享的同步线上（默认**Line2**）发送硬件 GPIO 触发脉冲，从而确保每个成员捕捉同一瞬间。Chloros 增加了 PTP 时间同步、实时预览（按相机划分的图块或单个对齐的多波段合成图像）以及同步捕获——每次捕获过程都会生成一个**帧组**，其中所有相机共享相同的时间戳和帧 ID（在捕获输出中显示为 `fid:N`）。

阵列是单色（M3M）相机生成植被指数的方式——每台相机贡献一个波段，阵列将它们对齐成多波段堆栈。参见 [单色相机与植被指数](mono-indices.md)。

连接数组有三种等效的方法，它们均运行相同的“智能预处理”流程：

| 界面 | 入口点 |
| --- | --- |
| 图形界面 | “相机”选项卡 → **连接阵列**（蓝色按钮） |
| CLI | `chloros-cli lattice array-connect --serials SN1,SN2,…`（第一个序列号 = 主相机） |
| Python SDK | `connect_array(serials=[…])` → `ArraySession`（第一个序列号 = 主设备） |

“智能预处理”将按以下顺序执行： 网络功能检测（ICMP DF 探测 + GVSP 探测）、同步层选择、自动缩减帧大小以适应传输线、启用 PTP、按摄像头自动选择像素格式、根据各摄像头保存的状态进行自动曝光初始化，以及在 Line2 上配置 GPIO 触发。

{% hint style="info" %}
在上述任何操作生效之前，必须确保摄像头在链路上可达——有关发现、寻址以及首次连接校准下载的详细信息，请参阅 [连接摄像头](connecting.md)。 对于多摄像头系统，主机网卡的接收环设置与链路速度同样重要；完整的“症状→解决方案”对照表请参见[CLI 参考 § 主机网卡设置与调优](../reference/cli-reference.md#host-nic-setup--tuning-lattice-arrays)。
{% endhint %}

## “阵列连接”对话框

“摄像机”选项卡 → **连接阵列**将打开一个三步向导：**选择 → 显示模式 → 设置**。

### 步骤 1 — 选择主摄像头和从摄像头



<!-- SCREENSHOT-NEEDED: Array Connect dialog, Select scene, with 3-4 LATTICE cameras discovered. Table showing Camera / Serial / IP / Master radio / Slave checkbox columns, with the green "GPIO master detected — selections pre-populated" probe banner visible above the table. -->对话框一打开就会立即扫描网络（“正在扫描网络...”），然后探测 GPIO 触发接线（“正在探测 GPIO 接线...”）。构建阵列至少需要 **2 台摄像头**。

当连接检测成功时，系统会自动预填角色选择项，并显示以下三种提示信息之一：

| 提示 | 含义 |
| --- | --- |
| “检测到 GPIO 主设备 — 选项已预填” (绿色) | 探测器已识别触发拓扑结构；“主设备”和“从设备”复选框已自动勾选。 |
| “未检测到主设备 — 请检查 GPIO 线缆”（橙色） | 没有摄像头检测到触发脉冲；请检查同步线缆。您仍可手动选择角色。 |
| “无同步线缆：{serials}”（橙色） | 列出的摄像头未连接同步线缆。 |

相机表包含以下列：**相机 / 串口号 / IP / 主相机（无线电） / 从相机（复选框）**：

* 请精确选择 **一个主相机**和**一个或多个从相机**。再次点击当前主相机的无线电图标可清除该选项。
* 标记为 **“无同步线”** 的摄像机绝不能被选为从机——没有触发线连接的从机将永远在同步线上等待，并输出无效信号。请将该摄像机作为独立摄像机连接。
* 已作为独立摄像机连接的摄像机*不会*被禁用：“阵列连接”功能会释放独立会话，并将摄像机重新纳入阵列中。

**下一步： 显示模式 →**在选定一台主摄像机和至少一台从属摄像机后生效。**重新扫描** 将重新执行设备发现和布线检测。

{% hint style="warning" %}
在扫描或探测进行期间，**取消** 功能将被禁用——在探测过程中取消操作可能会导致 LATTICE 摄像机固件中的SDK发生崩溃。请等待旋转图标停止。
{% endhint %}

### 步骤 2 — 显示模式



<!-- SCREENSHOT-NEEDED: Array Connect dialog, Display Mode scene, showing the two selectable cards ("Separate Cameras" and "Combined Cameras") with Combined selected/highlighted as the default. -->| 模式 | 显示效果 |
| --- | --- |
| **独立相机** | 每台相机一个实时图块，所有相机同时触发以确保帧同步。每台相机保留各自的颜色和设置。 |
| **组合相机** *（默认）* | 一个图块渲染对齐后的多波段 NDVI /index 合成图像。相机共享阵列颜色。 |

显示模式仅改变实时预览的呈现方式——两种模式下的采集行为完全相同。

### 步骤 3 — 阵列设置与预期效果

<!-- SCREENSHOT-NEEDED: Array Connect dialog, Settings scene, healthy state: left column with ROI / Binning / Pin resolution / Trigger Rate controls, right "Projected Outcome" column showing green "Simultaneous capture" tier, an fps range, the NIC line, the "Sim-emit burst" line, and the "Wire budget" line with a checkmark. -->

进入此场景时，Chloros 会向后端请求**建议**，并自动应用一套适合您 NIC 接收环的 ROI + 像素合并组合（它更倾向于使用像素合并而非 ROI 裁剪，因为像素合并能保留完整的视场）。 您所做的每次更改都会实时重新运行分析，并更新右侧的**预计结果**面板。

左侧栏 — 设置：

| 控件 | 选项 | 默认值 | 备注 |
| --- | --- | --- | --- |
| **ROI（视野）** | 全景 (2048×1536) / 半景 (1024×768) / 四分之一 (512×384) | 全景 | 传感器裁剪：以原生像素间距将图像裁剪为半景或四分之一区域。 |
| **像素合并** | 1× / 2×（2×2 求和） / 4×（4×4求和） | 1× | 硬件像素合并：2×2 = 以四分之一的线成本获得全视场；4×4 = 以十六分之一线成本获得全视场。若相机不支持像素合并，则此选项隐藏。 |
| **线侧图像**（读出） | — | — | 实际通过线路传输的像素合并后宽度×高度，被截断为16的倍数（最小64）。 |
| **引脚分辨率**| 复选框 | 关闭 | 当预计帧率低于**

1.5 fps** 时，Chloros 通常会在连接时自动提高像素合并倍数。锁定功能可保持您选择的帧尺寸并接受较低的帧率——并将超额配置转为硬性连接拒绝，而非自动降速。 |
| **触发速率** | 0.5–60 fps，步长 0.1 | 留空 = 自动 | 主控器的触发速率。留空则由 Chloros 自动推导。 |
| **带宽预算**| 20–2000 MB/s，步长 10 | 留空 = 自动 | 主机实际可吸收的带宽，单位为 MB/s ——**这是整个数组分配所依赖的唯一关键数值。** 由网络适配器自动检测。若阵列报告帧损坏，请降低该值：检测到的数值往往高估了 USB 适配器和共享交换机的实际能力。修改后将实时重新运行预测。 |

右侧列 — **预测结果**：

* **同步层级** — “同时捕获”（绿色）、“同时捕获（FTD交错发射）”（绿色）、“交错捕获（100 毫秒漂移）”（琥珀色）或“配置过大”（红色）。
* ****帧率预测** — 以范围形式显示（“暗 → 亮”），因为同步阵列的速率受最慢摄像机曝光时间的限制。
* **网卡行** — 链路速度和持续吞吐量（“网卡 {mbps} Mbps · 持续 {N} MB/s”）。
* **同步发射突发检查** — 主机的网卡环路能否同时接收来自所有摄像头的突发数据（“同步发射突发：X MB · 网卡环路可用带宽：Y MB ✓/✗”）。
* **线速预算检查** —— 稳态聚合需求与防冲突线速上限的对比（“线速预算：{n} 台摄像机要求的 {demand} MB/s · 上限 {ceiling} MB/s ✓/✗ 超额订阅”）。
* **“该链路最大摄像头数量：{n} — 由每台摄像头的带宽下限决定，因此分组处理不会提高该上限。”** — 当你接近（或超过）摄像头数量上限时显示。
* **“此设置下将出现帧丢失。”**— 红色警告，附带后端给出的原因，以及阻塞项列表和蓝色的**修复建议**（“让该阵列适应网络” / “解锁同时捕获”）。**“应用并连接”** 按钮在生成预测结果前处于禁用状态，其标签会说明拒绝的原因：

| 按钮标签 | 含义 | 实际解决方法 |
| --- | --- | --- |
| “正在分析...” | 分析仍在进行中。 | 请等待。 |
| **“此网络中的摄像头数量过多”**| 阵列对网络带宽的占用超额（聚合检查失败）。 | 减少摄像头数量、端到端启用巨型帧，或使用更快的网卡。**缩小 ROI 并无帮助** —— 详见下文。 |
| **“缩小 ROI 以启用”** | 这些设置下帧会丢失（突发/环检查失败）。 | 缩小 ROI、提高像素合并度，或修复网卡接收环。 |



<!-- SCREENSHOT-NEEDED: Array Connect dialog, Settings scene, over-subscribed state: red "Wire budget ... over-subscribed" line, the "Max cameras on this wire" hint, and the Apply button reading "Too many cameras for this network". Reproduce by configuring more cameras than the 1 GbE ceiling (e.g. 7+ cams at 1500 MTU) or with CHLOROS-simulated models via `lattice analyze-array`. -->连接过程中，可能会出现一个带绿色 **校准下载面板** 及逐个串口进度条：当摄像头首次连接到设备时，Chloros 会通过千兆以太网从摄像头下载约 3.8 MB 的出厂校准包 （每台相机约需 70 秒）。已缓存的相机不会显示此面板。参见 [连接相机](connecting.md)。

## 带宽：可支持的摄像头数量

阵列的承载能力取决于网络线路本身，而非Chloros，因此规划数据请参阅硬件手册：**[阵列带宽规划](https://mapir.gitbook.io/lattice-camera/setup/array-bandwidth-planning)**。

Chloros 如何处理这些数据：连接对话框会运行网络探测，预测可实现的帧率，并选择合适的层级。如果阵列对线路的占用率超过其带宽，系统会拒绝连接，而不是默默丢弃数据包——请参阅上文所述的“预测结果”面板。

## 当帧丢失时

摄像机可能因两种截然不同的原因缺失在已发布的组中，
且需要截然相反的解决方法。Chloros会分别统计这两种情况，而非报告一个
既未说明原因的“不完整”数值：

| 发生的情况 | 含义 | 排查位置 |
| --- | --- | --- |
| **损坏**— 帧已到达但结构损坏 | 网络路径上的 GVSP 数据包丢失 |**带宽预算**、网卡接收环、巨型帧、交换机 |
| **未到达**— 完全没有帧到达 | 摄像机未触发，或未输出任何数据 |**M8同步线**、同步线路、所有成员是否处于就绪状态 |

在阵列传输期间，系统每10秒重新评估帧分割情况。若比例超过5%，
系统将记录具体数值，并针对每台摄像机在首次出现时报告每个损坏的缓冲区，
随后每分钟汇总一次，以确保长时间会话记录的可读性。

**如果损坏帧数为零且“未到达”为零，则说明触发和电缆同步均正常**，所有丢失的帧均源于网络路径。解决方法是降低**线缆预算** 并
重新连接。

{% hint style="warning" %}
**降低触发率无法解决帧损坏问题。** 摄像机的数据包
分发节奏在连接时仅设置一次。降低触发率只会改变突发传输
的频率，而不会影响突发数据本身传输到网络线的速度。 在一套经过测量的 4 台摄像机系统中，
将触发率降低 5 倍没有任何效果，而将带宽预算从 240 降低到
200 MB/s，则使该系统的帧损坏率从 10.4% 降至零。
{% endhint %}

正在运行的数组无法自行重新规划——请断开并重新连接，以便连接时的
选择器能够根据新的带宽预算进行工作。

### USB 网络适配器的上限为 200 MB/s

USB 以太网适配器虽然标称其 *以太网* 链路速率，但其实际
可持续传输速率受限于 USB 总线及其驱动程序。一款 USB 10GbE 适配器曾被宣称
具有约 1000 MB/s 的吞吐量——这一数值从未被实际测得——而针对该虚幻冗余空间进行
四台摄像机时，基于该虚幻的冗余空间进行限速会导致 6–18 % 的帧出现损坏，而摄像机阵列
仍报告目标帧率正常。目前，USB 连接的适配器速率上限为
**200 MB/s**。该上限是绝对值而非百分比，因为限制因素在于
总线：USB 1 GbE适配器可获得约80 MB/s的吞吐量，因此不受影响。

如果您的主机速度确实快于该上限，请将**线速预算**调高以反映实际情况。

## PTP 时间同步

帧的 *同步* 由硬件触发信号实现；**PTP**（IEEE 1588 PTPv2）为所有设备提供一致的 *时间戳*。该功能在阵列连接时默认启用：

* **Chloros 主机后端运行 PTP 主时钟**。LATTICE 相机和 DAQ-E 光传感器在域 0 中对其进行从属同步，因此图像时间戳和 DAQ 光谱数据均基于同一时钟 （~1 毫秒）。
* `--no-ptp`（CLI）在实验台工作中会禁用该功能——此时不同相机之间的时间戳将**无法**进行比较。
* 通过 CLI 检查同步状态：

```bash
chloros-cli time-sync status     # grandmaster state, clock identity
chloros-cli time-sync peers      # slaves seen (cameras + DAQ-E sensors)
chloros-cli time-sync cameras    # per-camera PtpStatus / PtpOffsetFromMaster / PtpMeanPathDelay
```

“相机”选项卡本身没有 PTP 指示器；该选项卡中显示的各相机同步信息包括只读的 **角色**（主/从）、**同步线路** 以及阵列的“功能”层级。DAQ-E 的 PTP 状态显示在“光传感器”选项卡的传感器详细信息中。

## 实时阵列视图

<!-- SCREENSHOT-NEEDED: Cameras tab with a connected combined array: sidebar showing the ARRAY row (color badge, array name, "DAQ · on" pill) with indented member camera rows, and the main area showing the combined index composite tile with the LUT-colored NDVI render, top-left array name pill, and top-right fps readout. -->

主视频区域提供两种布局（可在顶部栏切换）：**网格视图**（每个图块即为一个单元格；解锁网格挂锁后可通过拖拽重新排序）和**列表视图**（顶部为全宽阵列，下方显示一台活动摄像头）。**画面缩放**滑块用于调整图块大小；当单元格宽度小于 200 像素时，名称/帧率叠加层会自动隐藏。**独立模式**下，每台摄像机显示一个图块。 每个图块上叠加显示：

* 摄像头名称（左上角），
* **帧率读数**（右上角）——这是后端报告的摄像头*真实抓取速率*，而非预览轮询速率（实时预览无论抓取速率如何，均上限为 30 fps），
* 状态点 — 绿色（正在流式传输）/ 琥珀色（正在加载）/ 红色（错误），
* 当 2 秒内未收到新帧时显示 **过期帧旋转图标** — 在任何连接/断开后约 5 秒内属于正常现象，此时后端正在重新平衡各摄像头的带宽分配。**组合模式**显示单个复合图块：后端会进行去拜耳化、缩放、对齐、降噪，转换为各波段辐射度（当绑定光传感器时还会加上 DLS 反射率），评估数组的索引表达式，应用 LUT，并将结果以 MJPEG 格式流式传输。 在渲染出第一个对齐帧之前，该图块会显示其状态：“正在准备阵列……”、“正在校准对齐……”、“等待第一帧……”；或者——如果自动对齐重试配额（约30秒）已用完——“需要对齐”，并显示一个**校准对齐**按钮。

有关组合模式的实用信息：

* 合成图像与**主**摄像机的帧对齐。 在合成图像上进行的 AE-ROI 对焦和点测光，对主摄像头而言是精确的，对从属摄像头而言则是近似的；使用**分屏视图**（阵列设置 → “显示成员摄像头”）可获得每个摄像头的像素级精确图块，而无需打开额外的摄像头连接。
* **显示图层**（阵列设置； 默认关闭）可让您选择前景和背景图层——任何成员摄像机或**索引**。当前景=索引时，超出 LUT 最小/最大范围的像素将显示背景图层。
* **渲染分辨率**（默认 720p）同时设定实时流的高度 *和* 保存的合成图像导出尺寸。每台摄像机的图像始终以全分辨率导出。
* 对齐结果按会话计算且不会持久保存——请参阅阵列设置面板中的“对齐”部分，了解均方根残差（RMS）及“重新校准”按钮。

## 采集：监控与分析

阵列采集表面可明确划分为 **监控级**（记录所见画面）和**分析级**（记录原始数据，稍后校准）：

| 工作流 | 级别 | 保存内容 | GUI | CLI |
| --- | --- | --- | --- | --- |
| **采集**（静帧） | 分析级 | 每次扫描一个同步帧组；每个选定的导出级别（原始/去拜耳化/辐射度/反射率/预览/索引）下的每台相机文件 + `.daq` 辅助文件 |**“全量捕获”**按钮 + 捕获设置 | `lattice array-capture` |
| **录制索引视频** | 监看 | 显示的实时组合索引合成画面 — 8位，预览分辨率，已烘焙LUT；需打开实时流 | ● 录制索引视频 （组合数组） | `lattice array-record` |
| **原始连拍 → 生成视频**| 分析 | 以全帧率捕获的原始传感器帧 + 清单 + `.daq`，随后离线重建为经过校准的辐射度/反射率/指数视频，并与数据采集（DAQ）读数实现时间同步 | ⦿ 录制原始连拍 →**生成视频** | `lattice array-burst` → `lattice array-build-video` |

经验法则：如果像素数据将用于*测量*，请使用“捕获”或“连拍”（分析级）；如果仅需*查看或演示*阵列所捕获的内容，请录制索引视频（监控级）。

### 捕获设置（GUI）



<!-- SCREENSHOT-NEEDED: Capture Settings pane (gear next to Capture All) with a connected array: capture-mode buttons (Single/Continuous/Interval), the bulk export-type toggle row, the Fastest Capture toggle, and the per-array group card showing the Aligned checkbox and the "Record index video" / "Record raw burst" buttons. -->**捕获全部**旁边的齿轮图标可打开“捕获设置”面板（需要打开项目——捕获数据将保存在该项目中）：

* **捕获模式**：**单次**（单轮捕获） /**连续**（连续捕获； 受捕获次数（默认 1）或持续时间（默认 10 秒）限制）/**间隔**（延时摄影：每 X 间隔捕获 N 帧，共计 Y 帧，默认每 5 秒 1 帧，持续 1 分钟）。
* **每台相机的导出类型**： 原始数据、去拜耳化、辐射度、反射率、预览、索引 — 所有适用的选项默认均处于启用状态。对于RGB-filter相机，辐射度/反射率选项被隐藏；**反射率仅在相机具备DAQ光传感器时显示**（无论是相机自带的还是从阵列继承的）；索引选项需要配置索引表达式。
* **对齐**（按阵列设置，默认**开启**）：将成员导出数据变换为阵列的对齐配置文件，确保导出数据实现像素级对齐。Raw格式始终保持未变换状态，但会在元数据中携带变换信息。
* **最快采集**（开关）：仅采集原始数据 + 分配的DAQ读数 + 免费的组合指数合成数据，在采集时跳过校准计算以实现最高速率——稍后可从保存的`.daq`文件中重建辐射度/反射率/指数。
* 选择项随项目保存。隐藏或暂停的相机将被跳过。

等效的 CLI（相同后端端点，相同语义）：

```bash
# One synced group, every applicable export level per camera (the default)
chloros-cli lattice array-capture -o output/

# Interval timelapse: one reflectance pass every 10 s for 5 minutes
chloros-cli lattice array-capture --interval 10 --duration 300 --processing reflectance -o timelapse/

# Fastest grab for a moving rig — raw + .daq now, calibrate later
chloros-cli lattice array-capture --fastest -o flightline/

# 30-second monitoring clip of the combined index view, plus a GIF
chloros-cli lattice array-record --duration 30 --fps 10 --gif -o monitoring/

# 5-second analysis-grade raw burst, then build the combined index video
chloros-cli lattice array-burst --duration 5 --build --products combined:index --fps 10 -o capture/
```

捕获时的 TIFF 压缩格式为 `deflate`（无损，默认）或 `none` — 完整的标志表、捕获文件夹结构以及重新处理规则请参见 [CLI 参考文档](../reference/cli-reference.md#capture-modes-recorders--offline-reprocess)。

## 配对 DAQ 光传感器

反射率和照度校正的预览需要来自 DAQ 传感器（在 **光传感器** 选项卡中连接）的下射光数据：

* 侧边栏的 **数组行**会显示一个**“DAQ · 开/关” 圆点图标**— 当设置了阵列级光传感器**或** 任何成员摄像机拥有独立传感器时，该按钮处于 *开启* 状态；其工具提示会精确列出哪个传感器为哪台摄像机供数据。
* 在阵列设置中 → **环境光传感器**→**光传感器** 下拉菜单中进行全阵列分配。 该选择将随项目保留，并传播至每个成员摄像头，同时各摄像头仍可通过其自身的传感器覆盖此设置。
* 其下方的状态行报告实时状态：**关闭**→ “正在等待首个光谱……” →**“活动 — 数组中的所有摄像机均已进行照度校正”** → 或者，如果过去 3 秒内未收到新的光谱数据，则会显示过期提示——系统将继续使用上次读数（在捕获路径中，读数永不过期）。

当分配了传感器后：“反射率”导出类型将可用，实时预览将经过照度校正，预测性自动曝光可使用该光谱，且每次反射率捕获都会将实际使用的 DAQ 读数作为 **`.daq` 旁载文件**写入图像旁边，以便日后可对该捕获进行重新处理。

## `array-connect` CLI 选项

| 标志 | 默认值 | 描述 |
| --- | --- | --- |
| `--serials SN1,SN2,…` | 自动发现所有 LATTICE 摄像头（需 ≥2 个） | **第一个串行端口为 MASTER.** |
| `--line {Line0,Line2,Line3}` | `Line2` | GPIO 同步线。 |
| `--target-fps F` | 自动 | 主控器触发频率。 |
| `--binning {1,2,4}` | 自动 | 硬件分档。 |
| `--force-tier {sim-capture-sim-emit, sim-capture-ftd-stagger, slip-emit-and-capture}` | 自动 | 专家级同步层选择器覆盖设置。 |
| `--wire-ceiling-mbps MB_PER_S` | 自动检测 | 主机线速预算（单位：MB/s）——即“CLI”中“**线速预算**”字段的对应值。若数组报告帧损坏，请降低该值。该设置随项目保存，因此后续重新连接时将自动恢复。 |
| `--no-recommend` | 关闭 | 跳过网络分析步骤。 |
| `--no-ptp` | 关闭 | 禁用 PTP（此时不同摄像机之间的时间戳将无法比较）。 |

`lattice array-list`、`array-status` 和 `array-disconnect` 用于管理持久会话。 完整的子命令参考，包括对齐（`align-calibrate` / `align-apply`）和网络工具，请参见 [CLI 参考 § chloros-cli lattice](../reference/cli-reference.md#chloros-cli-lattice) 中；SDK中的等效命令（`connect_array`、`ArraySession`、 `attach_array`、`analyze_array_network`) 则收录于 [SDK 参考文档](../reference/sdk-reference.md)。 根据 Python，线缆预算为 `connect_array(..., wire_ceiling_mbps=120)`，而“已启用但损坏/从未到达”的分类情况详见 [`/api/camera/array/<id>/capability`](../reference/sdk-reference.md#array-health--which-subsystem-is-losing-frames)。
