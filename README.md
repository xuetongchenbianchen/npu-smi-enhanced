# npu-smi-enhanced

**English** | [简体中文](README.zh-CN.md)

A lightweight command-line monitor for Huawei Ascend NPUs, with an `nvidia-smi`-style view of device health, utilization, memory, and running processes.

The tool calls the local Ascend driver's DCMI interfaces through Python's standard-library `ctypes` module. It requires no PyTorch, torch-npu, or third-party Python packages. Monitoring uses initialization and query interfaces only, without creating an ACL compute context.

## Features

- Logical device IDs, model names, Card/Chip mappings, PCIe bus IDs, and health status.
- Driver and CANN versions, with a timestamp for each snapshot.
- Temperature, power readings, and current AI Core frequency.
- **AI Core utilization** and HBM bandwidth utilization.
- Used and total HBM capacity, with memory usage percentages.
- Per-device process IDs, memory consumption, and command lines.
- Periodic refresh and JSON output for scripts.

> AI Core utilization and memory usage are separate metrics. Using 85% of device memory does not mean that compute utilization is 85%.

## Preview

The abbreviated example below uses illustrative values and a sample process name and PID:

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

## Requirements and tested environment

| Component | Requirement / tested configuration |
| --- | --- |
| Operating system | 64-bit Linux with access to the Ascend driver and NPU devices |
| Python | Python 3.6+; standard library only |
| Driver library | `/usr/local/Ascend/driver/lib64/driver/libdcmi.so` |
| Tested devices | Reported by the driver as `Ascend 9382`; 8 Cards and 16 logical NPUs |
| Tested driver | `25.5.2` |
| Tested CANN | `9.1.0` |

This initial version targets the configuration above. Other Ascend models, driver versions, and container setups have not been verified. The `ctypes` structures follow that driver's `dcmi_interface_api.h`; check interface and structure compatibility before using a different driver version.

Hardware queries do not require Python ACL. If the CANN version cannot be read, it is displayed as `N/A` while other queries continue.

## Quick start

```bash
git clone https://github.com/xuetongchenbianchen/npu-smi-enhanced.git
cd npu-smi-enhanced

python3 ./npu-smi-enhanced
```

### Install as a separate command

Install as `npu-smi-enhanced` to keep your existing system command available:

```bash
sudo install -m 0755 ./npu-smi-enhanced /usr/local/bin/npu-smi-enhanced
npu-smi-enhanced
```

For a user-local installation:

```bash
mkdir -p "$HOME/.local/bin"
install -m 0755 ./npu-smi-enhanced "$HOME/.local/bin/npu-smi-enhanced"
export PATH="$HOME/.local/bin:$PATH"
```

### Optional: use the `npu-smi` command name

To run this tool as `npu-smi`, first back up any existing file at the destination:

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

Only the monitoring options documented here are implemented. This tool **does not support all subcommands of the official `npu-smi` utility**. Use the separate command name if the official utility is already installed.

## Usage

```bash
# Take one snapshot; "info" is optional
npu-smi-enhanced
npu-smi-enhanced info

# Wait 2 seconds after each snapshot; press Ctrl+C to exit
npu-smi-enhanced -l 2

# Print JSON, including full process command lines and query errors
npu-smi-enhanced --json

# Stream JSON Lines: one snapshot per line
npu-smi-enhanced --json -l 2

# Show help
npu-smi-enhanced --help
```

If installed as `npu-smi`, substitute that command name in these examples. The `-l` value must be a positive, finite number. The interval between snapshot starts includes both query time and the requested delay. Table output refreshes in place in an interactive terminal; redirected output appends successive snapshots.

### JSON output

Top-level fields are `timestamp`, `driver`, `cann`, `devices`, and `query_errors`.

Each device object includes:

| Field | Meaning |
| --- | --- |
| `npu` | Logical device ID returned by DCMI |
| `card` / `chip` | DCMI Card ID / chip ID within that Card |
| `name` / `bus_id` | Device model / PCIe address |
| `health` | Health status code returned by the driver |
| `temperature_c` / `power_w` / `clock_mhz` | Temperature / power / AI Core frequency |
| `aicore_percent` | AI Core utilization, as a percentage |
| `memory_used_mib` / `memory_total_mib` | Used / total HBM capacity |
| `memory_percent` | Percentage of HBM capacity in use |
| `hbm_bandwidth_percent` | HBM bandwidth utilization |
| `processes` | Process array; each entry contains `pid`, `memory_mib`, and `command` |

Unavailable metrics are `null`. `processes: []` means the query succeeded and returned no processes; `processes: null` means the query was unavailable. Individual query failures leave other metrics intact and are recorded in `query_errors`. Initialization or device enumeration failures print an error and exit with a nonzero status.

## Understanding the readings

### HBM capacity versus ACL memory queries

This tool displays the full HBM capacity reported by DCMI, including memory used by the device system. Scripts based on `acl.rt.get_mem_info()` use the runtime's memory accounting, which can differ. For example, the same device may show `65536 MiB` here and approximately `62740 MiB` in an ACL-based script.

The sum of process memory usage may differ from total device memory usage. Device system and driver allocations also consume memory, and the individual queries do not form an atomic snapshot.

### Device IDs and container processes

- The `NPU` column uses driver logical IDs; `[Card/Chip]` shows the DCMI mapping. Card IDs are not treated as logical NPU IDs.
- Enumeration follows DCMI and device access permissions. It does not filter devices according to `ASCEND_RT_VISIBLE_DEVICES` or a framework's visible-device list.
- Process IDs come from the driver. Command lines are read from `/proc/<pid>/cmdline` in the current environment.
- Container PID namespaces or permissions may prevent command-line lookup, producing `[PID not visible in this namespace]`. Host-to-container PID translation is not implemented.
- Table output shortens absolute paths and truncates long commands. `--json` preserves full command lines; check for private arguments before sharing that output.

### Power and other metrics

Power readings are converted from DCMI values to watts. Their measurement scope and update frequency depend on the hardware and driver; summing readings across logical NPUs does not necessarily give whole-system power. The tool does not estimate power limits, fan speeds, or NVIDIA-specific performance states.

## Troubleshooting

**`libdcmi.so` cannot be found**

Check that the Ascend driver is installed and the library exists at the path listed above. Containers also need access to the driver library, its dependencies, and device nodes. This initial version uses a fixed default driver path.

**`dcmi_init failed` or device enumeration fails**

Check driver health, device nodes, and your user's permissions. Use `sudo` if your deployment requires administrator access. The tool does not repair drivers or change device configuration.

**Some fields show `N/A`**

Run `npu-smi-enhanced --json` and inspect `query_errors`. Missing interfaces, unsupported hardware, or access restrictions may cause these errors. Failed queries are not displayed as zero.

**The CANN version shows `N/A`**

Version discovery currently checks the toolkit's default `latest` paths and the CANN 9.1.0 path used in the tested environment. Custom installation paths may not be detected.

**Restore the previous script**

If you created the backup described above:

```bash
sudo cp --preserve=mode,timestamps \
    /usr/local/bin/npu-smi.acl-backup /usr/local/bin/npu-smi
hash -r
```

## Implementation and validation

The program is a single executable Python file. Validation on the 16 logical NPUs listed above covered enumeration, unique logical IDs, metric ranges, query return codes, table alignment, process queries, and JSON output.

Interface definitions are based on `/usr/local/Ascend/driver/include/dcmi_interface_api.h` from the tested environment. See also the [Ascend DCMI device management documentation](https://www.hiascend.com/doc_center/source/zh/HDK/2610/A3/A3DCMI/dcmia3_006.html).

This is an independent monitoring script, not Huawei's official `npu-smi` utility.
