# btop: A Practical User Reference

The official btop documentation covers installation and configuration options thoroughly. What it does not cover is what you are actually looking at once btop is running — what each panel shows, what the metrics mean, and how to navigate the interface efficiently.

This reference fills that gap.

---
## Installation

### Package managers

**Arch Linux**

```bash
sudo pacman -S btop
```

For GPU monitoring support:

```bash
yay -S btop-gpu
```

**Ubuntu / Debian**

```bash
sudo apt install btop
```

**Fedora**

```bash
sudo dnf install btop
```

**RHEL / Rocky / AlmaLinux 8+**

```bash
sudo dnf install epel-release
sudo dnf install btop
```

**macOS**

```bash
brew install btop
```

### Launch flags

|Flag|Description|Example|
|---|---|---|
|`-c, --config <file>`|Use a specific config file|`btop -c ~/btop-minimal.conf`|
|`-d, --debug`|Start with debug logging|`btop --debug`|
|`-f, --filter <filter>`|Start with a process filter applied|`btop -f nginx`|
|`-l, --low-color`|256-colour mode only|`btop --low-color`|
|`-p, --preset <id>`|Start with a specific preset|`btop -p 2`|
|`-t, --tty`|Force TTY mode|`btop --tty`|
|`-u, --update <ms>`|Set update interval|`btop -u 1000`|
|`--force-utf`|Skip UTF-8 locale check|`btop --force-utf`|
|`--default-config`|Print default config to stdout|`btop --default-config > ~/.config/btop/btop.conf`|

---

## The Interface at a Glance

btop divides the screen into five panels, each monitoring a different aspect of system activity:

|Panel|What it monitors|
|---|---|
|**CPU**|Processor usage across all cores|
|**Memory**|RAM and swap consumption|
|**Disk**|Storage read/write activity and usage|
|**Network**|Upload and download throughput|
|**Processes**|Running processes sorted by resource usage|

By default, CPU sits at the top, Memory and Network occupy the middle, and Processes fills the right side. This layout is configurable using presets — covered in the Presets section below.

---

## The CPU Panel

The CPU panel shows processor activity as a dual-line graph, with usage history scrolling from right to left in real time.

### The graph

The upper graph line and lower graph line each display a different CPU metric. By default both are set to Auto, which means btop selects the most informative available stat for your hardware. You can change what each line shows from the options menu — common choices include total usage, user space usage, system usage, and CPU steal (relevant on virtual machines).

The lower graph line is inverted by default, so it extends downward from the centre. This makes it visually easier to distinguish the two metrics at a glance.

### Core usage bars

To the right of the graph, btop shows individual usage bars for each logical CPU core. On a system with hyperthreading, each physical core appears as two logical cores. The bars update every two seconds by default.

### CPU metrics explained

**Usage %** — the percentage of CPU time spent on active work during the last update interval. This includes both user processes and kernel activity. A sustained reading above 80–90% across all cores indicates the system is under significant load.

**User** — CPU time consumed by user-space processes. This is the work your applications are doing.

**System** — CPU time consumed by the kernel. Elevated system usage (above 20–30%) can indicate heavy I/O activity, frequent context switching, or driver issues.

**Steal** — on virtual machines, steal represents CPU time the hypervisor has taken away from your VM to serve other tenants. Persistent steal above 5–10% means your VM is being resource-constrained by the host.

**Frequency** — the current operating frequency of the CPU in GHz. Modern processors scale frequency dynamically based on load and thermal conditions. A CPU running consistently below its rated frequency may be thermal throttling.

**Temperature** — core temperature in Celsius (or your configured scale). Sustained temperatures above 80–85°C on most consumer processors indicate inadequate cooling or heavy sustained load.

**Uptime** — time elapsed since the last system boot, displayed in the top right of the CPU panel.

**Wattage** — current power draw of the CPU package in watts, if supported by your hardware. Requires extended capabilities to be set at installation (`make setcap`).

---

## The Memory Panel

The memory panel shows current RAM and swap consumption as both graphs and percentage bars.

### Memory metrics explained

**Used** — memory actively in use by running processes. This is the most meaningful figure for assessing memory pressure.

**Available** — memory that can be allocated to new processes immediately, without the kernel needing to reclaim anything. This includes free memory plus memory that is currently cached but can be released on demand. Available is a more reliable indicator of actual free memory than the Free figure alone.

**Free** — memory not currently used for any purpose. On a healthy Linux system, free memory is typically low because the kernel aggressively uses spare RAM for caching. Low free memory is not a problem unless Available is also low.

