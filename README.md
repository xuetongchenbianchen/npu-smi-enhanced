# npu-smi-enhanced

一个面向华为昇腾 NPU 的轻量级命令行监控工具，以类似 `nvidia-smi` 的表格展示设备状态和进程显存占用。

通过 Python 标准库 `ctypes` 调用本机 Ascend 驱动的 DCMI 接口，不需要安装 PyTorch、torch-npu 或第三方 Python 包。监控过程只调用查询和初始化接口，不创建 ACL 计算上下文。

## 能看到什么

- 设备逻辑 ID、型号、Card/Chip 映射、PCIe Bus ID、健康状态。
- 驱动版本、CANN 版本、当前时间。
- 温度、功耗、AI Core 当前频率。
- **AI Core 利用率**、HBM 带宽利用率。
- HBM 已用容量、总容量、内存占用百分比。
- 每个 NPU 上的进程 PID、显存占用及命令行。
- 定时刷新，以及便于脚本处理的 JSON 输出。

> AI Core 利用率与内存占用率是两个不同指标。本工具会分别展示，避免把“显存占了 85%”误读成“算力使用了 85%”。

## 输出预览

以下为缩减后的示意数据，进程名称和 PID 为示例：

```text
+--------------------------------------+--------------------------------+---------------------------------+
| NPU-SMI (DCMI)                       | Driver: 25.5.2                 | CANN: 9.1.0                     |
+--------------------------------------+--------------------------------+---------------------------------+
| NPU  Name  [Card/Chip]               | Bus-Id / Health                | HBM Memory-Usage                |
| Temp   Power   AI-Core Clock         | AI-Core Util / HBM BW          | Memory Used / Total / Used %    |
+======================================+================================+=================================+
|   0  Ascend 9382 [0/0]               | 0000:9D:00.0 / OK              | 55884 / 65536 MiB               |
| 57C   350.6W   1800MHz               | 70% / 44%                      | Memory Used: 85.3%              |
+--------------------------------------+--------------------------------+---------------------------------+
| Processes:  NPU       PID   Type    NPU Memory   Process command                                        |
+=========================================================================================================+
|               0     12345      C   52803.5MiB   python3 inference.py                                    |
+---------------------------------------------------------------------------------------------------------+
```

## 环境要求与验证范围

| 项目 | 要求 / 已验证环境 |
| --- | --- |
| 操作系统 | 64 位 Linux，能够访问 Ascend 驱动和 NPU 设备 |
| Python | Python 3.6+，仅使用标准库 |
| 驱动库 | `/usr/local/Ascend/driver/lib64/driver/libdcmi.so` |
| 已验证设备 | 驱动报告为 `Ascend 9382`，8 个 Card、16 个逻辑 NPU |
| 已验证驱动 | `25.5.2` |
| 已验证 CANN | `9.1.0` |

首版针对上述环境实现，尚未在所有昇腾型号、驱动版本和容器部署方式上验证。`ctypes` 结构体依据该环境的 `dcmi_interface_api.h` 定义；使用其他驱动版本前，应确认接口和结构体兼容。

硬件查询本身不依赖 Python ACL。CANN 版本读取失败时显示 `N/A`，不会阻止其他指标查询。

## 快速开始

```bash
git clone https://github.com/xuetongchenbianchen/npu-smi-enhanced.git
cd npu-smi-enhanced

python3 ./npu-smi-enhanced
```

### 安装为独立命令

推荐保留现有系统命令，把本工具安装为 `npu-smi-enhanced`：

```bash
sudo install -m 0755 ./npu-smi-enhanced /usr/local/bin/npu-smi-enhanced
npu-smi-enhanced
```

没有系统目录写权限时，可以安装到用户目录：

```bash
mkdir -p "$HOME/.local/bin"
install -m 0755 ./npu-smi-enhanced "$HOME/.local/bin/npu-smi-enhanced"
export PATH="$HOME/.local/bin:$PATH"
```

### 可选：使用 `npu-smi` 命令名

如果你希望像原来的简易脚本一样，直接输入 `npu-smi`，先备份同路径下已有的文件：

```bash
if [ -e /usr/local/bin/npu-smi ]; then
    sudo cp --backup=numbered --preserve=mode,timestamps \
        /usr/local/bin/npu-smi /usr/local/bin/npu-smi.acl-backup
fi
sudo install -m 0755 ./npu-smi-enhanced /usr/local/bin/npu-smi
hash -r
command -v npu-smi
npu-smi
```

本工具只实现本文列出的监控参数，**不兼容原厂 `npu-smi` 的全部子命令**。已有原厂工具时，建议使用独立命令名。

## 使用方法

```bash
# 单次查询（info 可以省略）
npu-smi-enhanced
npu-smi-enhanced info

# 每次采集完成后等待 2 秒再刷新；Ctrl+C 退出
npu-smi-enhanced -l 2

# 输出完整 JSON，含未截断的进程命令行及查询错误
npu-smi-enhanced --json

# 持续输出 JSON Lines，每次快照占一行
npu-smi-enhanced --json -l 2

# 帮助
npu-smi-enhanced --help
```

