# Chloros Python SDK 参考文档

**版本：**

1.2.0**生成时间：**2026-07-29 19:19 ·**修订时间：** 2026-08-30**包：** `chloros-sdk` (PyPI)**适用对象：** 针对大型语言模型（LLM）使用进行了优化；易于人类阅读。**范围：** `import chloros_sdk` 公开的所有类、 函数和辅助函数，并附有可直接复制粘贴的示例，涵盖图像处理、单摄像头控制、同步数组、数据采集（DAQ）传感器以及项目自动化。

若您只需了解重点内容，请跳转至：
- [安装与快速入门](#installation)
- [适用于 LATTICE 阵列的 Smart-Connect](#smart-connect-for-lattice-cameras)
- [DAQ 传感器专题](#daq-sensor-sessions)
- [项目自动化](#project-automation--chlorosproject)
- [Smart-AE / Smart-Capture](#smart-ae--smart-capture)

---

## 60 秒速览架构

SDK 是在 Chloros 后端（与桌面 GUI 和 CLI 使用的 Flask 服务器相同）之上构建的一层轻量级 Python 接口。要实现自动化，您只需导入 `chloros_sdk` 并调用高级方法； 在底层实现中，每次调用都会转化为向端口 5000 上的本地后端发送的 HTTP 请求 — `http://127.0.0.1:5000/api/...` （刻意未采用 `localhost`，因为该名称在 Windows 上会首先解析为 `::1`，且针对仅支持 IPv4 的后端，每次请求耗时约 2 秒）。 后端拥有硬件池——摄像头、数据采集传感器、对准配置文件、帧缓冲区——因此SDK脚本可以与GUI共存，而无需争夺串行端口或网卡带宽。

您将使用以下三个操作界面：

1. **`ChlorosLocal` + 免费函数**（`process_folder`、`process_lattice_capture`）——图像处理管道。通过一次 Python 调用，即可对整个文件夹执行校准/去拜耳化/索引导出。
2. **Smart-connect 句柄**（`connect_camera`、`connect_array`、`connect_daq_sensor`）——为实时硬件建立持久后端会话。 与 GUI 相同的“smart-prep”流程：网络探测、层级自动选择、PTP、AE 初始化、GPIO 触发配置。
3. **`ChlorosProject` / `open_project`** — 加载已保存的项目（包含 `cameras.json` + `sensors.json` + `project.json` 的文件夹），一次性连接所有设备，并通过命名句柄进行捕获。

如果尚未有后端处于监听状态，界面 1 和 2 将 **自动启动本地后端** （即 GUI/CLI 调用的同一捆绑二进制文件）——因此，在全新终端中运行该脚本即可生效，无需您事先启动后端。若要禁用此功能，请传入参数 `auto_start_backend=False`（例如，当指向远程后端时，该后端永远不会被启动）。参见 [后端自动启动]（#backend-auto-start）。Surface 3 的行为有所不同：`open_project()` 不接受 `auto_start_backend` 参数，而 `connect_all()` 永远不会启动后端——它会向 `http://127.0.0.1:5000` 一次，若无人响应， 则会静默地回退到直接（无后端）的 `lattice_sdk` 设备控制模式。只有 `proj.process()` 和 `stream(..., overlays=True)` 会延迟构建一个 `ChlorosLocal()`（该进程会自动启动）。

这三个进程均受身份验证限制：需在该机器上运行一次 `chloros-cli login`，或通过桌面图形界面登录。若在没有有效会话的情况下调用 `SDK`，将触发 `ChlorosAuthenticationError` 异常。

要求：
- Python 3.7+（如软件包所声明；在 3.10 版本上开发/测试）
- 本地已安装 Chloros Desktop（后端二进制文件包含在安装程序中）
- 有效的 Chloros+ 登录账号。访问 SDK / CLI 的最低要求为 **Copper**级或更高（Copper / Bronze / Silver / Gold）；免费的**Iron**级不具备访问 SDK / CLI 的权限。此限制在**服务器端**强制执行：每个带有 SDK / CLI 标记的请求必须同时包含有效会话和付费套餐，否则后端将返回 `403` 并附带 `error_code: PLAN_UPGRADE_REQUIRED`（显示为 `ChlorosLicenseError` 由 `ChlorosLocal` 返回，并由 `connect_*` 辅助函数返回为 `ChlorosConnectError`）。已注销的调用者将收到 `401` / `AUTH_REQUIRED` (`ChlorosAuthenticationError`) ——这两者有所区别，因为重新运行 `chloros-cli login` 可以修复前者，但无法修复后者。
- 在套餐的宽限期内支持离线使用：许可层级将从服务器验证缓存（5 分钟）或已签名、机器绑定的许可缓存（30 天（针对月度套餐）， 年费计划则至订阅到期日）。当该宽限期届满时，套餐将转为免费状态，且SDK / CLI的访问将暂停，直至该设备能成功连接服务器一次。`chloros-cli status` (`GET /api/license-status`)在免费套餐下仍可访问，因此原因显而易见 ——这是唯一一条不受层级限制的 SDK / CLI 路由。
- Windows 10/11 64 位、**Ubuntu 22.04 LTS 或更新版本**，或 Jetson（JetPack 6）。**不**支持 Ubuntu 20.04：`.deb` 的依赖项源自后端链接的库（包括 `libc6 (>= 2.34)`），而 Focal 版本自带的 glibc 版本为 2.31。

---

## 安装

Python（SDK）是在 Chloros 后端之上构建的一层轻量级 Python 接口。 对于除少数仅涉及数据采集（DAQ）的工作流以外的所有情况，您需要在本地安装 **Chloros 桌面软件包**（Windows 安装程序或 Linux `.deb`）——该软件包提供了后端二进制文件、用于 LATTICE 相机的 Arena SDK 运行时以及校准包。

最新下载：[`https://mapir.gitbook.io/chloros/download`](https://mapir.gitbook.io/chloros/download)

### 步骤 1 — 安装 Chloros 平台软件包

#### Windows (.exe)

1. 从下载页面下载 `Chloros-Setup-x.y.z.exe`。
2. 运行安装程序并按照向导操作。默认安装路径为 `C:\Program Files\MAPIR\Chloros\`。
3. 至少启动一次 Chloros，并使用您的 Chloros+ 账户登录。

#### Linux amd64 (.deb)

```bash
sudo dpkg -i chloros-amd64.deb
sudo apt-get install -f         # only if dpkg reports missing dependencies
chloros-cli --version
chloros-cli login user@example.com 'YourPassword'
```

#### Linux arm64 — Jetson (JetPack 6)

```bash
sudo dpkg -i chloros-arm64-jp6.deb
sudo apt-get install -f
chloros-cli --version
chloros-cli login user@example.com 'YourPassword'
```

### 步骤 2 — 安装 Python SDK

**Chloros 安装程序附带了一个匹配的 SDK wheel 包。** 每个 Windows 安装程序和 Linux .deb 包都会在磁盘上放置一个 `chloros_sdk-X.Y.Z-py3-none-any.whl`，其版本与 GUI / CLI / 后端版本完全一致。您无需追踪 PyPI 即可保持同步。

#### Windows

安装程序会使用您系统中的 Python 自动运行 `pip install` 来处理捆绑的 wheel 包（优先使用 `py.exe` 启动器，若不可用则回退至 `python -m pip`）。无需任何操作——成功安装后，`import chloros_sdk`在成功安装后可在您的Python环境中运行。如果系统上没有Python，安装程序会静默跳过此步骤，而GUI和CLI将继续正常工作。

#### Linux (.deb)

该 .deb 包将 wheel 文件放置在 `/usr/lib/chloros/sdk/` 路径下。`postinst` 会输出确切的命令——由于遵循 PEP 668 的发行版默认拒绝 pip 向全局路径写入，因此我们不会自动安装：

```bash
pip install --user /usr/lib/chloros/sdk/chloros_sdk-*.whl
```

对于隔离网络的 Jetson 部署，此过程完全离线进行——wheel 包已存在于磁盘上。

#### 公共 PyPI

对于仅支持 pip 的主机（未安装 Chloros 桌面包；远程后端或仅 DAQ 工作流）：

```bash
pip install chloros-sdk
```

PyPI 会在发布版本的安装程序构建时，PyPI 也会相应更新，因此发布的 wheel 包将与最新稳定版本保持一致。开发版本（例如 `1.1.4.dev1`）仅通过捆绑的安装程序 wheel 包提供。

#### 验证

```python
import chloros_sdk
print(chloros_sdk.__version__)
print("CAMERA_AVAILABLE =", chloros_sdk.CAMERA_AVAILABLE)
print("DAQ_AVAILABLE    =", chloros_sdk.DAQ_AVAILABLE)
print("PROJECT_AVAILABLE =", chloros_sdk.PROJECT_AVAILABLE)
```

> **需订阅Chloros+。** 所有 SDK 调用均需有效的 Chloros+ 登录凭据。请在每台机器上运行一次 `chloros-cli login user@example.com 'YourPassword'`；凭据将缓存于 `~/.chloros/` 中。

### 我需要桌面软件包吗？

对于大多数工作流而言，仅靠 pip 软件包是**不够的**。以下是每种 SDK 界面所需的内容：

| SDK 界面 | 是否需要桌面软件包？ | 原因 |
| --- | --- | --- |
| `ChlorosLocal`、`process_folder`、`process_lattice_capture` | **是** | 在 `/usr/lib/chloros/chloros-backend`（Linux）或 `C:\Program Files\MAPIR\Chloros\…` (Windows) 自动启动后端二进制文件。 |
| `connect_camera`、`connect_array`、`connect_daq_sensor`、`analyze_array_network`、 `list_*`、`discover_*` | **是**（本地）**/ 否**（远程）**| 通过后端运行纯HTTP客户端。本地后端 → 需要桌面软件包。远程后端 → `backend_url=`**通过隧道**（参见“远程后端模式”——随附的后端仅绑定回环地址）。 |
| `ChlorosProject` / `open_project` | **是** | 通过后端访问已保存的项目。 |
| 直接 LATTICE 类（`LatticeCamera`、`CameraPool`、`Calibration`、`DLS`、…） | **是** | 需要 Arena SDK 原生运行时（随桌面版软件包提供）。否则，`CAMERA_AVAILABLE` 在导入时即为 `False`。 |
| 直接 DAQ 类（`DAQUSensor`、`DAQMSensor`、`DAQESensor`、`SensorFleet`、`discover_all`） | **否** | 基于 pyserial/bleak/zeroconf 的纯 Python。 仅使用 pip 的环境可端到端驱动数据采集设备。 |

### 远程后端模式（仅使用 pip 的主机，通过隧道）

> **随附的后端无法通过局域网访问。** 生产版
> 仅绑定回环（包括两个回环系列），并硬性拒绝
> 唯一的非回环模式（`CHLOROS_CLOUD_MODE`），因此
> `backend_url="http://<lan-ip>:5000"` **无法与已安装的
> Chloros* 配合使用* ——该模式仅对源/开发
> 后端有效。若要驱动另一台机器上的后端，请自行转发其环回
> 端口，并将 SDK 指向隧道：

```bash
# on the pip-only host: forward local 5000 to the Chloros machine's loopback
ssh -N -L 5000:127.0.0.1:5000 user@chloros-host
```

```python
import chloros_sdk

BACKEND = "http://127.0.0.1:5000"   # the tunnel endpoint

chloros_sdk.connect_camera("213800234", backend_url=BACKEND)
chloros_sdk.connect_array(serials, backend_url=BACKEND)
chloros_sdk.connect_daq_sensor(eth_host="daq-e-1.local", backend_url=BACKEND)
```

无头主机/CI/机器人主机可以保留一台安装了完整桌面的机器作为“Chloros服务器”，其余所有机器则配置为 `pip install chloros-sdk` ——但它们之间的传输依赖于上述用户自行配置的隧道，而非直接的局域网URL连接。

> **已知限制 — `ChlorosLocal`不支持仅通过pip进行通信。** `ChlorosLocal(backend_url=BACKEND)` 目前会在其构造函数中，在探测 URL 之前就解析本地后端二进制文件，并且当未安装任何桌面软件包时（即使远程后端可达），也会抛出 `ChlorosBackendError` 异常（“未找到 Chloros 后端…”）。 只有上述智能连接界面（`connect_camera` / `connect_array` / `connect_daq_sensor`，以及 `analyze_array_network` 和 `list_*` / `discover_*` 辅助程序）可在仅安装 pip 的主机上运行。

### 仅数据采集（DAQ）工作流 （仅 pip 主机）

如果您仅需 DAQ 传感器，且不涉及 LATTICE 相机或图像处理，则 pip 软件包是自包含的：

```bash
pip install chloros-sdk
```

```python
from chloros_sdk import DAQUSensor, DAQMSensor, DAQESensor, discover_all

for d in discover_all(timeout=3.0):
    print(d.model, d.display, d.address)   # USB serials: d.extra.get("serial_number")

sensor = DAQUSensor(port="/dev/ttyUSB0")
sensor.connect()
sensor.start_streaming()
```

无需后端、无需 .deb 包，也无需登录 Chloros+ 即可直接硬件数据采集工作。

---

## 快速入门

```python
import chloros_sdk

# === Image processing ===
results = chloros_sdk.process_folder(
    "C:/DroneImages/Flight001",
    indices=["NDVI", "NDRE", "GNDVI"],
)

# === Live LATTICE single-cam ===
with chloros_sdk.connect_camera("213800234") as cam:
    cam.set_settings(exposure_time=10000, gain=0.0)
    cam.capture("output/")

# === Live LATTICE synchronized array (GUI smart-prep flow) ===
with chloros_sdk.connect_array(
        ["213800234", "214000533", "214701288", "214701292"]) as arr:
    arr.capture("output/", processing="reflectance")

# === Live DAQ spectral sensor ===
with chloros_sdk.connect_daq_sensor() as daq:    # smart-detect USB / BLE / ETH
    for frame in daq.latest(n=5):
        print(frame["spectrum"][:10])

# === Drive a saved project end-to-end ===
proj = chloros_sdk.open_project("/path/to/project")
proj.connect_all()
proj.arrays["main_rig"].capture("output/", processing="reflectance")
proj.disconnect_all()
```

---

## 顶级 API 索引

```python
import chloros_sdk

# === Image processing (full pipeline) ===
chloros_sdk.ChlorosLocal                          # class
chloros_sdk.process_folder(...)                   # one-shot helper
chloros_sdk.process_lattice_capture(...)          # LATTICE-friendly defaults
chloros_sdk.read_image_audit_tags(path)           # post-run audit

# === Live cameras (persistent backend pool) ===
chloros_sdk.connect_camera(serial, ...)           # → CameraSession
chloros_sdk.connect_array(serials, ...)           # → ArraySession (smart-prep)
chloros_sdk.attach_array(serials_or_id, ...)      # → ArraySession (attach without re-connecting)
chloros_sdk.list_cameras()
chloros_sdk.list_arrays()
chloros_sdk.discover_lattice_cameras()
chloros_sdk.analyze_array_network(...)            # network capability + recommendation
chloros_sdk.CaptureResult                         # list subclass returned by ArraySession.capture
chloros_sdk.RecorderHandle                        # handle for an array record()/burst() job

# === Live DAQ sensors (persistent backend pool) ===
chloros_sdk.connect_daq_sensor(...)               # → DAQSensorSession
chloros_sdk.discover_daq_sensors()                # scan USB/BLE/ETH (finds a DAQ-M MAC)
chloros_sdk.list_daq_sensors()

# === Project lifecycle ===
chloros_sdk.open_project(path)                    # → ChlorosProject
chloros_sdk.ChlorosProject                        # class
chloros_sdk.AlignmentSpec                         # dataclass
chloros_sdk.ArrayHandle, CameraHandle, SensorHandle

# === Direct-hardware (no-backend) classes (from lattice_sdk / daq_sdk) ===
chloros_sdk.LatticeCamera, CameraSettings, PRESETS, CameraPool
chloros_sdk.Calibration, CalibrationCoefficients, FilterModel, list_filters
chloros_sdk.DLS, NetworkDiagnostics
chloros_sdk.DAQUSensor, DAQMSensor, DAQESensor, SensorFleet, discover_all

# === Exceptions ===
chloros_sdk.ChlorosError                          # base
chloros_sdk.ChlorosBackendError
chloros_sdk.ChlorosLicenseError
chloros_sdk.ChlorosConnectionError
chloros_sdk.ChlorosProcessingError
chloros_sdk.ChlorosAuthenticationError
chloros_sdk.ChlorosConfigurationError
chloros_sdk.ChlorosConnectError                   # raised by smart-connect surface
chloros_sdk.LatticeError, CameraNotFoundError, ...  # from lattice_sdk

# === Availability flags ===
chloros_sdk.CAMERA_AVAILABLE     # True iff lattice_sdk imported cleanly
chloros_sdk.DAQ_AVAILABLE        # True iff daq_sdk imported cleanly
chloros_sdk.PROJECT_AVAILABLE    # True iff ChlorosProject deps available
```

---

## 图像处理 — `ChlorosLocal`

核心管道类。首次使用时启动后端，创建/配置项目，监控进度，并返回运行后摘要。

### 构造函数

```python
ChlorosLocal(
    api_url="http://127.0.0.1:5000",   # backend URL (also: backend_url=)
    auto_start_backend=True,            # spawn backend if not running
    backend_exe=None,                   # override backend binary path
    timeout=30,                         # request timeout seconds
    backend_startup_timeout=60,         # backend boot timeout
    processing_timeout=14400,           # hard cap on process() (4 h)
    processing_stuck_timeout=1800,      # no-progress threshold (30 min)
)
```

### 方法

| 方法 | 描述 |
| --- | --- |
| `create_project(project_name, camera=None)` | 创建新项目（可选使用相机模板，如 `"Survey3N_RGN"`）。 |
| `import_images(folder_path, recursive=False)` | 导入 RAW/TIF/JPG/DNG 图像 **以及 `.daq` 光传感器记录**。返回 `count`（图像）和 `scan_count`（记录）。仅当文件夹中既无图像也无记录时才发出警告。 |
| `export_light_sensor(daq=True, csv=True)` | 针对项目中的每条光传感器记录，将校准后的 `.daq` + `.csv` 写入 `<project>/Light Sensor/`。 参见 [光传感器记录](#light-sensor-recordings--calibrated-daq--csv)。 |
| `configure(debayer=..., vignette_correction=..., reflectance_calibration=..., indices=[...], export_format=..., ppk=..., daq_log_path=..., input_level=..., radiometric_output=..., array_alignment=..., array_alignment_crop=..., array_alignment_interpolation=..., custom_settings=None)` | 设置处理参数。 |
| `process(mode="parallel", wait=True, progress_callback=None, poll_interval=2.0)` | 运行处理管道。返回 `{"status": "complete", "async": False}`，以及当后端提供时返回的 `summary` 密钥——参见 [运行后摘要与提示](#post-run-summary--hints)。 |
| `get_config()` / `get_status()` / `status()` | 检查后端状态。 |
| `logout()` | 清除缓存的凭据。 |
| `shutdown_backend()` | 终止后端（若由SDK-started启动）。 |
| `discover_cameras()` | **通过该实例的后端** 发现 LATTICE 摄像头（`/api/camera/discover`）。返回一组字典列表（`serial`, `model`, `ip`, …) — 与GUI/CLI所见结构相同。若未发现任何摄像头或后端不可达，则返回空列表。 |
| `camera_capture(output_dir, format="tiff", **settings)` |**通过后端**捕获单帧（由该句柄自动由该句柄自动启动），使其获得与 GUI/ CLI 相同的预处理（默认 12 位，池资源复用，嵌入式校准元数据）。使用 `serial=` 或 `device_index=` 解析目标；传递 `exposure`/`gain`/`pixel_format`/`preset` 作为 `**settings`。返回旧版元数据字典 (`filepath`、 `width`、`height`、`pixel_format`、`exposure_time`、`gain`、 `timestamp`)。|
| `camera_stream(serial, *, fps=10.0, overlay=None, decode=True, connect_timeout=10.0, read_timeout=15.0)` | 从后端的 `/api/camera/<serial>/stream-annotated` 路由上，通过池化摄像头 ——通过后端的 `/api/camera/<serial>/stream-annotated` 路由 （服务器端绘制的斑马线/网格/十字线/直方图/峰值/光斑）。`decode=True` 返回 BGR 数组；`False` 返回原始JPEG字节。也可通过-project 访问，即 `ChlorosProject.stream(overlays=True)`。 |

用作上下文管理器以确保清理：

```python
with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("FieldA_2026-05-26", camera="Survey3N_RGN")
    cl.import_images("C:/DroneImages/Flight001")
    cl.configure(
        vignette_correction=True,
        reflectance_calibration=True,
        indices=["NDVI", "NDRE", "GNDVI"],
        export_format="TIFF (16-bit)",
    )
    results = cl.process(mode="parallel", wait=True)
print(results["summary"])
```

### 光传感器记录 — 已校准的 `.daq` + `.csv`

DAQ-U / DAQ-M / DAQ-E 可以在**不**使用其校准包的情况下进行记录。这正是
公开的 [`chloros_scripts`](https://github.com/mapircamera/chloros_scripts)
记录器（`record_daq.py`）的默认行为：它们写入原始传感器计数值，并在
文件中添加时间戳，以便 Chloros 能根据**序列号**获取该传感器的出厂校准值——优先查询本地缓存，
其次查询 MAPIR 云端——并在导入时应用该校准值。

Chloros 将结果写回为每条记录两个产品，位于
`<project>/Light Sensor/`下：

| 产品 | 内容说明 |
| --- | --- |
| `<name>_calibrated.daq` | 可重新处理的存档文件——与实时记录采用相同的架构，但现在声明了生成该文件的处理包。重新导入时**不会**再次进行校准。 |
| `<name>_calibrated.csv` | 基于传感器自身波长网格的谱辐照度（单位：W/m²/nm），每行对应一个读数，外加光度学列（总功率、明视/暗视勒克斯、PPFD及其蓝/绿/红分量、峰值波长）。 |
| `<name>_raw.daq` / `<name>_raw.csv` | **仅限无捆绑传感器（DAQ-A）。** 原始光谱传感器计数——*非*辐照度。详见下文。 |

`process()` 将其作为其中一个阶段执行此导出操作。它**不需要**图像：
单独飞行的一台光传感器本身就是一种完整的流程，此类项目按设计
默认不包含任何图像。

**DAQ-A 记录数据以原始计数值形式导出。** DAQ-A 系列早于按序列号划分的
捆绑系统，因此无需获取捆绑数据——它是在野外通过
反射率标靶进行校准，因此从未需要捆绑文件。这些记录导出时
使用 `_raw` 文件干名而非 `_calibrated`：采用不同的文件名而非文件内的标志， 因为该标识必须在作为纯文件名通过电子邮件转发时保持有效。
`.csv` 文件头中显示为 `raw spectral sensor counts (NOT irradiance)`，并提示这些
值仅在**同一**文件内具有可比性——这正是基于目标基于目标的校准
正是为此而使用这些值——而非跨传感器比较。与功率相关的光度学列（总功率、
明视/暗视勒克斯、PPFD）返回**NULL**，而非根据计数进行积分计算。

对于 DAQ-U / DAQ-M / DAQ-E（其数据包根本无法获取）仍会被**跳过**，
而非写入原始数据：这种情况下数据包确实存在，“重新连接并重新处理”是切实可行的建议。

旧版 **v1.01 / v1.02** 记录（由 DAQ-A-SD 写入）不包含每次读数的纪元，
仅包含文件的写入时间。图像↔下行光匹配器仍会拒绝这些数据——将
帧与写入时间进行匹配会导致隐性错误——但导出器会读取这些数据，且
CSV 会输出 `clock=daq_created_on`，因此该产品会注明当前使用的是哪个时钟。

```python
import chloros_sdk

with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("DAQ-U_2026-08-26")
    cl.import_images("C:/Flights/raw_daq")     # .daq only — no camera involved
    result = cl.export_light_sensor()          # or just cl.process()

for rec in result["exported"]:
    print(rec["csv"])
for rec in result["skipped"]:
    print("skipped", rec["source"], "--", rec["reason"])
```

若无法获取某条记录的校准包（离线状态，或传感器无
文件校准），则会在 `skipped` 下报告 **并附带原因**。该记录绝不会
作为包含原始计数数据的“已校准”文件写出——请连接互联网并
重新运行，导出操作即可完成。

### 进度回调

```python
def show_progress(percent, message):
    print(f"[{percent:3d}%] {message}")

with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("FieldA")
    cl.import_images("C:/DroneImages/Flight001")
    cl.configure(indices=["NDVI"])
    cl.process(progress_callback=show_progress, poll_interval=1.0)
```

### 运行后摘要与提示

完成后，`process()` 会获取 `GET /api/processing-summary` 并将正文作为 `result["summary"]` 附加。该获取操作仅尽最大努力，绝不会阻塞成功的 返回结果——若摘要不可用，`process()` 将回退到普通 `{"status": "complete", "async": False}` 结构。`summary["hints"]` 中的每一条记录——包含建议修复措施的完整句子， 例如某次运行为何输出为零——也会作为Python的`UserWarning`重新发出，因此即使您从未检查过该字典，输出为零的运行也能实现自诊断：

```python
result = cl.process()
for hint in result.get("summary", {}).get("hints", []):
    print("HINT:", hint)
# hints also arrive on the warnings channel:
#   python -W always::UserWarning your_script.py
```

`summary["totals"]` 是机器可读的部分：

| 键 | 统计内容 |
| --- | --- |
| `models` | 运行中的相机组。 |
| `images_in_groups` | 这些组中的源图像。 |
| `targets_found` | 检测到的反射率目标。 | |
| `images_calibrated` | 该运行校准的图像。 |
| `exported_files` | **该运行生成的图像产品文件。** |
| `daq_recordings_exported` / `daq_recordings_skipped` | 光传感器记录数据，特意单独计数——它们来自不同的阶段，且在完全没有图像的运行中也存在，因此若将其合并，会导致仅包含数据采集（DAQ）的运行看起来像是有导出了图像。 |

此外还有：`summary["output_dirs"]` （写入的每个目录），
`summary["light_sensor_export"]`、`summary["stopped"]`（当用户中断
运行时为真，因此部分计数不会被误判为产出不足的已完成运行），以及
`summary["groups"]`（按组细分）。

`exported_files`是由管道在**写入时**记录的，而非事后从
项目的图像对象中扫描得到的。并行和GPU策略会构建自己的图像
对象（在 GPU 路径的工作子进程中），因此旧的扫描机制会为
每次此类运行报告 `0 file(s) written`，随后发出零导出提示——即使在
运行一切正常的情况下也是如此。 如果您根据该数字编写脚本，现在一次正常的并行运行
将报告非零计数。

Light-sensor 跳过报告会显示读取器针对每个文件实际确定的原因——例如
不可读的架构、 缺失的捆绑包、写入错误——这些原因已**去重**，因此因同一原因被跳过的二十个文件
会被视为单一原因，而非二十次重复的记录。

> **当运行未生成任何图像时，不会触发 `process()`。**这是 SDK 与
> CLI 之间存在刻意差异：`chloros-cli process` 将“请求了产物，但未
> 写入任何产物”视为失败并返回非零状态，而 SDK 则正常返回，并通过
> `summary` / 提示。如果您的管道在空运行时应停止，请
> 自行检查——检查 `summary`（或统计项目文件夹下的文件数量），而不是依赖于
> 异常是否出现。常见原因包括输入文件夹未被识别为
> 捕获源，以及因不适用于当前摄像头而跳过的输出结果（例如，仅由 RGB
> 摄像头生成的辐射度数据）。

### 便捷函数

```python
# One-call process: project + import + configure + process
results = chloros_sdk.process_folder(
    folder_path="C:/DroneImages/Flight001",
    project_name="FieldA_2026-05-26",
    camera="Survey3N_RGN",
    indices=["NDVI", "NDRE", "GNDVI"],
    vignette_correction=True,
    reflectance_calibration=True,
    export_format="TIFF (16-bit)",
    mode="parallel",
    debayer="High Quality (Faster)",      # or "Texture Aware (Slow, Highest Quality)"
    ppk=False,
    recursive=False,
    processing_timeout=14400,
)

# LATTICE-friendly defaults (no panel-target detection, standard debayer)
results = chloros_sdk.process_lattice_capture(
    folder_path="C:/Captures/2026-05-13_Field",
    indices=["NDVI"],
)

# Audit which calibration sources were applied to a processed image
tags = chloros_sdk.read_image_audit_tags("output/Reflectance_Calibrated/x.tif")
print(tags["CalibrationSource"])   # 'per_serial' / 'legacy_lookup' / 'none'
print(tags["VignetteSource"])      # 'per_serial' / 'legacy_polynomial' / 'none'
```

### 支持的值

```python
# export_format
"TIFF (16-bit)"           # default, recommended
"TIFF (32-bit, Percent)"  # reflectance percentage as float32
"PNG (8-bit)"
"JPG (8-bit)"

# debayer
"High Quality (Faster)"               # standard, default
"Texture Aware (Slow, Highest Quality)"  # neural debayer, Chloros+ only
"Standard (Fast, Medium Quality)"      # alias used internally for LATTICE

# input_level (LATTICE only; Survey3 .raw ignores)
"auto"        # default — infers from each file's XMP ProcessingLevel tag
"raw"         # force-treat as raw Bayer
"debayered"   # force-treat as already-debayered BGR
"processed"   # force-treat as already-calibrated radiance

# array_alignment / array_alignment_crop (LATTICE arrays; None = keep saved setting)
True          # backend default — apply the module-to-module transform stamped
              # in each capture's Chloros:Alignment* XMP to every product
False         # export in native sensor geometry / skip the common-overlap crop

# array_alignment_interpolation (alignment warp resampling)
"bilinear"    # backend default
"nearest"     # preserves exact source DNs (no inter-pixel value mixing)
"cubic"
```

#### 辐射测量输出（LATTICE 多光谱处理流程）

`process` 处理流程的 LATTICE 多光谱（M3C/M3M）导出级别——`reflectance`（默认）、 `radiance`、`sensor-response` 或 `all`（每张图像的每个适用模式）——与项目中的 **“辐射度输出”** 处理设置相对应。 `configure()` 有一个专用的关键字：

```python
with chloros_sdk.ChlorosLocal() as cl:
    cl.create_project("Field_A")
    cl.import_images("C:/Captures/lattice_flight")
    cl.configure(
        radiometric_output="radiance",   # reflectance (default) / radiance / sensor-response / all
        export_format="TIFF (32-bit, Percent)",
    )
    cl.process()
```

高级“后门” ——通过 `custom_settings` 写入项目的 `"Radiometric output"` 键——仍然有效，但请注意这会替换整个设置块（参见下方的警告）：

```python
cl.configure(custom_settings={
    "Project Settings": {
        "Processing": {"Radiometric output": "radiance"},
        "Export": {"Calibrated image format": "TIFF (32-bit, Percent)"},
    }
})
```

`reflectance`（默认）将相机辐射度除以**时间戳匹配的 DAQ 下行辐射**，该值由图像旁记录的 `.daq` (DAQ-U/M/E)**或与图像一同发现的 DAQ-M 原生 `.csv`**；任何本地缺失的单相机或 DAQ 校准包将**在首次使用时自动从 AWS 获取**。CLI将此功能以按产品类型分类的开关形式呈现于`chloros-cli process`：`--radiance`/`--no-radiance`、`--reflectance`/`--no-reflectance`、`--debayered`、`--preview` 上以按类型设置的产品开关形式提供。

> `custom_settings` **将** 整个计算设置块（按设计，它会绕过 `configure()` 的其他关键字和验证）。使用时，请像上例那样包含您需要的每个 `Project Settings` 键。

---

## 适用于 LATTICE 摄像头的 Smart-Connect

针对实时硬件的持久化后端会话。使用与 GUI 相同的端点，因此 SDK / CLI / GUI 上的行为完全一致。

### 单台摄像头 — `CameraSession`

```python
import chloros_sdk

# Open by serial; reuses existing pool entry if one exists
with chloros_sdk.connect_camera("213800234") as cam:
    # cam is a CameraSession; supports context manager + manual disconnect
    cam.set_settings(
        exposure_time=10000,    # microseconds
        gain=0.0,               # dB
        pixel_format="BayerRG12",
        target_brightness=80,
        ae_damping=8.0,
    )
    cam.capture("output/", ext=".tiff")
```

#### `connect_camera()` 签名

```python
connect_camera(
    serial,
    *,
    preset=None,                       # "default" | "high_quality" | "high_speed" | "triggered"
    settings=None,                     # dict overlaid on the preset
    backend_url="http://127.0.0.1:5000",  # deliberately not 'localhost' (::1-first on Windows ≈ 2 s/request)
    timeout=60.0,
    auto_start_backend=True,           # spawn a local backend if none is running
) -> CameraSession
```

#### `CameraSession` 方法

| 方法 | 描述 |
| --- | --- |
| `read_nodes(names, enum_names=(), timeout=30.0)` | 读取 GenICam 节点；返回 `{nodes, errors, enums, device}`。 |
| `set_settings(**kwargs)` | 按友好名称写入节点 (`exposure_time`、`gain`、`pixel_format`、 `width`, `height`, `target_brightness`, `ae_damping`, `ae_upper_limit`, `trigger_mode`, `trigger_source`, …）。 |
| `capture(output_dir="output", ext=".tiff", jpeg_quality=95, processing=None, levels=None, force_daq=None, settings=None, timeout=None)` | 捕获 **单帧** 帧。返回一个包含帧元数据字典的单元素列表。（已移除连拍/多帧捕获功能——若需捕获系列帧，请在循环中调用 `capture()`。) |
| `disconnect()` | 从资源池中释放。若已连接到已打开的会话，则不执行任何操作。 |

`capture()` 导出控制 （与数组 + GUI 模型相同）：

- `processing` / `levels` — `processing="all"` 保存所有适用的导出类型；`levels=["raw","radiance"]` 仅保存上述类型（覆盖 `processing`）。若省略两者，则采用后端默认设置。
- `force_daq=True` — 即使在仅原始数据的抓取中，也会将分配的 DAQ/DLS 读数保存为 `.daq` 辅助文件，以便日后可将帧重新处理为反射率/折射率数据。若未关联任何 DAQ，则不执行任何操作。

### 同步数组 — `ArraySession`（Smart-Prep）

`connect_array` 是多摄像头设置的 **推荐入口点**。 它在后台运行完整的 GUI 智能预处理流程：

1. **网络分析** (`/api/camera/array/recommend`) — 查找能够满足 sim-emit 层级且不丢帧的最大帧大小。
2. **层级自动选择** — 若总线能够处理，则选择 `sim-capture-sim-emit`；否则选择 `sim-capture-ftd-stagger` 或 `slip-emit-and-capture`。
3. **自动缩减**— 当链路无法维持请求的分辨率时，会静默缩减帧大小/增加像素合并。**此安全措施不涵盖聚合超订阅**： 线路承载的摄像头数量过多无法通过缩小帧来解决 — 参见 [超额订阅](#over-subscription-the-per-cam-floor)。
4. 默认**启用 PTP** — 不同摄像头之间的时间戳可精确到微秒级。
5. **按摄像头自动选择像素格式** —— RGB 摄像头 → `BayerRG8`，多光谱摄像头 → `BayerRG12`。
6. **AE 初始化** — 快照记录每台摄像头的当前 AE 状态，以防止连接时在运行过程中重置曝光。
7. **GPIO 触发配置** — `connect_array` 使每台相机（`TriggerMode=On`、`TriggerSource=Line2`）处于就绪状态，以便主设备的脉冲通过 M8 电缆驱动从设备。 此步骤仅适用于阵列模式：若仅打开单个摄像头，则通过 `LatticeCamera` 命令使其进入自由运行模式。

```python
import chloros_sdk

# First serial is the MASTER (fires the trigger pulse); rest are slaves.
with chloros_sdk.connect_array(
        ["213800234", "214000533", "214701288", "214701292"]) as arr:
    print(arr.array_id, arr.sync_mode, arr.ptp_enabled)
    arr.capture("output/", processing="reflectance")
```

#### `connect_array()` 签名

```python
connect_array(
    serials,                              # list[str]; serials[0] = master
    *,
    line="Line2",                         # GPIO sync line: Line0 | Line2 | Line3
    target_fps=None,                      # master trigger fire rate (auto if None)
    force_tier=None,                      # override tier picker; see below
    wire_ceiling_mbps=None,               # host sustained wire budget, MB/s (auto if None)
    width=None,                           # explicit frame size; skips network analysis
    height=None,
    pixel_format=None,
    binning=None,
    recommend=True,                       # set False to skip the recommend step
    ptp_enable=True,                      # set False to disable PTP
    backend_url="http://127.0.0.1:5000",  # same IPv6-avoidance default as connect_camera
    timeout=180.0,
    auto_start_backend=True,              # spawn a local backend if none is running
) -> ArraySession
```

`force_tier` 值：
- `"sim-capture-sim-emit"` — 真正的同步（所有摄像头在同一时钟沿触发）。
- `"sim-capture-ftd-stagger"` — 灵活的时间域交错（凸轮在略有偏移的时间点发射，从而使数据包在传输线路上串行化）。
- `"slip-emit-and-capture"` — 按凸轮顺序捕获（无时间同步； 当无帧大小符合同步模式时唯一的选择）。

`wire_ceiling_mbps` 会覆盖 **主机的持续线速预算**（单位为 MB/s）—— 该单一
数值是整个数组配额的基准。保留默认值 `None` 以使用自动检测的
值。 当阵列报告 GVSP 损坏帧时，请降低该值：自动值是根据
网卡报告的链路速率推算得出的，但该速率往往高估了 USB 适配器、带宽较窄的 PCIe 通道以及
繁忙的共享结构时往往被高估——这种高估通常表现为帧损坏，而非
肉眼可见的链路变慢。该值会保存在项目阵列捕获块中，因此
重新打开项目或后续执行 `connect_array` 时，它会像其他阵列设置一样被恢复。
参见 [数组健康状况](#array-health--which-subsystem-is-losing-frames)。

#### 超额订阅（每台摄像头的下限）

Sim-emit 调速机制为每台摄像头分配一部分防碰撞安全带宽，下限为 **每台摄像机 8 MB/s**（`per_cam_floor_bps`）。一旦 `N × floor` 超过防冲突安全上限，阵列便会**超额分配带宽**—— 此时的故障模式是 GVSP 数据包丢失，而非帧率降低 —— 且 目前尚无针对帧大小的解决方案：**帧内字节的合并和ROI会减少每帧的字节数，而非速率控制机制中每秒传输的字节数**——而聚合检查比较的正是后者。在1 GbE主机上的实际全分辨率上限为：**6 台摄像机 @ 1500 MTU，其中 9 台使用巨型帧**（分析响应中的 `max_cams_collision_safe` 报告了您该链路的上限）。解决方法：减少摄像机数量、端到端启用巨型帧，或使用更快的网卡。

- `analyze_array_network()` 和 `/api/camera/array/connect` 响应中包含 `oversubscribed`、 `aggregate_demand_bps`、`collision_safe_ceiling_bps`、`max_cams_collision_safe` 和 `per_cam_floor_bps`。当 `oversubscribed` 为真时，X 为真时，该投影会**将 fps 字段清零**（`achievable_fps_max` / `fps_bright` / `fps_dark`），而非报告一个虽能工作但速度缓慢且容易误导的速率。
- `POST /api/camera/array/connect` 接受一个 `pin_resolution` 主体参数（**仅限 HTTP —— 不是 SDK 关键字参数**；`connect_array` 不暴露该参数）。 锁定会移除分桶逐步缩减的安全网，因此当设置了 `pin_resolution` 且连接超额订阅时，系统会**直接拒绝** 该连接，并返回一条列出所有补救措施的错误信息。若不进行锁定，连接将按逐步缩减流程继续，但会警告称缩减无法消除聚合值。
- 测试环境 应急方案：在后端环境中设置 `CHLOROS_ARRAY_ALLOW_OVERSUBSCRIBED=1`，将拒绝级别降级为高强度警告——您仍可建立连接并接受数据包丢失。

#### 数组健康状况 —— 哪个子系统正在丢失帧

`GET /api/camera/array/<array_id>/capability` 在已连接的阵列上携带一个活跃的 `health` 块，
该状态基于滚动 **10 秒** 滚动窗口内重新评估。它将帧丢失
细分为两种需要相反修复措施的原因，而不是一个未指明具体原因的“不完整”率：

| 字段 | 含义 | 涉及子系统 |
| --- | --- | --- |
| `gvsp_corrupt_rate_pct`（按串口） | 帧**已到达但结构损坏**— GVSP 数据包丢失。 |**网络**：带宽预算、速率调节、网卡接收环、 MTU |
| `never_arrived_rate_pct`（按序列号） | 帧**从未到达**——相机未触发，或未发送任何数据。 |**触发/同步**：M8电缆、`line=`、`TriggerMode` |
| `worst_gvsp_corrupt_pct` / `worst_never_arrived_pct` | 每种情况下的最差摄像头速率。 | — |
| `per_cam_rate_pct` | 每台摄像头的综合不完整 （两种原因合并计算）。 | — |
| `stable_for_seconds` | 每台摄像机保持在 0.01% 以下的时间长度的时间 | — |

与 `health` 一起，同一条记录还报告了整个分配所占用的数量：

| 字段 | 含义 |
| --- | --- |
| `wire_ceiling_mbps` | 主机当前生效的持续带宽配额，MB/s。 |
| `wire_ceiling_source` | 该数值来源的文字说明 — 例如 `USB-capped 200 MB/s (was theoretical 1062; …)` 或 `user override 120 MB/s (auto said 200)`。 |
| `wire_ceiling_is_user_set` | `true` 由 `wire_ceiling_mbps=` 设置时。 |
| `nic_is_usb` | `true` 用于 USB 以太网适配器。 |

此端点没有 SDK 封装函数——请直接读取：

```python
import requests, chloros_sdk

arr = chloros_sdk.attach_array(["213800234", "214000533"])
h = requests.get(
    f"http://127.0.0.1:5000/api/camera/array/{arr.array_id}/capability",
    timeout=10).json()

health = h.get("health", {})
print("wire ceiling:", h["wire_ceiling_mbps"], "MB/s", h["wire_ceiling_source"])
print("corrupt (network) :", health.get("worst_gvsp_corrupt_pct"), "%")
print("absent  (trigger) :", health.get("worst_never_arrived_pct"), "%")

if (health.get("worst_gvsp_corrupt_pct") or 0) > 1.0:
    # Network path. Reconnect with a lower budget -- NOT a lower target_fps.
    arr.disconnect()
    arr = chloros_sdk.connect_array(serials, wire_ceiling_mbps=120)
```

**读取结果：**非零的 `gvsp_corrupt_rate_pct` 且 `never_arrived_rate_pct` 为 0 时，
表示触发和电缆同步完全正常，100% 的丢包发生在网络路径上——将
`wire_ceiling_mbps` 并重新连接。反向模式则表明问题出在同步线缆或
触发线上。

> **`target_fps` 并非帧损坏的控制因素。** GevSCPD 时序在
> 连接时写入一次，因此降低触发率会改变占空比，而非
> 同时发射的突发速率。实测将需求削减5倍未见改善，而
> 将线速上限从240 MB/s降至200 MB/s后，同一测试平台的数据包损坏率从10.4%降至
> 0.00%。

> **TRI032S 固件不支持流中自动缩减。** 正在运行的数组无法
> 自行修复此问题；请断开并重新连接，以便连接时间选择器根据
> 新的上限重新规划。

**USB 以太网适配器的上限为 200 MB/s**，无论其
标称速率如何：将链路速率转换为持续传输速率的效率表源自
PCIe， 而 USB 网卡虽然会广播其以太网链路速率，但受限于
USB 总线及其驱动程序。该上限是绝对值，而非相对值——USB 1 GbE 适配器
可达到约 80 MB/s，且不受此限制影响。

#### `ArraySession` 方法

| 方法 | 描述 |
| --- | --- |
| `status(timeout=10.0)` | 实时 `{fps, ptp, frame_count, last_error, …}`。 |
| `capture(output_dir="output", format="tiff", processing="debayered", levels=None, aligned=None, render_index=None, force_daq=None, smart=False, timeout=300.0)` | 一个同步捕获组。返回一个 `CaptureResult`（帧字典列表 + `.skipped`）。导出控制如下。 |
| `capture(..., smart=True)` | **智能采集** — 等待所有摄像头上的 AE 值稳定后，再触发采集。 |
| `capture_fastest(output_dir="output", force_daq=True, render_index=True, timeout=120.0)` | 最快采集： 仅原始数据 + 指定的数据采集读数（+ 免费组合索引）。与 GUI 中的“最快采集”按钮功能一致。 |
| `capture_repeated(output_dir="output", count=None, duration_s=None, interval_s=0.0, on_capture=None, **capture_kwargs)` | 在一个有界循环中执行单次/连续/间隔采集。返回 `list[CaptureResult]`。**需要 `count` 以及/或 `duration_s`**，以便其终止 （SDK不支持Ctrl+C）。 |
| `record(output_dir="output", fps=10.0, duration_s=None, video=True, gif=False, timeout=30.0)` | 开始将实时组合索引视图录制为视频/GIF → `RecorderHandle`。每个数组仅有一个合成录制器。 |
| `burst(output_dir="output", duration_s=None, max_frames=None, index_config=None, serial_index_config=None, timeout=30.0)` | 开始高帧率原始拜耳连拍 → `RecorderHandle`。 使用 `build_video()` 进行离线重新处理。 |
| `build_video(burst_dir, products=None, fps=10.0, video=True, gif=False, save_tiffs=False, wait=True, poll_s=2.0, timeout=1800.0)` | 将已保存的原始连拍数据离线重新处理为校准后的视频。阻塞直至完成（`wait=True`），并 返回 `{outputs, errors, combined}`。 |
| `build_video_status(job_id, timeout=15.0)` | 轮询离线构建任务：`{running, result, error, burst_dir}`。 |
| `disconnect()` | 释放整个数组。 |

`capture()` 导出控制（与 GUI/CLI 使用的端点相同）：

- `processing` / `levels` — `processing="all"`（或 `levels=["raw","radiance",…]`）为每个 CAM 保存所有适用的导出类型； 而单个 `processing` 值仅保存该级别。
- `aligned=True` — 将每个成员的非原始导出数据变换为数组的 [对齐配置文件](#array-alignment)（共注册）； 原始数据保持未变换状态，但会在元数据中携带该变换信息。若数组无对齐配置文件，则回退为未对齐状态（并在结果的 `alignment` 中显示警告） 。
- `render_index=False` — 跳过按相机生成的植被指数叠加层；默认情况下会在配置的位置进行渲染。
- `force_daq=True` — 将分配的 DAQ/DLS 读数保存为 `.daq` 旁文件，即使所选层级无需该数据亦然。

**TIFF 压缩（HTTP -only 参数）：**`ArraySession.capture()` 不发送 `compression` 键，因此应用后端默认设置 — `POST /api/camera/array/capture` 读取 `compression` 主体参数，`"deflate"`（无损 zlib L1 + 水平预测器，每帧全分辨率约 4.1 MB）。`"none"` 以未压缩格式（约 6.3 MB/帧）写入，且**写入速度快约 5 倍** ——两者均为无损格式，导入时读取结果完全一致。SDK未为此提供任何关键字参数；解决方法是使用`chloros-cli lattice array-capture --compression none`或原始HTTP。DEFLATE还会持有Python的GIL，因此压缩写入无法在各摄像头写入线程间并行处理——以传感器速率持续捕获8路全分辨率 以传感器速率进行全分辨率捕获需要使用 `compression: "none"`。详情：[CLI 参考 → array-capture](cli-reference.md)。**按成员导出覆盖（仅限 HTTP）：**同一端点也接受 `exclude_serials`（列表 ——从已保存的集合中移除成员；数组仍作为单个同步组触发，被排除的成员将通过 `excluded` 返回），`serial_levels`（`{serial: [level tokens]}` 按-摄像机级覆盖），以及 `serial_index`（按 `{serial: bool}` 摄像机索引叠加覆盖）。这些是与 GUI 功能对等的主体参数，**目前还不是 SDK 关键字参数**； 映射中缺失的成员将回退到全数组范围的 `levels` / `render_index`。

##### 检查被跳过的 Cam — `CaptureResult.skipped`

`ArraySession.capture()` 返回一个 `CaptureResult`，它是一个 `list` 的子类：对其进行迭代、索引操作或 `len()` 处理——所有现有模式均能正常运行。新代码可检查 `.skipped` 属性，以查看哪些凸轮被排除以及原因。 最常见的情况是，当您请求 `processing="radiance"` 或 `"reflectance"` 时，混合滤光阵列中存在 RGB 相机——对于宽带传感器而言，每像素的辐射度（per-Bayer radiance）没有意义，因此后端会跳过这些相机，而不是生成无意义的数据。

```python
with chloros_sdk.connect_array(serials) as arr:
    result = arr.capture("output/", processing="reflectance")

    # Back-compat: iterate as a plain list
    for frame in result:
        print(frame["filepath"], frame["serial"])

    # New: see why N-1 cams were saved
    for skip in result.skipped:
        print(f"skipped SN:{skip['serial']} reason={skip['reason']}")
        # e.g. {'serial': '214701292', 'level': 'reflectance',
        #       'reason': 'reflectance-not-applicable-to-rgb-cam',
        #       'filter': 'RGB'}
```

原因标记遵循 `<level>-not-applicable-to-rgb-cam` 的模式（每个被跳过的级别一个条目，每个条目包含 `level`）。反射率相关的跳过情况包括：`reflectance-skipped-no-fresh-dls`（无新的下行光读数可用）、`reflectance-skipped-bound-daq-unavailable (…)`（无法连接到绑定的数据采集设备）以及`dls-uncalibrated-band-<nm>`——该波段大部分超出数据采集Q光传感器经辐射计量校准的范围（~374–974 nm）之外，因此基于DAQ的绝对反射率划分被拒绝，帧数据将直接降级为传感器响应。在已上市的SKU中，仅F988会触发此情况； 该相机支持的工作流程是反射率面板工作流。

`processing` 级别：

| 级别 | 输出 |
| --- | --- |
| `"raw"` | 单通道拜耳（单色相机：单波段）直接来自传感器。 |
| `"debayered"` *（SDK默认）* | 通过双线性去马赛克处理获得的3通道BGR（单色相机：1通道灰度）。 |
| `"radiance"` | 通过完整的辐射测量链获得的 float32 W/m²/sr/nm。仅限多光谱模式——RGB相机被跳过。 |
| `"reflectance"` | uint16 0..32768（Pix4D 兼容）；需与实时数据采集设备配对以获取绝对参考。仅限多光谱模式。 |
| `"display"` | 与 GUI 预览相匹配的完整处理链（根据相机配置文件进行色彩校正、白平衡和伽马校正）。 |
| `"all"` | 每台相机**每个适用级别一个文件**（与 GUI 的“捕获全部” /CLI默认设置）。返回的`CaptureResult`文件中，每个`(cam, level)`对应一个帧字典，每个字典中包含该级别；不适用的级别则出现在`.skipped`中。用于任何 反射率帧所用的数据采集读数，均作为`.daq`旁载数据保存。 |

> **注意 — 默认值与CLI不同。** `ArraySession.capture()` 的默认值为 `processing="debayered"`；`chloros-cli lattice array-capture` 命令的默认值为 `processing="all"`。请显式地将 `processing="all"`，以与CLI/GUI的多级保存功能保持一致。

### 捕获模式与记录器

阵列界面与GUI捕获面板功能一致：单帧/连续/间隔/最快快门模式，外加两个记录器（实时复合视频和原始连拍→离线重处理）。

```python
import time, chloros_sdk

with chloros_sdk.connect_array(serials) as arr:
    # Single (default) — one synced group
    arr.capture("out/", processing="reflectance")

    # Fastest — raw + .daq + combined index now, calibrate later
    arr.capture_fastest("flightline/")

    # Interval — one reflectance pass every 2 s, 5 passes (bounded so it ends)
    arr.capture_repeated("timelapse/", count=5, interval_s=2.0,
                         processing="reflectance",
                         on_capture=lambda i, r: print(f"pass {i}: {len(r)} frames"))

    # Combined-index video/GIF recorder (needs the combined live view streaming)
    with arr.record("monitoring/", fps=10, gif=True) as rec:
        time.sleep(30)
    print(rec.result["video_path"])

    # Raw-Bayer burst → offline reprocess into calibrated video(s)
    with arr.burst("capture/", duration_s=5) as b:
        pass
    out = arr.build_video(b.result["out_dir"], products=[
        {"kind": "per_cam", "level": "reflectance"},
        {"kind": "combined", "level": "index"}])
    print(out["outputs"])
```

- **`capture_repeated`**是 SDK 的连续/间隔循环。由于没有 `Ctrl+C` 可在脚本中中断该循环，因此您**必须** 传递 `count` 和/或 `duration_s`（达到其中任意一个即停止）。`interval_s` 从每次循环的开始处开始计时（与 GUI 一致）。 剩余的 kwargs 将直接传递给 `capture()`。
- **`record`** 属于 *监控级*：它按显示状态捕获实时组合-索引复合流，因此必须打开组合流才能接收帧数据。每个数组仅允许一个复合记录器（若已有记录器运行则触发异常）。
- **`burst` → `build_video`** 属于 *分析-级*：`burst` 以抓取循环的全速率写入原始帧 + 每帧清单 + 每个独特的 DLS 读数对应一个 `.daq`（位于 `<output>/bursts/<base>/` 下）（无链式处理， 不使用exiftool，无实时预览）。`build_video`将每帧与最近的`.daq`进行时间匹配，并重新运行导入管道中的辐射度/反射率/折射率处理链。 `products` 是一个 `{"kind": "per_cam"|"combined", "level": "radiance"|"reflectance"|"index"}` 列表（默认：组合指数）。`burst().stop()` 还会自动触发一次尽最大努力的组合指数构建，结果作为 `build_job` 返回。

#### `RecorderHandle`

由 `ArraySession.record()` 和 `ArraySession.burst()` 返回。可将其用作上下文管理器，在作用域退出时自动停止，或手动控制其运行。

| 成员 | 描述 |
| --- | --- |
| `job_id` | 后端作业 ID（字符串）。 |
| `kind` | `"composite"`（来自 `record`） 或 `"raw"`（来自 `burst`）. |
| `start_stats` | `start` 调用返回的字典。 |
| `result` | 运行期间的 `None`； 停止后返回的最终停止结果字典。 |
| `stats(timeout=10.0)` | 实时作业统计信息（写入帧数、实际帧率、耗时）。 |
| `stop(timeout=60.0)` | 停止记录器；返回并缓存最终结果。幂等 （第二次调用将返回缓存的结果）。 |

```python
rec = arr.burst("capture/")
# ... drive manually ...
print(rec.stats()["frames"])
result = rec.stop()
print(result["out_dir"], result.get("build_job"))
```

### 连接到已连接的数组 — `attach_array`

如果数组已处于运行状态 （由GUI打开，或先前SDK会话已调用`connect_array`），请使用`attach_array`获取其句柄，而非重新连接。在这种情况下，`connect_array`始终会报错“Camera  <sn>已处于数组中<id>”的错误，因为针对池中成员发送POST请求`/array/connect`不具备幂等性；`attach_array`会读取`/api/camera/array/list`，并通过array_id 或序列号进行匹配。

```python
import chloros_sdk

# By serials (matches if every serial is a member of one existing array)
arr = chloros_sdk.attach_array(
    ["213800234", "214000533", "214701288", "214701292"])

# By array_id (when you've already noted it down)
arr = chloros_sdk.attach_array("array-1779862544497")

# attach_array returns the same ArraySession as connect_array
arr.capture("output/", processing="reflectance")
```

模式：SDK 与桌面 GUI 共用租户的脚本应首先尝试 `attach_array`，若池中尚无数组，则回退至 `connect_array` 池中。

```python
import chloros_sdk

try:
    arr = chloros_sdk.attach_array(serials)
except chloros_sdk.ChlorosConnectError:
    arr = chloros_sdk.connect_array(serials)
```

> **重要提示 — 上下文管理器退出时确实会断开连接。**`ArraySession.disconnect()` 始终会 POST `/array/disconnect`；它不具备 `CameraSession` / `DAQSensorSession` 所具有的“已连接但未拥有”保护机制。如果您正在通过 GUI 进行多租户操作，且不希望在作用域退出时拆解数组，**请勿使用 `with` 代码块** —— 将句柄保存在普通变量中，并跳过显式的 `disconnect()`：
>
> ```python
> arr = chloros_sdk.attach_array(serials)
> arr.capture("output/", processing="reflectance")
> # … script ends; array stays up for the GUI
> ```

### 网络分析辅助工具

在打开数组之前非常有用——可预测您拟定的设置是否适用：

```python
result = chloros_sdk.analyze_array_network(
    master_serial="214701288",
    slave_serials=["213800234", "214000533", "214701162"],
    width=2048, height=1536,
    pixel_format="BayerRG12",
    binning=1,
)

if result["status"] == "ok":
    print("Use the requested settings.")
elif result["status"] == "auto_capped_fps":
    r = result["recommended"]
    print(f"Keep the resolution; cap the trigger rate at {r['recommended_target_fps']} fps")
elif result["status"] == "auto_shrunk":
    r = result["recommended"]
    print(f"Shrink to {r['out_width']}x{r['out_height']} binning={r['binning']}")
elif result["status"] == "needs_force_slip":
    print("Sim-sync impossible on this wire; force_tier='slip-emit-and-capture' required")
```

`status` 是 `ok` / `auto_capped_fps` / `auto_shrunk` / `needs_force_slip` 之一（否则为 `error`）。`auto_capped_fps` 表示所请求的分辨率仅在触发率受限的情况下才适合 RX 环——保留该分辨率并将 `target_fps=result["recommended"]["recommended_target_fps"]` 传递给 `connect_array`（参见[示例 6](#6-capability-probe-before-connecting-a-4-cam-array)）。

**如何解读投影** （与 GUI 的“阵列设置”面板模型相同）：

- **连拍（`frame_bytes_total`）按每台摄像头的实际像素格式进行汇总。**单色**M3M**摄像头会以 Mono12（2 B/px）格式流式传输 ，无论您传入的 `pixel_format` 参数为何，因此由三台单色摄像机组成的 4 摄像机全分辨率帧大小为**~25 MB** ，而非基于全部 8 位假设得出的 ~12.6 MB。后端会根据型号解析每台摄像头的格式。
- **通量 (`burst_fits_nic_ring`) 具有“排水感知”特性**，而非“整帧与环总线”的二分法：当主机从 RX 环中读取数据的速度快于摄像头向其写入数据的速度时，模拟-emit 机制适用于主机从 RX 环中读取数据的速度快于卡填充该环的情况。10G 主机 + 1 GbE 卡即使在突发数据超过环容量时仍能**允许**全分辨率数据通过；而 1 GbE 主机则会阻塞（`needs_force_slip` / `auto_shrunk`）。
- **`achievable_fps_max` 是一个保守的串行检索上限** — `max(readout+emit, N×emit)` 将每台摄像头的发送速率限制在 1 GbE 摄像机链路带宽内，且与曝光时间无关。例如，对于 4 台摄像头的全分辨率 12 位阵列，帧率约为 2.8 fps（与运行时测得的约 2.7–3.0）。完整模型：[CLI 参考 → 阵列帧率与连拍模型](cli-reference.md#array-fps--burst-model)。
- **超额订阅（`oversubscribed: true`）指 N × 每台摄像头的下限值超过了防冲突上限** ——帧率字段（`achievable_fps_max` / `fps_bright` / `fps_dark`）读值为 0， 且自动缩减/分桶无法解决此问题（这些机制仅降低每帧字节数，而非每秒定速传输的字节数）。解决方法包括减少摄像头数量、使用巨型帧或更换更快的网卡；`max_cams_collision_safe`报告了上限值（1 GbE 网络下，MTU 为 1500 时支持 6 台全分辨率摄像头，使用巨型帧时支持 9 台）。 该响应还包含 `aggregate_demand_bps`、`collision_safe_ceiling_bps` 和 `per_cam_floor_bps`（8 MB/s）。参见 [超额订阅](#over-subscription-the-per-cam-floor)。

### 发现与列表

```python
chloros_sdk.discover_lattice_cameras()   # list all cams visible to the backend
chloros_sdk.list_cameras()               # cams currently in the pool
chloros_sdk.list_arrays()                # active arrays in the pool
```

---

## 智能自动曝光（Smart-AE）/ 智能抓拍（Smart-Capture）

LATTICE 阵列一经连接便会在后台持续运行自动曝光（AE），但新定位的场景需要片刻时间才能收敛。**智能捕获** 提供了一套便捷的解决方案：它会轮询每台摄像机的曝光值，等待阵列在整个窗口内稳定后，再触发捕获。其功能等同于图形界面操作：桌面应用中的“智能”捕获按钮会调用相同的后端端点。

```python
import chloros_sdk

with chloros_sdk.connect_array([
        "213800234", "214000533", "214701288", "214701292"]) as arr:
    # Initial pose
    arr.capture("pose_a/", processing="reflectance", smart=True)
    input("Move the rig, then press Enter...")
    # New pose — smart-capture waits for AE to re-settle automatically
    arr.capture("pose_b/", processing="reflectance", smart=True)
```

通过 `ChlorosProject`（下一节）进行控制时，您将获得更多调节选项：

```python
proj.arrays["main_rig"].capture_smart(
    output_dir="out/",
    processing="reflectance",
    settle_timeout_s=5.0,           # max wait
    stability_window_s=1.5,         # exposure must hold steady this long
    exposure_tolerance_pct=5.0,     # %-spread allowed within the window
)
```

该 smart-AE 策略默认较为保守。对于要求严格的辐射测量工作，请收紧 `exposure_tolerance_pct` 参数；对于变化迅速的场景，若仅需“大致准确”的结果，则可放宽该参数。

---

## DAQ 传感器会话

用于光谱传感器的持久后端池（通过 USB 连接的 DAQ-U、通过 BLE 连接的 DAQ-M、通过以太网连接的 DAQ-E）。与相机功能相呼应：智能检测、池资源复用、幂等连接。

### 智能检测（零配置）

```python
import chloros_sdk

with chloros_sdk.connect_daq_sensor() as daq:
    print(daq.model, daq.transport, daq.address)
    for frame in daq.latest(n=10):
        spectrum = frame["spectrum"]   # list[float] (W/m²/nm if calibrated)
        is_sat = frame["is_saturated"]
        x, y, z = frame["x"], frame["y"], frame["z"]
        print(len(spectrum), is_sat)
```

优先级：以太网 → BLE → USB。传递任意一个显式提示即可锁定传输协议。

### 锁定传输协议

```python
# DAQ-U on a specific serial port
daq = chloros_sdk.connect_daq_sensor(transport="usb", port="COM3")

# DAQ-M over BLE by MAC (implies transport="ble")
daq = chloros_sdk.connect_daq_sensor(mac="AA:BB:CC:DD:EE:FF")

# DAQ-E over Ethernet by hostname (implies transport="eth")
daq = chloros_sdk.connect_daq_sensor(eth_host="daq-e-xxx.local")

# Tuning knobs
daq = chloros_sdk.connect_daq_sensor(
    port="COM3",
    integration_time=64,      # ms
    frame_avg=20,
    enable_ae=True,
    start_streaming=True,
)
```

### `DAQSensorSession` 方法

| 方法 | 描述 |
| --- | --- |
| `status(timeout=10.0)` | 池条目摘要（流式传输/记录状态、波长范围、校准 SHA、积分时间、 帧平均值、AE状态）。 |
| `latest(n=1, timeout=10.0)` | 返回最多 N 个最近的光谱帧。 |
| `stream_start()` / `stream_stop()` | 恢复 / 暂停流式传输 （句柄保持打开状态）。 |
| `record_start(output_dir=None, device_name=None)` | 开始录制 .daq 文件。返回文件路径。对于未配备 AWS 校准包的 DAQ-U/M 设备，此操作将被拒绝（DAQ-E 除外）。 |
| `record_stop()` | 停止录制。返回 `{path, rows}`。 |
| `disconnect()` | 从池中释放。对于已附加但非自有句柄，此操作无效。 |

> **电平校正配置文件（`cap_id`）并非SDK的控制参数。** `connect_daq_sensor()` / `DAQSensorSession` 不暴露任何 `cap_id`参数或`set_cap`方法。请通过CLI（`chloros-cli daq pool-connect --cap-id …` / `chloros-cli daq pool-set-cap …`）或后端`/api/daq` HTTP 路由（`/api/daq/connect` 和 `/api/daq/<id>/cap-id` 接受 `cap_id`）。

### 发现 — 查找用于连接的地址

`discover_daq_sensors()` 会扫描 USB / BLE / ETH 接口，以查找您*可能*能打开的传感器。它是 `discover_lattice_cameras()` 在 DAQ 方面的对应命令，也是获取 **DAQ-M 的 BLE MAC** 的唯一途径——DAQ-E 拥有主机名，DAQ-U 拥有 COM 端口，但 MAC 既不会显示在设备上，也不会被操作系统列出。

```python
for s in chloros_sdk.discover_daq_sensors():
    print(s["transport"], s["address"], s["model"], s["extra"])
# ble  C3:D8:85:E0:0A:19  DAQ-M  {'name': 'NSP32_SPECTRUM'}
# usb  COM3               None   {'manufacturer': 'Intel'}

# `address` is exactly what connect_daq_sensor wants:
for s in chloros_sdk.discover_daq_sensors(transports=["ble"]):
    if s["model"] == "DAQ-M":
        daq = chloros_sdk.connect_daq_sensor(mac=s["address"])
```

| 字段 | 描述 |
| --- | --- |
| `transport` | `usb` \| `ble` \| `eth`. |
| `address` | COM 端口 / BLE MAC / 主机名 — 作为 `port=` / `mac=` 传递给 `connect_daq_sensor` / `eth_host=`. |
| `display` | 人类可读标签。 |
| `model` | `DAQ-U` \| `DAQ-M` \| `DAQ-E`, 或 `None` 表示扫描无法识别的端口（若无探针，USB 串行适配器无法区分， 因此未知项会被显示出来而非隐藏）。 |
| `extra` | 按传输方式分类的详细信息（BLE 广播名称、USB 制造商、DAQ-E IP/固件/…）。空值将被省略。 |

| 参数 | 默认值 | 描述 |
| --- | --- | --- |
| `transports` | 全部三项 | 限制扫描的序列（或 CSV 字符串）。当您明确所需内容时值得传入——BLE 是速度较慢的部分。 |
| `scan_timeout` | 5 | 按传输方式划分的扫描窗口（单位：秒）；后端会将其限制在 1–20 之间。 |
| `timeout` | 60.0 | 整个调用中 `HTTP` 的上限 （与SDK中的其他部分一致）。 |
| `auto_start_backend` | `True` | 若未运行本地后端，则启动一个。对于远程 `backend_url` 绝不启动。 |

> **池中已打开的传感器不会显示。** 已连接的 BLE 外设将停止广播，且无法探测已打开的 COM 端口，因此发现功能仅列出*可连接的设备*。刚连接完设备后出现空结果是正常现象——若需获取已持有的设备，请使用 `list_daq_sensors()`。无法（未安装 bleak / zeroconf）会被跳过而非抛出异常，因此未启用蓝牙的设备仍能获取其 USB 和以太网的响应。

### 列表

```python
for s in chloros_sdk.list_daq_sensors():
    print(s["sensor_id"], s["model"], s["transport"], s["wavelength_range"])
```

### Co-多实例支持（带 GUI / CLI）

如果 GUI 已打开一个传感器，从 Python 调用 `connect_daq_sensor(port="COM3")` 将返回一个标记为 `already_connected=True` 的句柄。该会话的 `disconnect()` 此时为无操作，因此您的SDK脚本在示波器退出时不会从GUI中强行移除该传感器。

### 直接硬件类（无后端）

`daq_sdk`由-导出，因此您也可以在进程内端到端地驱动传感器，而无需后端：

> **可用性：**`daq_sdk`随Chloros桌面版安装程序提供，**不**包含在PyPI包中——`pip install chloros-sdk`为您提供`lattice_sdk`，但保留了`chloros_sdk.DAQ_AVAILABLE == False`。使用这些类之前请确认该标志；在仅安装 pip 的主机驱动器上，请改用 [`connect_daq_sensor()`](#daq-sensor-sessions) 进行传感器连接，该方式无需本地传输库。

```python
from chloros_sdk import DAQUSensor, DAQMSensor, DAQESensor, discover_all

# Discovery
for d in discover_all(timeout=3.0):
    print(d.model, d.display, d.address)   # USB serials: d.extra.get("serial_number")

# Direct DAQ-U
sensor = DAQUSensor(port="COM3")
sensor.connect()
sensor.start_streaming()
# ... use sensor.add_spectrum_callback(...) ...
sensor.stop()
```

若当您希望与图形用户界面共享所有权时，请优先使用智能连接路径（`connect_daq_sensor`）；对于独占传感器所有权的无头脚本，请使用直接类。

---

## 项目自动化 — `ChlorosProject`

已保存的Chloros项目是一个包含`cameras.json` + `sensors.json` + `project.json`的文件夹。 `open_project` 加载清单文件，而 `connect_all` 则会使用保存的设置将所有已保存的设备连接到网络——其硬件状态与图形用户界面（GUI）生成的状态完全一致。

### 最小示例

```python
import chloros_sdk

proj = chloros_sdk.open_project("/home/user/Chloros Projects/Field_A")
report = proj.connect_all(verbose=True)
print(report)  # {'cameras': {...}, 'arrays': {...}, 'sensors': {...}}

# Cameras and arrays are addressable by name OR serial / array_id
cam = proj.cameras["FrontLeft"]
cam.capture("./out", format="tiff", processing="reflectance")

arr = proj.arrays["main_rig"]
arr.capture("./out", format="tiff", processing="reflectance")

# Read a DAQ
spectrum = proj.sensors["Sky"].read()

# Trigger every device simultaneously
proj.capture_all("./out")

proj.disconnect_all()
```

或作为上下文管理器：

```python
with chloros_sdk.open_project("/path/to/proj") as proj:
    proj.connect_all()
    proj.arrays["main_rig"].capture("./out", processing="reflectance")
```

### `ChlorosProject` 方法

| 方法 | 描述 |
| --- | --- |
| `connect_all(cameras=True, arrays=True, sensors=True, verbose=False, align=None)` | 发现并连接所有已保存的设备。返回按类划分的连接报告。若存在正在监听 `127.0.0.1:5000` 的后端，则使用该后端；否则会无提示地回退到直接 （无需后端）`lattice_sdk` 设备控制模式——该方法绝不会创建后端。 |
| `disconnect_all()` | 关闭所有连接。 |
| `capture_all(output_dir=".")` | 从每台摄像头获取一帧图像，并从每个传感器获取阵列及光谱。 |
| `stream(camera, overlays=False, fps=10.0)` | 生成器，从指定名称的摄像头 （或阵列）生成 BGR `numpy` 帧。`overlays=False` 是直接的 `lattice_sdk` 抓取循环（阵列生成 `{serial: frame}` 字典）。`overlays=True` 通过 `ChlorosLocal.camera_stream()` → 后端的 `/api/camera/<serial>/stream-annotated` MJPEG 数据流进行路由，并将摄像头的已保存 `ui.overlay` 块作为查询参数传递。需要后端模式和一台 **独立摄像头**：直接模式的摄像头会触发 `RuntimeError` （后端无法获取该进程拥有的摄像头），而数组会触发 `NotImplementedError`（按摄像头合成叠加层——按名称流式传输成员）。单次操作等效形式：`CameraHandle.capture(annotated=True)`。 |
| `align_arrays(align=True, verbose=False)` | 对当前连接的每个数组运行对齐操作。 |
| `process(mode="parallel", wait=True, progress_callback=None, poll_interval=2.0)` | 对项目的图像运行校准/索引处理流程（包含 `ChlorosLocal.process`；这四个是**唯一**被接受的关键字参数 ——`indices=`等会引发`TypeError`；通过`ChlorosLocal.configure()`设置索引）。懒加载方式构建一个`ChlorosLocal()`，该对象会自动启动 后端。 |

属性：
- `proj.cameras` — `Dict[str, CameraHandle]` 以名称和序列号为键。
- `proj.arrays` — 以名称和 array_id 为键的 `Dict[str, ArrayHandle]`。
- `proj.sensors` — `Dict[str, SensorHandle]`，以名称和 slot_id 为键。
- `proj.config` — `project.json["config"]` 字典。

### `CameraHandle`

```python
cam = proj.cameras["FrontLeft"]

# Save a frame to disk (processing-aware)
filepath = cam.capture(
    output_dir="./out",
    format="tiff",
    processing="radiance",           # see the level table below
    apply_calibration=True,          # DSNU + flat + 3x3 unmix + NIST
    apply_white_balance=True,        # DLS-aware WB
    apply_index=False,
    index_expression=None,
)

# In-memory grab (numpy array)
frame = cam.grab(processing="debayered")
frame, header = cam.grab(processing="radiance", with_metadata=True)

# Frame iterator (generator)
for arr in cam.frame_stream(processing="debayered", fps=5, count=100):
    my_analysis(arr)
```

**处理级别。** `capture()`、`grab()` 和 `frame_stream()` 均采用相同的 `processing`
令牌，且该链是累积的 ——每个级别都会执行其上方的所有内容：

| 级别 | 输出 | 备注 |
| --- | --- | --- |
| `raw` | 1 通道拜耳，传感器原生 | 不进行去马赛克。此级别不支持叠加。 |
| `debayered` | 3 通道 BGR (**默认**) | 双线性去马赛克。唯一无需后端模式即可运行的级别。 |
| `radiance` | float32, W/m²/sr/nm | 完整的辐射测量链：去马赛克 + 3×3 解混（多光谱） + DSNU + 平场校正 + NIST 标度，已剔除曝光量 × 增益，因此数值为绝对值。 |
| `reflectance` | uint16, 32768 = 1.0 | 辐射度除以降射辐照度（ρ = π·L/E）。需要 DLS/DAQ 读数——参见下文注释。 |
| `display` | 8 位 sRGB 风格 | GUI等效渲染：通过相机的活动色彩配置文件实现 CCM + 白平衡 + 伽马校正。 |

除 `debayered` 以外的任何值都需要后端模式；直接模式相机将触发
`NotImplementedError`。`reflectance` 需要一个可用的下行读数——帧终点会自动将
汇总的 DAQ 拉入相机的 DLS 插槽，但如果未绑定 DAQ，链路会拒绝
反射率输出，并会在返回的元数据中如实标记降级情况，而非默默
返回质量较低的处理结果。

> **反射率DN标度——请勿硬编码。** LATTICE反射率使用`32768` = ρ 1.0，并标记
> XMP `Chloros:PixelScale=32768`；Survey3反射率使用`65535` = ρ 1.0，且不携带
> `Chloros:*` 标签。读取该标签并以此除以。该值定义在 uint16 域中，因此对于所有进行缩放的格式（16 位 TIFF、8位 PNG /JPG、32 位百分比）——请先将
> 存储的数据类型归一化回 uint16（从 8 位乘以 257，从浮点数乘以 65535）。唯一例外：
> 源为 8 位的捕获数据若以 8 位 TIFF 格式写入，则会被*裁剪*，而非重新缩放，因此没有比例值描述
> 它——Chloros在这种情况下会完全省略 `PixelScale` 和 MicaSense 元组。将 LATTICE 反射率文件中缺失的
> 标签视为“无有效比例”，而非默认值。

> **EXIF 信息将随导出文件一并传递。** `process()` 将源捕获文件的 GPS 数据块
> **及其 ExifIFD** 复制到每个产品中，因此导出文件会包含 `FocalLength`、`FNumber`、
> `ExposureTime`、`ISO`、`DateTimeOriginal` 和 `CameraSerialNumber` 以及
> 地理参考信息。`FocalLength` 是 Pix4D 用于计算地面采样距离的依据——如果没有它，
> 重建结果将退化为严重错误的比例尺（一个实测案例中，411 米的场地
> 变成了47.8公里）。该副本特意未使用`-all:all`：IFD0的结构标签会破坏
> LATTICE 输出，而 `ExifImageWidth`/`Height` 则被排除，因为它们描述的是源
> 捕获过程，而非导出的栅格。

捕获阶段子标志（适用于辐射测量级别 — `radiance`、`reflectance`、`display`）：

| 标志 | 默认值 | 含义 |
| --- | --- | --- |
| `apply_calibration` | `True` | DSNU + 平场校正 + 3x3 解混 + NIST 辐射计量标度。 |
| `apply_white_balance` | `True` | 白平衡查找表（WB LUT）。当数据采集卡（DAQ）与相机绑定时，支持DLS。 |
| `apply_index` | `False` | 植被指数评估。 |
| `index_expression` | `None` | 覆盖公式。非空值 → 自动启用指数。 |
| `annotated` | `False` | 叠加 GUI 装饰（条纹/网格/峰值）。不适用于 `raw`。 |

### `ArrayHandle`

```python
arr = proj.arrays["main_rig"]

# Single synced capture group
files = arr.capture("./out", format="tiff", processing="reflectance")
# → {"213800234": "/path/to/x.tif", "214000533": "/path/to/y.tif", ...}

# Multi-level: each serial's value becomes an ordered LIST, not a str
files = arr.capture("./out", processing="all")
# → {"213800234": ["/raw.tif", "/debayered.tif", ...], "combined": "/idx.tif"}

# Smart capture (wait for AE to settle)
result = arr.capture_smart(
    "./out", processing="reflectance",
    settle_timeout_s=5.0,
    stability_window_s=1.5,
    exposure_tolerance_pct=5.0,
)
print(result["frames"], result["settle"])

# In-memory grab: {serial: numpy array}
frames = arr.grab(processing="debayered")
frames = arr.grab(processing="radiance", with_metadata=True)

# Stream-to-disk loop
arr.stream(count=60, output_dir="./stream", fps=5, processing="raw")

# Frame-iterator (tolerates per-cam drops; great for downstream analysis pipelines)
for frames in arr.frame_stream(processing="radiance", fps=5, count=100):
    if "213800234" in frames:
        my_analysis_pipeline(frames["213800234"])

# Preview iterator (live MJPEG-equivalent; tolerates partial cycles)
counts = arr.preview_stream("./preview", fps=3.0, duration=30.0)
print(counts)  # frames written per serial
```

> **返回类型为 `CapturePathMap`，而非 `Dict[str, str]`。**
> `chloros_sdk.CapturePathMap` 即 `Dict[str, Union[str, List[str]]]`：单级
> `processing` 为每个序列分配一条路径，而多级结构（`"all"`，或
> 显式的 `levels` 列表）则为该
> 相机保存的所有产品的 **有序列表**，其中包含为该
> 摄像机保存的所有产品。如果存在实时流式传输的组合合成画面，它将通过额外的
> `"combined"` 键传入，而非通过序列号。假设使用 `str` 的代码在
> 列表 形式时会出错，且没有任何类型检查器会提出异议——在列表形式发布后的一段时间内，注释中曾写着 `Dict[str, str]`
>，这就是该别名存在的原因。若需要平铺形式，请规范为
>：
>
> ```python
> paths = arr.capture(processing="all")
> flat = [p for v in paths.values()
>         for p in (v if isinstance(v, list) else [v])]
> ```

### 数组对齐

`ArrayHandle` 公开了完整的对齐表面。默认情况下，配置文件仅在会话中有效——若要持久化，请显式调用 `export_alignment()`。

```python
from chloros_sdk import AlignmentSpec

arr = proj.arrays["main_rig"]

# Defaults: ORB / affine / one synced snapshot — same as the GUI's auto-cal
result = arr.calibrate_alignment()
print(result["profile"]["rms_residual_px"])

# Custom spec for tough scenes (low-contrast canopy)
spec = AlignmentSpec(
    method="feature_orb",         # feature_orb / feature_akaze / phase_correlation / checkerboard / manual
    model="rigid",                # translation / rigid / affine / homography
    num_frames=5,
    max_features=8000,
    ratio_threshold=0.7,
    ransac_threshold_px=2.0,
    min_matches=30,
    max_reproj_err_px=2.0,
)
arr.calibrate_alignment(spec)

# Or tweak one knob at a time
arr.calibrate_alignment(num_frames=3, model="affine")

# Inspect / manipulate
status = arr.alignment_status()
arr.tweak_alignment("214701292", dx=2.5, dy=-1.0, rotation_deg=0.0, scale=1.0)
arr.export_alignment("/tmp/main_rig_alignment.json")
arr.import_alignment("/tmp/main_rig_alignment.json", validate=True)
arr.clear_alignment()
```

#### 连接时对齐

`connect_all(align=...)` 可在连接时自动对齐每个数组：

```python
# Align every array with defaults
proj.connect_all(align=True)

# Per-array control
proj.connect_all(align={
    "main_rig": AlignmentSpec(num_frames=5, model="affine"),
    "side_rig": True,             # use defaults
    "verify_rig": False,          # skip
})
```

若未指定，则回退到 `project.json["config"]["auto_align_on_connect"]`。

### `SensorHandle`

```python
spectrum = proj.sensors["Sky"].read()
# (spectrum_list, is_saturated, integration_time, x, y, z) — matches the
# daq_sdk add_spectrum_callback signature.
```

---

## 直接硬件（无后端）

若希望完全不依赖后端（CI、无头机器人、嵌入式系统），请直接导入 `lattice_sdk` 和 `daq_sdk` ——这两个模块均由 `chloros_sdk` 重新导出。请注意 `CAMERA_AVAILABLE` / `DAQ_AVAILABLE` 的限制条件：`lattice_sdk` 包含在 PyPI 包中（但需要安装 Arena SDK 运行时）， 而 `daq_sdk` 仅随桌面安装版提供。

```python
from chloros_sdk import (
    # cameras
    LatticeCamera, CameraSettings, PRESETS, CameraPool,
    Calibration, CalibrationCoefficients, FilterModel, list_filters,
    DLS, NetworkDiagnostics, gpu_info, gpu_available,
    # discovery
    discover_cameras, discover_cameras_via_backend,
    # exceptions
    LatticeError, CameraNotFoundError, StreamError, CaptureError,
    CalibrationError, NetworkError, DLSError,
)

# Find a camera and capture in one go
cams = discover_cameras(timeout_ms=3000)
print(cams)

settings = PRESETS["high_quality"]
with LatticeCamera(serial="213800234", settings=settings) as cam:
    result = cam.capture(output_dir="./out", format="tiff")
    print(result.filepath, result.width, result.height)
```

##### 预设与触发机制

四个预设中有三个采用**自由运行**模式：相机持续曝光，且
`capture()`会返回下一帧。`triggered`是例外——它会在第2行将
相机，等待第 2 行出现硬件沿，因此在检测到沿之前不会进行任何拍摄。

| 预设 | 触发方式 | 适用场景 |
| --- | --- | --- |
| `default` | 自由运行 | 通用 |
| `high_speed` | 自由运行 | 8 位， 60 fps上限，短曝光 |
| `high_quality` | 自由运行 | 12 位，无帧率上限 — 静态照片的常规选择 |
| `triggered` | **待机，第 2 线** | 相机通过 M8 同步线连接，并由其他设备触发 |

如果您选择 `triggered`（或自行设置为 `trigger_mode="On"`），且
第 2 线未被驱动，则每个 `capture()` 都会超时——这是正确的，因为你要求
相机等待。 SDK在发生这种情况时会进行说明；请参阅
[捕获期间的 SC_ERR_TIMEOUT](#direct-hardware-backend-free)。

> **注意 — 连接时的“GVSP probe”/ `SC_ERR_TIMEOUT -1011` 消息并非错误。**&gt; 连接时，SDK会尝试协商**巨帧**（9000字节的GVSP数据包）以获得更高的吞吐量。在直接点对点的网卡链路上 （例如链路本地 `169.254.x.x` 地址）上，网络通常无法传输巨帧，因此此探测会超时并记录如下日志：
>
> ```
> [Network] GVSP probe: unexpected error (TimeoutError: ... SC_ERR_TIMEOUT -1011)
> [Network] GVSP probe at 9000 did not deliver a complete buffer; reverting to ICMP-chosen size
> [Network] GVSP packet size: 1500 bytes (standard)
> ```
>
> 这是**设计的备用方案**：SDK会自动恢复为标准的1500字节数据包，摄像机仍能正常连接（随后的`[chunk-enable …]`行属于正常的连接序列）。数据捕获功能依然有效。
>
> 您可以跳过此探测，但**它不仅仅是一个日志静音器——它会关闭巨帧功能。** 无论您的网络性能如何，摄像机对“禁止分片”的ping响应大小上限始终为1500字节，因此仅靠ping测试永远无法检测到巨帧；只有这个探测命令才能做到。 禁用它后，无论在何种网络环境下，摄像头都将永远发送标准的 1500 字节数据包：
>
> ```bash
> CHLOROS_GVSP_PROBE_FALLBACK=0   # gives up jumbo — see the warning it prints
> ```
>
> 仅在您*确知*网络无法传输巨型帧的情况下才值得启用，这样每台摄像头可节省约一秒的连接时间。 由于这是一种实质性的权衡而非表面上的调整，现在当你使用SDK时，系统会明确提示：
>
> ```
> [Network] ⚠️ GVSP probe disabled (CHLOROS_GVSP_PROBE_FALLBACK=0) — staying at
> 1500 bytes, jumbo NOT tested. … if this network does carry it, you are giving
> up ~1.45x wire ceiling. Unset the variable to test for jumbo.
> ```
>
> **除非有特殊原因，否则请勿更改。** 若保持启用状态，每次连接时系统都会重新检测实际网络环境：当连接到支持巨包的交换机，下次连接时系统会自动识别巨包，无需任何配置，也无需重启。
>
> 若您*希望*获得巨包吞吐量，请启用端到端巨包（网卡 MTU 设为 9000 且交换机支持转发），或在确认链路支持的情况下通过 `CHLOROS_GVSP_PACKET_SIZE_FORCE=9000` 进行固定——尽管在确认链路支持巨包时，建议使用按命令设置的 `CHLOROS_GVSP_PACKET_SIZE_FORCE=9000 python …` 而非永久设置，因为固定大小会跳过探测并停止适应前端网络。**所有** 路径中的设备都必须支持巨帧传输——包括任何 PoE 分路器或注入器，这通常是导致原本支持巨帧的配置无法传输巨帧的原因。

> **在 `capture()` / `grab*()` 期间出现的问题属于另一类——那才是真正的错误。**&gt; 上述说明仅涉及由**连接时间探针**记录的 `-1011` 错误。若该错误由**捕获** 操作触发，则表示摄像头已成功连接，但未发送任何图像：
>
> ```
> File ".../lattice_sdk/camera.py", line ..., in grab_frame_with_metadata
>   buffer = self._get_buffer(timeout)
> lattice_sdk.exceptions.CaptureError: Capture failed: ... SC_ERR_TIMEOUT -1011
> ```
>
> 关键线索在于：摄像机的 *控制* 通道状态正常——发现功能正常，设置和 `[chunk-enable …]` 写入操作均成功——但 *每个* 帧都会超时。
>
> **通常的原因是相机已设置为硬件触发模式。** 对于 `trigger_mode="On"` 和 `trigger_source="Line2"`，在 M8 同步线缆上接收到电平变化之前，摄像机不会发送任何数据。如果该线路未连接任何线缆，每次抓取操作都将无限期等待。摄像机并未损坏，网络也 没有问题——它完全按照指令在运行。
>
> `CameraSettings()` 以及 `default` / `high_speed` / `high_quality` 预设支持自由运行， 而在启用状态下因超时而失败的抓取操作会显示具体原因，而非仅显示一个孤立的 `-1011`。`PRESETS["triggered"]` 会启用 Line2，这是按设计预期的。
>
> 要强制任何摄像机进入自由运行：
>
> ```python
> settings = PRESETS["high_quality"]
> settings.trigger_mode = "Off"        # free-run; don't wait for an M8 edge
> ```
>
> 如果使用 `trigger_mode="Off"` 时仍超时，说明摄像头确实没有传输数据——请将日志和 `ip link show` 发送给我们。

#### 色彩配置文件（RGB 实时预览）—— `set_color_profile`

`LatticeCamera.set_color_profile(profile, custom_cct_k=None)` 用于选择 RGB 相机上 **实时预览** 的显示色彩配置文件（多光谱相机忽略此设置）：

| 配置文件 | 含义 |
| --- | --- |
| `raw` | 完全绕过辐射度链。 |
| `linear` | DSNU + 平滑校正 + 白平衡，无CCM，无伽马校正。 |
| `natural` | 线性 + 实测CCM + sRGB伽马，仅采用基础处理（色度平滑 + 高光去饱和）——这是最逼真的默认设置。 |
| `enhanced` | `natural` 加上完整的 hub-parity 后期处理 （去色散、鲜活度、CLAHE局部对比度）。画面更丰富，但**每帧处理成本约为两倍**，因此实时帧率较低。 |
| `custom_temp` | `natural`，但白平衡固定为 `custom_cct_k` 开尔文值（忽略 DLS；后端端）。 |

该配置文件是一个**仅限实时预览**的速度/外观调节旋钮：保存的截图始终能获得完整丰富的最终效果，无论选择何种配置文件，因此选择 `natural` 以换取帧时间并不会降低存储到磁盘上的图像质量。未知配置文件会提升 `ValueError`；当可访问 chloros 后端 可达时，该更改也会通过POST请求发送至后端，以便下一帧预览反映该变化（使用SDK且未连接后端的用户仍会收到设置变更）。

```python
with LatticeCamera(serial="214701292") as cam:   # RGB cam
    cam.set_color_profile("enhanced")            # richer look, lower LIVE fps
    cam.set_color_profile("custom_temp", custom_cct_k=5600)
```

#### 单色 (M3M) 相机与 `Calibration`

单色 **M3M** 相机 (`M3M-<lens>-F<wavelength>`) 属于单波段：仅有一个灰度平面， 无拜耳马赛克，无 3×3 光谱串扰矩阵。`Calibration` 可识别该相机并暴露一个 `is_mono` 标志。 反射率仍作为每波段的辐射测量图适用（解混矩阵为单位矩阵），但对单台相机进行多波段运算会产生结果而非返回无意义数据：

```python
from chloros_sdk import Calibration, CalibrationError

calib = Calibration("M3M-L87-F685")
print(calib.is_mono)        # True  (False for any M3C / RGN Bayer cam)
print(calib.filter_type)    # 'mono'  (sentinel; not a real crosstalk key)

# NDVI needs two bands (Red + NIR); one mono band can't supply both.
try:
    calib.compute_ndvi(reflectance_frame)
except CalibrationError as e:
    print(e)   # "...single-band mono (M3M) camera. Combine multiple..."
```

若要利用单色硬件构建植被指数，请将不同波长的多台 M3M 相机组合成对齐的多波段堆栈（参见 [阵列对齐](#array-alignment)），并在该堆栈上计算指数，而非仅针对单台相机计算。

DAQ 直接模式：

```python
from chloros_sdk import (
    DAQUSensor, DAQMSensor, DAQESensor,
    SensorFleet, discover_all, DiscoveredSensor,
    apply_sensor_settings, SensorSettings,
)

for d in discover_all(timeout=3.0):
    print(d)

sensor = DAQUSensor(port="COM3")
sensor.connect()
apply_sensor_settings(sensor, settings={"integration_time_ms": 64, "frame_avg": 20})
sensor.start_streaming()
# ... sensor.add_spectrum_callback(your_callback) ...
sensor.stop()
```

> **`apply_sensor_settings` 接受的键**— 确切地是 `integration_time_ms`、`frame_avg`、`ae_enabled`、`sunshine_diffuser_installed`（DAQ-E；已弃用，建议改用 `cap_id`）、`filter_model`（DAQ-M）、 以及 `cap_id`（所有 DAQ 类型；`None`/`""`/`"none"` = 裸传感器，无电容校正）。未知 键值将被**静默忽略** ——例如，`{"integration_time": 64}` 不会产生任何效果（必须是 `integration_time_ms`）。返回 `{"applied": [...], "errors": {...}}` 且绝不抛出异常。

`chloros_sdk` 仅重新导出上述使用的核心接口。完整的 `daq_sdk` 公共 API（22 个名称）增加了以下内容——请直接从 `daq_sdk` 导入：

```python
from daq_sdk import (
    DAQULogger, DAQMLogger, DAQELogger,     # rotating-file recorders (the ones the GUI uses)
    ConnectResult, FleetRecordResult,       # SensorFleet result types
    discover_all_detailed, build_sensor,    # detailed discovery + build-by-descriptor
    scan_eth_devices, DaqEControl,          # DAQ-E Ethernet scan + control channel
    scan_ble_devices, detect_ble_device, list_ble_devices,   # DAQ-M BLE discovery
    detect_port, list_serial_ports,         # DAQ-U serial-port discovery
    TcpSerial,                              # serial-over-TCP transport shim
)
```

---

## 异常

捕获基类以处理“Chloros中出现的任何错误”：

```python
import chloros_sdk

try:
    chloros_sdk.process_folder("/path/to/folder")
except chloros_sdk.ChlorosAuthenticationError:
    print("Run `chloros-cli login` first.")
except chloros_sdk.ChlorosLicenseError:
    print("Chloros+ subscription required.")
except chloros_sdk.ChlorosError as e:
    print(f"Chloros error: {e}")
```

> `ChlorosAuthenticationError` 和 `ChlorosConfigurationError` 与其他内容一同在顶级导出；如图所示，它们也可从 `chloros_sdk.exceptions` 导入。

层次结构：

```

ChlorosError
├── ChlorosBackendError           (backend failed to start / unreachable)
├── ChlorosConnectionError        (HTTP transport failure)
├── ChlorosLicenseError           (subscription / tier gate)
├── ChlorosAuthenticationError    (login required)
├── ChlorosConfigurationError     (bad configure() / open_project() inputs)
└── ChlorosProcessingError        (pipeline failed)

ChlorosConnectError                (raised by connect_camera / connect_array /
                                    connect_daq_sensor only — derives from
                                    plain Exception, NOT from ChlorosError,
                                    so `except ChlorosError` will not catch it)

lattice_sdk exceptions:
LatticeError
├── CameraNotFoundError
├── CameraConnectionError
├── StreamError
├── CaptureError
├── CalibrationError
├── NetworkError
└── DLSError
```

---

## 端到端示例

### 1. 使用自定义进度条处理文件夹

```python
from chloros_sdk import ChlorosLocal

def progress(percent, message):
    bar = "#" * (percent // 5)
    print(f"\r[{bar:<20s}] {percent:3d}% {message}", end="", flush=True)

with ChlorosLocal() as cl:
    cl.create_project("FieldA_2026-05-26")
    cl.import_images("C:/DroneImages/Flight001", recursive=True)
    cl.configure(
        debayer="High Quality (Faster)",
        vignette_correction=True,
        reflectance_calibration=True,
        indices=["NDVI", "NDRE", "GNDVI", "SAVI"],
        export_format="TIFF (16-bit)",
    )
    cl.process(progress_callback=progress)
print()
```

### 2. 实时 LATTICE 阵列 → 反射率 + DAQ 参考

```python
import chloros_sdk

# Open a paired sensor first so the array's reflectance step has an
# absolute reference. Smart-detect picks USB / BLE / ETH automatically.
with chloros_sdk.connect_daq_sensor() as daq:
    with chloros_sdk.connect_array([
            "213800234", "214000533", "214701288", "214701292"
    ]) as arr:
        # Smart capture: wait for AE to settle, then snap
        arr.capture("./out", processing="reflectance", smart=True)

        # Record the corresponding DAQ frames as ground truth
        daq.record_start(output_dir="./out", device_name="sky-reference")
        # ... do whatever capture campaign ...
        info = daq.record_stop()
        print(info["path"], info["rows"])
```

### 3. 项目驱动的采集任务

```python
import time, chloros_sdk

with chloros_sdk.open_project("/home/user/Chloros Projects/Field_A") as proj:
    report = proj.connect_all(verbose=True, align=True)
    if report["arrays"]["errors"]:
        raise SystemExit(f"Array(s) failed to connect: {report['arrays']['errors']}")

    rig = proj.arrays["main_rig"]

    # Re-align right before the campaign
    rig.calibrate_alignment(num_frames=5)
    rig.export_alignment("./alignments/main_rig.json")

    # 50 sequential single-frame captures at 2 fps
    for i in range(50):
        frames = rig.capture(
            output_dir=f"./out/frame_{i:04d}",
            processing="reflectance",
            apply_calibration=True,
            apply_white_balance=True,
        )
        time.sleep(0.5)

    # End-of-day: process the captured folder. process() accepts only
    # mode/wait/progress_callback/poll_interval — indices come from the
    # project's saved config (or set them via ChlorosLocal.configure()).
    proj.process()
```

### 4. 多摄像头帧流 → NumPy 处理管道

```python
import chloros_sdk
import numpy as np

with chloros_sdk.open_project("/path/to/proj") as proj:
    proj.connect_all()
    rig = proj.arrays["main_rig"]

    for frames in rig.frame_stream(
            processing="radiance",
            fps=5.0, count=300,
            apply_calibration=True,
            apply_white_balance=True):
        # frames is {serial: numpy_array}; cams not delivering this tick are omitted
        for serial, frame in frames.items():
            print(serial, frame.shape, frame.dtype, frame.mean())
```

### 5. 无头直接硬件（无后端）采集脚本

```python
from chloros_sdk import LatticeCamera, PRESETS, discover_cameras

cams = discover_cameras(timeout_ms=3000)
print(f"Found {len(cams)} cams")

settings = PRESETS["high_quality"]
for c in cams:
    with LatticeCamera(serial=c.serial, settings=settings) as cam:
        result = cam.capture(output_dir="./out", format="tiff")
        print(c.serial, result.filepath)
```

### 6. 连接 4 摄像头阵列前的功能检测

```python
import chloros_sdk

serials = ["214701288", "213800234", "214000533", "214701162"]

probe = chloros_sdk.analyze_array_network(
    master_serial=serials[0],
    slave_serials=serials[1:],
    width=2048, height=1536,
    pixel_format="BayerRG12",
)

if probe["status"] == "ok":
    arr = chloros_sdk.connect_array(
        serials, width=2048, height=1536, pixel_format="BayerRG12")
elif probe["status"] == "auto_capped_fps":
    r = probe["recommended"]
    print(f"Keeping resolution; capping trigger rate at "
          f"{r['recommended_target_fps']} fps")
    arr = chloros_sdk.connect_array(
        serials, width=2048, height=1536, pixel_format="BayerRG12",
        target_fps=r["recommended_target_fps"])
elif probe["status"] == "auto_shrunk":
    r = probe["recommended"]
    print(f"Auto-shrinking to {r['out_width']}x{r['out_height']} "
          f"binning={r['binning']} for sim-sync")
    arr = chloros_sdk.connect_array(
        serials,
        width=r["out_width"], height=r["out_height"],
        pixel_format=r["pixel_format"], binning=r["binning"])
elif probe["status"] == "needs_force_slip":
    print("Wire can't sustain sim-sync; falling back to slip mode")
    arr = chloros_sdk.connect_array(
        serials, force_tier="slip-emit-and-capture")
else:
    raise RuntimeError(f"Probe error: {probe.get('error')}")
```

### 7. 等效的捕获配方（纯Python）

CLI的配方DSL在Python中具有直接等效实现：

```python
import time, chloros_sdk

with chloros_sdk.open_project("/path/to/proj") as proj:
    proj.connect_all()
    cam = proj.cameras["FrontLeft"]
    rig = proj.arrays["main_rig"]
    sky = proj.sensors["Sky"]

    # apply
    # (CameraHandle has no direct apply method; use the underlying lattice_sdk
    #  helper or the backend's /api/camera/<sn>/apply-settings via requests)
    # For most cases just use cam.cam.set_exposure(...) in direct mode or
    # the GUI's saved settings via project.connect_all().

    # wait
    time.sleep(2)

    # capture
    cam.capture("pose_a/", format="tiff", processing="radiance")

    # stream
    rig.stream(count=60, fps=5, output_dir="stream/", processing="raw")

    # sensor read
    print(sky.read())
```

---

## 后端自动启动

智能连接（smart-connect）入口点——`connect_camera`、`connect_array`、`connect_daq_sensor` 和 `discover_lattice_cameras` — 是轻量级的HTTP客户端，它们假定后端正在监听`127.0.0.1:5000`（即smart-connect接口的默认URL）。当GUI或CLI已运行时，后端便会处于运行状态。但在仅运行脚本的环境中，后端可能尚未启动——因此这些函数会在首次调用前**自动启动捆绑的后端二进制文件** （无窗口模式，与`ChlorosLocal`相同）并在首次调用前等待最长`backend_startup_timeout`的时间，直至其启动完成。

规则：

- **仅会启动本地URL。** 指向 `localhost` / `127.0.0.1` / `[::1]` 的 `backend_url` 符合条件；任何其他主机均被视为他人的 机器，因此绝不会被创建。
- **后端将保持运行状态以便复用**（与CLI相同）——脚本退出时不会自动关闭后端。重新运行脚本将复用现有的后端。
- 在上述任何调用中使用**`auto_start_backend=False`**进行退出 （例如，当你指定了远程后端，或自行管理后端生命周期时）。

```python
import chloros_sdk

# Fresh shell, no backend running, no GUI open — this still works:
with chloros_sdk.connect_camera("213800234") as cam:   # spawns the backend
    cam.capture("output/")

# Remote backend (via tunnel — see Remote-Backend Mode): don't spawn one locally
arr = chloros_sdk.connect_array(serials,
                                backend_url="http://127.0.0.1:5000",
                                auto_start_backend=False)
```

如果无法找到或启动捆绑的二进制文件，后续的 HTTP 调用将触发一个可处理的、**支持平台感知型**的`ChlorosConnectError`错误，而非简单的连接拒绝跟踪信息——在Windows上，它会引导您使用桌面应用程序或`chloros-cli`命令；在Linux（无GUI）上，它会引导您使用`chloros-cli` 命令或 `.deb`。

---

## 环境与头文件

SDK会为每个后端 HTTP 调用标记 `X-Chloros-Client: sdk`。 后端应用 SDK / CLI 的授权规则（需登录 **且** 拥有付费的 Chloros+ 套餐），而非 GUI 免费层级路径。此设置会在导入时自动完成——您无需进行任何操作。

`http://localhost` 和 `http://127.0.0.1` 被识别为本地后端。对其他主机 （例如您自己的分析服务）将保持不变。

通过传入 `backend_url=`（或在 `ChlorosLocal` 上传入 `api_url=`）来覆盖后端 URL：

```python
chloros_sdk.connect_camera("213800234", backend_url="http://127.0.0.1:5000")
chloros_sdk.connect_array(serials, backend_url="http://127.0.0.1:5000")
chloros_sdk.connect_daq_sensor(eth_host="daq-e-1.local",
                                backend_url="http://127.0.0.1:5000")
chloros_sdk.ChlorosLocal(backend_url="http://127.0.0.1:5000")
```

（非回环的 `backend_url` 仅能到达源/设备后端——随附的后端仅绑定回环；有关隧道模式，请参阅“远程后端模式”。）

---

## 版本控制与兼容性

- SDK 版本以 `chloros_sdk.__version__` 的形式对外提供。
- SDK 将行为与捆绑的后端版本绑定。将较旧的 SDK 与较新的后端混合使用通常可行（向前兼容的端点）， 但若将较新的 SDK 与较旧的后端混合使用，可能会在新端点上引发 `404` 错误——请将桌面应用升级至匹配版本。
- 智能连接接口（`connect_camera` / `connect_array` / `connect_daq_sensor`）和网络分析端点返回稳定的JSON模式；新字段均为补充性字段。

---

## 故障排除提示

- **`ChlorosAuthenticationError: Login required`** → 在该机器上运行一次 `chloros-cli login EMAIL PASSWORD`，或通过 Chloros 桌面应用程序登录。
- **`ChlorosConnectError: No Chloros backend is running …`** → 智能连接调用会自动启动本地后端，因此该提示仅在捆绑的二进制文件无法（例如：仅安装pip且无桌面包的主机）。该提示信息会根据平台自动调整：在 Windows 上，请打开桌面应用或运行任意 `chloros-cli` 命令；在 Linux 上，请运行 `chloros-cli` 命令（该平台无图形界面） 或安装 `.deb`。对于远程后端，请传递 `backend_url=`（以及 `auto_start_backend=False`）。
- **`CAMERA_AVAILABLE == False`** 在导入时 → `lattice_sdk` 加载失败（通常是因为未安装 Arena SDK 运行时 DLL）。非摄像机表面仍可正常工作。
- **数组连接返回低于原生分辨率**→ 后端智能预处理会自动缩小帧尺寸以适应传输带宽。使用 `analyze_array_network()` 查明原因，然后升级链接、接受缩小，或传递 `force_tier="slip-emit-and-capture"` 进行顺序捕获。该缩小机制的安全保护**不**涵盖聚合超额订阅（`oversubscribed: true`，fps 字段为 0）：当摄像机数量超过网络带宽时，无法通过像素合并或 ROI 来解决 ——请减少摄像头数量、启用巨型帧，或更换更快的网卡（参见 [超订阅](#over-subscription-the-per-cam-floor)）。
- **`analyze_array_network()` 报告 NIC 接收环容量过小（~0.26 MB）/ 连接门上显示“FRAMES WILL DROP”** → 主机网卡的接收环处于默认状态（网卡驱动更新后通常会重置为 32）。在 Realtek USB 10GbE 适配器上，设置 `ReceiveBufferLen=256` 和 `PendingReceives=64`（提升权限）， 随后重启后端使其重新读取环。完整操作流程：[CLI 参考 → 主机网卡设置与调优](cli-reference.md#host-nic-setup--tuning-lattice-arrays)。
- **主机在重启/关机时卡死，随后出现 WMI `Invalid class` 错误 / 网卡无法启用** → 过时的 USB 10GbE 驱动程序导致 `DRIVER_POWER_STATE_FAILURE`（蓝屏 `0x9F`）。将适配器驱动程序更新至最新版本（≥ 2026），并重新-应用接收环设置。参见 [CLI 参考 → 主机网卡设置与调优](cli-reference.md#host-nic-setup--tuning-lattice-arrays)。
- **反射率测量被拒绝** → 获取绝对刻度反射率时，必须将运行中的数据采集（DAQ）与摄像头（或阵列）绑定。可通过图形用户界面（GUI）进行绑定，或使用 `processing="radiance"`（W/m²/sr/nm），该模式无需配对传感器。
- **`smart=True` 捕获时间长于预期** → AE 收敛速度取决于场景动态；若需更快的 （稳定性较低）的触发器，请收紧`exposure_tolerance_pct`或缩短`stability_window_s`。

---

## 参见

- [CLI 参考文档](cli-reference.md) — 每个CLI子命令都对应一个SDK调用。
- [DAQ 传感器指南](../daq/README.md) — 针对特定传感器的接线、校准和记录规则。
- 在线文档：`https://mapir.gitbook.io/chloros/api-python-sdk`</id></sn>