**Cached** — memory used by the kernel to cache recently accessed files from disk. The kernel will reclaim this memory automatically when processes need it. High cached values are normal and indicate the disk cache is working effectively.

**Buffers** — memory used by the kernel for temporary storage of data in transit between processes and disk. Typically a small value.

**Swap Used** — the portion of swap space currently in use. Swap is disk space used as overflow when physical RAM is exhausted. Light swap usage (a few hundred MB) is normal on systems that have been running for a long time. Heavy or growing swap usage indicates memory pressure — the system is moving data to disk because RAM is full, which significantly impacts performance.

---

## The Disk Panel

The disk panel shows storage devices mounted on the system, with usage and I/O activity for each.

### Disk metrics explained

**Usage bar** — the filled portion represents used space as a percentage of total capacity. The label shows used and total capacity in human-readable units.

**Read / Write speeds** — current throughput in MiB/s for read and write operations. These values update each interval and reflect activity since the last measurement, not a running average.

**IO % (busy time)** — the percentage of time the disk was busy servicing I/O requests during the last interval. A disk at 100% IO is saturated — it is handling requests as fast as it can and any additional requests are queuing. Sustained high IO % on a system disk can cause broad performance degradation across all processes that read or write files.

**IO mode** — pressing `i` toggles IO mode, which replaces the usage bars with large scrolling graphs for read and write throughput. Useful when diagnosing disk-intensive workloads.

---

## The Network Panel

The network panel displays upload and download throughput as scrolling graphs, one for each direction.

### Network metrics explained

**Download (Rx)** — incoming network traffic in Kibibits or Mebibits per second. The graph scales automatically to the highest recent value, so the visual peak always represents the maximum observed throughput, not a fixed ceiling.

**Upload (Tx)** — outgoing network traffic. Displayed on the lower half of the panel, inverted by default.

**Auto scaling** — btop scales the graph axes dynamically based on actual throughput. This means the graph always uses the full vertical space regardless of whether you are transferring at 1 Mbps or 1 Gbps. The current scale is shown on the left axis.

**Interface selection** — if your system has multiple network interfaces (ethernet, Wi-Fi, VPN, loopback), press `y` to cycle between them. The active interface name is shown at the top of the panel.

---

## The Process Panel

The process panel lists all running processes and is the most interactive part of the btop interface.

### Column definitions

|Column|Meaning|
|---|---|
|**PID**|Process ID — a unique identifier assigned by the kernel at launch|
|**Program**|The name of the executable|
|**Arguments**|The full command including flags and arguments|
|**Threads**|Number of threads the process is running|
|**User**|The user account the process is running under|
|**Memory %**|Percentage of total RAM in use by this process|
|**CPU %**|Processor usage, either as a share of total CPU or per-core depending on configuration|

### Sorting

The process list is sorted by CPU usage by default, using lazy sorting — the top process updates gradually rather than jumping around constantly as CPU usage fluctuates. This makes it easier to identify which process is consistently consuming the most CPU over time.

Press `e` to switch to direct CPU sorting, which updates the top process immediately on each interval. Press `f` to open the sort field selector and choose any column as the sort key.

### Filtering

Press `f` to enter a filter string. btop will display only processes whose name or arguments match the input. This is useful on busy systems with many running processes. Clear the filter by pressing `f` again and deleting the input.

### Process details

Select any process using the arrow keys and press `Enter` to open the detailed view. This shows:

- Full command and arguments
- CPU and memory usage history graphs
- Open file descriptors
- Memory breakdown (if `proc_info_smaps` is enabled in config — slower but more accurate)
- Child processes

### Signals

With a process selected, press `s` to open the signal menu. btop provides 31 signals to choose from:

`SIGHUP`, `SIGINT`, `SIGQUIT`, `SIGILL`, `SIGTRAP`, `SIGABRT`, `SIGBUS`, `SIGFPE`, `SIGKILL`, `SIGUSR1`, `SIGSEGV`, `SIGUSR2`, `SIGPIPE`, `SIGALRM`, `SIGTERM`, `SIGCHLD`, `SIGCONT`, `SIGSTOP`, `SIGTSTP`, `SIGTTIN`, `SIGTTOU`, `SIGURG`, `SIGXCPU`, `SIGXFSZ`, `SIGVTALRM`, `SIGPROF`, `SIGWINCH`, `SIGIO`, `SIGPWR`, `SIGSYS`

The most commonly used signals are:

|Signal|Effect|
|---|---|
|`SIGTERM`|Graceful termination — asks the process to stop cleanly|
|`SIGKILL`|Forced termination — immediately kills the process, cannot be caught or ignored|
|`SIGHUP`|Reload configuration — commonly used to restart services without stopping them|
|`SIGSTOP`|Pause the process|
|`SIGCONT`|Resume a paused process|

For quick access without opening the menu, press `K` to send SIGKILL directly to the selected process, or `t` to send SIGTERM.

Sending signals to system processes or processes owned by other users requires btop to be running with elevated capabilities. See the Troubleshooting section for how to set this up.

### Tree view

Press `e` to toggle tree view, which groups child processes under their parent. This is useful for understanding process relationships — for example, seeing all worker threads or child processes spawned by a web server or database.

---

## Keyboard Shortcuts

btop is designed for keyboard-driven use. The following shortcuts are available from the main interface:

### Navigation

|Key|Action|
|---|---|
|`Arrow keys`|Move selection in process list|
|`Page Up / Page Down`|Scroll process list by page|
|`Home / End`|Jump to top or bottom of process list|
|`Tab`|Cycle focus between panels|

### Process controls
| Key     | Action                                    |
| ------- | ----------------------------------------- |
| `Enter` | Open detailed view for selected process   |
| `s`     | Open signal menu (31 signals available)   |
| `K`     | Send SIGKILL directly to selected process |
| `t`     | Send SIGTERM to selected process          |
| `f`     | Filter processes                          |
| `e`     | Toggle tree view                          |
| `r`     | Reverse sort order                        |
| `u`     | Pause / resume process list updates       |
| `N`     | Set nice value for selected process       |
| `F`     | Toggle following selected process         |
### Interface controls

|Key|Action|
|---|---|
|`m`|Open main menu|
|`o`|Open options menu|
|`h`|Open help menu|
|`p`|Cycle through presets (0–9)|
|`i`|Toggle IO mode for disk panel|
|`y`|Cycle network interface|
|`b`|Toggle battery meter|
|`+` / `-`|Increase / decrease update interval|
|`q`|Quit btop|

### Vim-style navigation (optional)

If `vim_keys = true` is set in the config file, the following keys become available for directional navigation in lists:

|Key|Action|
|---|---|
|`h`|Left|
|`j`|Down|
|`k`|Up (note: conflicts with kill signal — use `K` to kill when vim keys are active)|
|`l`|Right|
|`g`|Jump to top|
|`G`|Jump to bottom|

---

## Presets

Presets allow you to save and switch between different panel layouts. btop ships with three built-in presets and supports up to nine custom presets (0–9), with preset 0 always reserved for the default full layout.

Press `p` to cycle through available presets. The current preset number is shown in the top bar.

### Built-in presets

|Preset|Layout|
|---|---|
|0|All panels visible (default)|
|1|CPU and processes only|
|2|CPU, memory, and network|
|3|CPU with block graph symbols, network with TTY symbols|

### Defining custom presets

Presets are defined in the config file using the `presets` option. The format is:

```
presets = "box_name:position:graph_symbol,box_name:position:graph_symbol ..."
```

Each preset is separated by a space. Within a preset, each panel entry uses:

- `box_name` — one of `cpu`, `mem`, `net`, `proc`, or `gpu0` through `gpu5`
- `position` — `0` for default position, `1` for alternate position
- `graph_symbol` — `default`, `braille`, `block`, or `tty`

Example — two custom presets, the first showing only CPU and processes, the second showing all panels with block-style graphs:

```
presets = "cpu:1:default,proc:0:default cpu:0:block,mem:0:block,net:0:block,proc:0:block"
```

---

## Configuration Reference

All btop options can be changed from within the interface via the options menu (`o`), or by editing the config file directly at:

```
~/.config/btop/btop.conf
```

The following options are the most commonly adjusted. For the full list, run `btop --default-config`.

### Display

|Option|Default|Description|
|---|---|---|
|`color_theme`|`"Default"`|Theme name. Place custom themes in `~/.config/btop/themes/`|
|`theme_background`|`true`|Show theme background colour. Set to `false` for terminal transparency|
|`truecolor`|`true`|Use 24-bit colour. Set to `false` for 256-colour terminals|
|`rounded_corners`|`true`|Rounded panel borders. Disabled automatically in TTY mode|
|`graph_symbol`|`"braille"`|Graph style: `braille` (highest resolution), `block`, or `tty`|
|`clock_format`|`"%X"`|Clock format in strftime syntax. Use `/host`, `/user`, or `/uptime` for system info|