安装为 `npu-smi` 后，将上述命令名替换为 `npu-smi` 即可。`-l` 必须是大于零的有限数字；实际两次采样开始的间隔为查询耗时加等待时间。在交互终端中自动清屏刷新，重定向输出时按快照追加。

### JSON 输出

顶层字段包括 `timestamp`、`driver`、`cann`、`devices` 和 `query_errors`。

每个设备对象包含以下核心字段：

| 字段 | 含义 |
| --- | --- |
| `npu` | DCMI 返回的逻辑设备 ID |
| `card` / `chip` | DCMI Card ID / Card 内芯片 ID |
| `name` / `bus_id` | 型号 / PCIe 地址 |
| `health` | 驱动返回的健康状态码 |
| `temperature_c` / `power_w` / `clock_mhz` | 温度 / 功耗 / AI Core 频率 |
| `aicore_percent` | AI Core 利用率，百分比 |
| `memory_used_mib` / `memory_total_mib` | HBM 已用 / 总容量 |
| `memory_percent` | HBM 已用容量占比 |
| `hbm_bandwidth_percent` | HBM 带宽利用率 |
| `processes` | 进程数组，每项含 `pid`、`memory_mib`、`command` |

不可用的指标输出为 `null`。`processes: []` 表示查询成功但没有返回进程，`processes: null` 表示查询不可用。局部查询失败会保留其他成功指标，并记录到 `query_errors`；初始化或设备枚举失败则打印错误并以非零状态退出。

## 如何理解数据

### HBM 容量与旧版 ACL 脚本不同

本工具显示 DCMI 报告的完整 HBM 容量，包括设备系统内存占用。原先使用 `acl.rt.get_mem_info()` 的脚本展示的是运行时的内存口径，两者可能不同。例如，同一设备可能在这里显示 `65536 MiB`，在 ACL 脚本中显示约 `62740 MiB`。

进程显存之和不一定等于设备已用显存：设备系统、驱动等也会占用内存，各项查询也并非原子快照。

### 设备编号与容器进程

- `NPU` 列使用驱动逻辑 ID；`[Card/Chip]` 展示实际 DCMI 映射，不把 Card ID 当作逻辑 NPU ID。
- 枚举范围由 DCMI 和设备访问权限决定，不按 `ASCEND_RT_VISIBLE_DEVICES` 或框架自身的可见设备列表筛选。
- 进程 PID 来自驱动，命令行从当前环境的 `/proc/<pid>/cmdline` 读取。
- 容器 PID 命名空间或权限可能使某些命令行不可读，此时显示 `[PID not visible in this namespace]`；不做宿主机 PID 与容器 PID 的自动映射。
- 表格会简化绝对路径并截断长命令，`--json` 保留完整命令行。分享 JSON 前请检查命令行中是否包含私密参数。

### 功耗与其他指标

功耗按 DCMI 接口的原始值转换为瓦特，其采样范围和更新频率取决于硬件及驱动；不要直接把所有逻辑 NPU 的读数相加当作整机功耗。工具不估算功耗上限、风扇转速或 NVIDIA 特有的性能状态。

## 常见问题

**找不到 `libdcmi.so`**

确认已安装 Ascend 驱动，且上述库路径存在。容器中还需由运行环境正确提供驱动库、其依赖及设备节点。首版使用固定的默认驱动路径。

**`dcmi_init failed` 或设备枚举失败**

检查驱动状态、设备节点和当前用户权限。需要管理员权限的部署环境可用 `sudo` 执行。该工具不会自动修复驱动或修改设备配置。

**某些字段为 `N/A`**

执行 `npu-smi-enhanced --json`，查看 `query_errors`。这通常表示接口缺失、当前硬件不支持或权限受限。工具不会把失败查询显示成零。

**CANN 版本为 `N/A`**

目前检查 toolkit 的默认 `latest` 路径和已验证环境的 CANN 9.1.0 路径；自定义安装目录可能无法识别版本。

**恢复先前的简易脚本**

如果按上面的步骤做了备份，可以恢复：

```bash
sudo cp --preserve=mode,timestamps \
    /usr/local/bin/npu-smi.acl-backup /usr/local/bin/npu-smi
hash -r
```

## 实现与验证

主程序为单个可执行 Python 文件。开发时已在上述 16 个逻辑 NPU 上验证设备枚举、编号唯一性、指标范围、查询返回码和表格对齐，并检查进程查询和 JSON 输出。

接口依据本机驱动头文件 `/usr/local/Ascend/driver/include/dcmi_interface_api.h`。可参考[昇腾 DCMI 设备管理接口文档](https://www.hiascend.com/doc_center/source/zh/HDK/2610/A3/A3DCMI/dcmia3_006.html)。

本项目为独立监控脚本，并非华为官方 `npu-smi`。