### Update rate

|Option|Default|Description|
|---|---|---|
|`update_ms`|`2000`|Refresh interval in milliseconds. Lower values increase CPU usage. 2000ms is recommended for accurate graph samples|

### CPU

|Option|Default|Description|
|---|---|---|
|`cpu_graph_upper`|`"Auto"`|Metric shown in upper CPU graph|
|`cpu_graph_lower`|`"Auto"`|Metric shown in lower CPU graph|
|`cpu_invert_lower`|`true`|Invert the lower graph direction|
|`cpu_single_graph`|`false`|Show only one CPU graph line|
|`show_uptime`|`true`|Display system uptime in CPU panel|
|`check_temp`|`true`|Show CPU temperature|
|`temp_scale`|`"celsius"`|Temperature scale: `celsius`, `fahrenheit`, `kelvin`, or `rankine`|
|`show_cpu_watts`|`true`|Show CPU power draw (requires `make setcap`)|

### Processes

|Option|Default|Description|
|---|---|---|
|`proc_sorting`|`"cpu lazy"`|Default sort column|
|`proc_tree`|`false`|Start in tree view|
|`proc_per_core`|`false`|Show CPU % as share of one core rather than total CPU|
|`proc_mem_bytes`|`true`|Show memory as bytes rather than percentage|
|`proc_filter_kernel`|`false`|Hide kernel threads from the process list|

### Memory

|Option|Default|Description|
|---|---|---|
|`show_swap`|`true`|Display swap usage|
|`swap_disk`|`true`|Show swap as a disk entry in the disk panel|
|`mem_graphs`|`true`|Show graphs instead of plain meters for memory|

### Network

|Option|Default|Description|
|---|---|---|
|`net_auto`|`true`|Auto-scale network graphs|
|`net_sync`|`true`|Sync upload and download graph scales|
|`net_iface`|`""`|Default network interface. Leave empty for auto-detection|

---

## Troubleshooting

### Graph characters are not rendering correctly

btop uses Unicode Braille characters for its graphs by default. If you see placeholder boxes, question marks, or misaligned characters, your terminal font does not include the required Unicode ranges.

Solutions:

- Switch to a font that includes Braille characters. Nerd Fonts and Terminess Powerline both include the required ranges.
- Alternatively, change `graph_symbol` to `block` or `tty` in the options menu. These use more common Unicode characters and will render correctly on most fonts.

If text is misaligned in Konsole or Yakuake specifically, disable "Bi-Directional text rendering" in the terminal settings.

### btop warns about UTF-8 locale

btop requires a UTF-8 locale to render its interface correctly. If you see a warning at startup, check your locale setting:

```bash
locale
```

The `LANG` or `LC_ALL` value should end in `.UTF-8`, for example `en_US.UTF-8`. To set it for the current session:

```bash
export LANG=en_US.UTF-8
```

To set it permanently, add the line to your shell profile (`~/.bashrc`, `~/.zshrc`, or equivalent), or configure it system-wide via your distribution's locale settings.

To bypass the check without changing your locale:

```bash
btop --force-utf
```

### Colours look wrong or washed out

If btop is displaying muted or incorrect colours, your terminal may not support 24-bit truecolor. Verify truecolor support with:

```bash
printf '\033[38;2;255;100;0mTRUECOLOR\033[0m\n'
```

If the output appears in orange, truecolor is supported. If it appears in a different colour or is not highlighted, set `truecolor = false` in the btop config to fall back to 256-colour mode.

### GPU metrics are not showing

GPU monitoring requires:

1. A btop binary compiled with `GPU_SUPPORT=true`
2. The correct driver and support library for your GPU (nvidia-ml for NVIDIA, rocm_smi_lib for AMD)

On Arch Linux, install the AUR package `btop-gpu` instead of the standard `btop` package for a pre-compiled binary with GPU support enabled.

To verify GPU support is active, start btop in debug mode:

```bash
btop --debug
```

GPU detection errors will appear in the log output.

### Cannot send signals to processes

If the signal menu is unresponsive or returns permission errors, btop does not have the required capabilities to send signals to processes owned by other users.

Fix by running after installation:

```bash
sudo make setcap
```

Or run btop with elevated privileges:

```bash
sudo btop
```

Note that running as root is not recommended for routine use. The `setcap` approach is preferred.

---



_For bug reports and feature requests, see the upstream tracker at [github.com/aristocratos/btop/issues](https://github.com/aristocratos/btop/issues)._