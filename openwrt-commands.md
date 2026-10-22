# Useful OpenWrt Commands Cheat Sheet

A practical reference for OpenWrt system debugging, services, networking, logs, package management, storage, and development.

> **Note:** OpenWrt commonly uses BusyBox, so not every command or option from a full Linux distribution will be available. Availability also depends on the OpenWrt release and installed packages.

## 1. Ubus — Runtime Information and RPC

`ubus` is OpenWrt's inter-process communication (IPC) mechanism. It lets you query system state and call methods exposed by running services.

| Command | Description |
|---|---|
| `ubus list` | List registered ubus objects |
| `ubus list -v` | List objects and their available methods |
| `ubus call system board` | Show device model, firmware version, kernel version, and target |
| `ubus call system info` | Show uptime, memory, and load averages |
| `ubus call service list` | List services managed by procd |
| `ubus call service list '{"name":"network"}'` | Show information about a specific service |
| `ubus call network.interface dump` | Show network interface and protocol status |
| `ubus call network.device status` | Show network device information |
| `ubus call log read` | Read the system log buffer |
| `ubus call session list` | List ubus sessions |
| `ubus monitor` | Monitor ubus messages and events |
| `ubus listen` | Listen for ubus events |
| `ubus -v list network.interface` | Show methods available on the network interface object |

### Particularly useful commands

Inspect available methods:

```sh
ubus -v list
```

Inspect the network interface API:

```sh
ubus -v list network.interface
```

Check system information:

```sh
ubus call system board
ubus call system info
```

Monitor IPC activity:

```sh
ubus monitor
```

`ubus monitor` can generate substantial output on a busy device.

## 2. Services and procd

OpenWrt uses `procd` as its init system and service supervisor.

| Command | Description |
|---|---|
| `ubus call service list` | List procd-managed services |
| `/etc/init.d/your_service status` | Check status, if supported by the init script |
| `/etc/init.d/your_service start` | Start a service |
| `/etc/init.d/your_service stop` | Stop a service |
| `/etc/init.d/your_service restart` | Restart a service |
| `/etc/init.d/your_service reload` | Reload configuration, if supported |
| `/etc/init.d/your_service enable` | Enable a service at boot |
| `/etc/init.d/your_service disable` | Disable a service at boot |
| `/etc/init.d/your_service enabled` | Check whether a service is enabled at boot |
| `ls /etc/init.d/` | List installed init scripts |
| `ls /etc/rc.d/` | List boot-time service symlinks |

Replace `your_service` with the actual service name.

### Inspect a service

For a service named `cwy_logging`:

```sh
ubus call service list '{"name":"cwy_logging","verbose":true}'
cat /etc/init.d/cwy_logging
```

Check whether a process is running:

```sh
ps
ps w
pgrep -af cwy_logging
```

The availability of `pgrep` and its supported options depends on the installed BusyBox utilities.

**Important:** A service appearing in the procd service list does not necessarily mean its daemon is currently running. Check the process list as well.

## 3. System Logs and Debugging

OpenWrt typically uses `logd` for local system logging.

| Command | Description |
|---|---|
| `logread` | Display the system log buffer |
| `logread -f` | Follow new log messages in real time |
| `logread -e error` | Filter messages containing `error` |
| `logread -l 100` | Display the last 100 log messages |
| `dmesg` | Display kernel messages |
| `dmesg -w` | Follow new kernel messages, if supported |
| `logread -f \| grep -i error` | Follow logs and filter for errors |
| `ls -l /dev/log` | Inspect the local logging socket |
| `ps w \| grep -E 'logd\|rsyslog\|syslog-ng'` | Look for common logging daemons |

### Kernel and driver debugging

```sh
dmesg | tail -n 100
dmesg | grep -iE 'error|fail|warn'
logread -f
```

Inspect logging configuration:

```sh
uci show system | grep log
cat /etc/config/system
```

If you're debugging a daemon that uses `syslog()`, start with `logread -f` to check whether its messages reach OpenWrt's local logging system.

## 4. Networking

OpenWrt uses UCI for persistent configuration and netifd for network configuration and management.

| Command | Description |
|---|---|
| `ip addr` | Show IP addresses |
| `ip link` | Show network interfaces and link state |
| `ip route` | Show IPv4 routes |
| `ip -6 route` | Show IPv6 routes |
| `ip neigh` | Show the neighbor table |
| `ifstatus wan` | Show the status of the WAN interface |
| `ifstatus lan` | Show the status of the LAN interface |
| `ubus call network.interface dump` | Show runtime network interface information |
| `uci show network` | Display network configuration |
| `cat /etc/config/network` | Read network configuration |
| `bridge link` | Show bridge port information |
| `bridge fdb show` | Show bridge forwarding database entries |
| `ping 8.8.8.8` | Test IP connectivity |
| `nslookup openwrt.org` | Test DNS resolution |
| `wget -O- https://openwrt.org` | Test HTTP connectivity |

Some commands, such as `bridge`, may require additional packages or a sufficiently recent iproute2 installation.

### Debug a network interface

```sh
ifstatus wan
ubus call network.interface dump
ip -s link
ip route get 8.8.8.8
```

These help check interface state, counters, and routing decisions.

### Monitor network configuration events

```sh
ubus listen network.interface
```

Listen for network interface events. This can help when investigating DHCP renewals, interface restarts, or connectivity changes.

## 5. UCI — Configuration Management

UCI (Unified Configuration Interface) is OpenWrt's configuration system.

| Command | Description |
|---|---|
| `uci show` | Display all UCI configuration |
| `uci show system` | Display system configuration |
| `uci show network` | Display network configuration |
| `uci show firewall` | Display firewall configuration |
| `uci get system.@system[0].hostname` | Read the configured hostname |
| `uci changes` | Show pending, uncommitted configuration changes |
| `uci commit system` | Save pending system configuration changes |
| `uci revert system` | Discard pending changes to the system configuration |
| `cat /etc/config/network` | Read the network configuration file |
| `uci export network` | Export network configuration in UCI syntax |

### Example: Change the hostname

Read the current hostname:

```sh
uci get system.@system[0].hostname
```

Set a new hostname:

```sh
uci set system.@system[0].hostname='my-router'
```

Save the configuration:

```sh
uci commit system
```

Apply it to the running system, if needed:

```sh
/etc/init.d/system reload
```

Whether a reload is necessary depends on the setting and the service that consumes it.

**Important:** `uci set` generally changes the in-memory UCI configuration state, while `uci commit` persists the change to the configuration file. Some changes require a service reload or restart before taking effect.

## 6. Package Management

OpenWrt generally uses `opkg` or, on newer releases and certain builds, `apk`.

| Command | Description |
|---|---|
| `opkg list-installed` | List installed packages on opkg-based systems |
| `opkg list` | List packages known to opkg |
| `opkg info package_name` | Show package information |
| `opkg files package_name` | List files installed by a package |
| `opkg status package_name` | Show installed package status |
| `opkg update` | Update package repository indexes |
| `opkg install package_name` | Install a package |
| `opkg remove package_name` | Remove a package |
| `apk info` | List installed packages on apk-based systems |
| `apk info -a package_name` | Show package information on apk-based systems |

The available package manager depends on the OpenWrt version and build configuration.

### Find which package owns a file

On an opkg-based system, try:

```sh
opkg search /usr/bin/example
```

This can help identify which package installed a particular executable.

## 7. Filesystems, Storage, and Memory

| Command | Description |
|---|---|
| `df -h` | Show filesystem capacity and free space |
| `df -i` | Show inode usage |
| `mount` | Show mounted filesystems |
| `cat /proc/mounts` | Show the kernel's mounted filesystem table |
| `lsblk` | List block devices, if installed |
| `block info` | Show block device information |
| `cat /proc/mtd` | List MTD flash partitions |
| `cat /proc/partitions` | List kernel-recognized partitions |
| `free -h` | Show memory usage, if supported |
| `cat /proc/meminfo` | Show detailed memory statistics |
| `du -h -d 1 /overlay` | Show directory disk usage, if supported |
| `df -h /tmp` | Identify the filesystem backing `/tmp` |
| `cat /proc/swaps` | Show active swap areas |

### Understand OpenWrt storage

Run:

```sh
df -h
mount
```

These help distinguish the read-only firmware filesystem (`/rom`), writable overlay storage, and separate partitions.

For Celerway devices with `/overlay`, `/rom`, `/opt`, and `/tmp/cwy_shared`, these commands are useful when investigating where files are stored and which partition is running out of space.

Check the overlay:

```sh
df -h /overlay
```

Find large directories:

```sh
du -h -d 1 /overlay 2>/dev/null
```

The `-d` option is not available in every BusyBox build. Check `du --help` or use an available alternative if necessary.

## 8. Processes, CPU, and Memory

| Command | Description |
|---|---|
| `ps` | List running processes |
| `top` | Monitor processes and system load |
| `uptime` | Show uptime and load averages |
| `free` | Show memory usage, if supported |
| `cat /proc/cpuinfo` | Show CPU information |
| `cat /proc/meminfo` | Show memory information |
| `cat /proc/loadavg` | Read load averages |
| `cat /proc/uptime` | Read system uptime |
| `pidof daemon_name` | Find a daemon's PID, if available |
| `kill PID` | Send SIGTERM by default |
| `kill -9 PID` | Send SIGKILL |

### Inspect a daemon

```sh
ps w
ps w | grep my_daemon
```

Inspect its executable and command line:

```sh
readlink /proc/PID/exe
cat /proc/PID/cmdline
```

Replace `PID` with the actual process ID. `/proc/PID/cmdline` contains NUL-separated arguments, so its output may not be formatted as a normal command.

### Inspect resource usage

```sh
cat /proc/meminfo
cat /proc/loadavg
top
```

These are useful when investigating a daemon that consumes too much memory or CPU.

## 9. Firewall and Connections

OpenWrt commonly uses firewall4 (`fw4`) with nftables on modern installations. Older systems may use firewall3 and iptables.

| Command | Description |
|---|---|
| `uci show firewall` | Show configured firewall rules |
| `fw4 print` | Print generated firewall configuration on firewall4 systems |
| `nft list ruleset` | Show the active nftables ruleset |
| `iptables -L -n -v` | List iptables rules on systems using iptables |
| `ss -tulnp` | Show listening TCP/UDP sockets, if supported |
| `netstat -tulnp` | Show listening sockets, if available |
| `conntrack -L` | Show connection-tracking entries, if installed |

### Check listening ports

```sh
ss -tulnp
```

For example, to look for a process listening on TCP port 514:

```sh
ss -tulnp | grep ':514'
```

If `ss` is unavailable, check whether `netstat` is installed.

### Inspect firewall rules

```sh
nft list ruleset
```

Use this on systems running nftables. Avoid modifying active firewall rules directly unless you understand how those changes interact with OpenWrt's firewall service.

## 10. Wireless

| Command | Description |
|---|---|
| `iw dev` | List wireless interfaces |
| `iwinfo` | Show wireless interface information, if installed |
| `iwinfo wlan0 info` | Show details for a wireless interface, if supported |
| `iwinfo wlan0 assoclist` | Show associated clients, if supported |
| `wifi status` | Show wireless status on builds that support it |
| `uci show wireless` | Show wireless configuration |
| `logread -f` | Follow logs while debugging Wi-Fi |

Wireless interface names vary. First check:

```sh
iw dev
```

Then use the appropriate interface name.

## 11. System Information and Hardware

| Command | Description |
|---|---|
| `ubus call system board` | Show OpenWrt device and firmware information |
| `ubus call system info` | Show uptime, memory, and load |
| `uname -a` | Show kernel and architecture information |
| `cat /etc/openwrt_release` | Show OpenWrt release metadata |
| `cat /etc/openwrt_version` | Show build version, if present |
| `cat /proc/cpuinfo` | Show CPU information |
| `cat /proc/mtd` | Show flash partition information |
| `dmesg` | Show kernel and hardware initialization messages |
| `ls -l /dev/` | Inspect device nodes |

### Quick system report

```sh
ubus call system board
ubus call system info
uname -a
df -h
free
```

This provides an overview of the device's firmware, kernel, memory, and storage. If `free` is unavailable, use `cat /proc/meminfo`.

## 12. Init Scripts and Boot Configuration

| Command | Description |
|---|---|
| `ls /etc/init.d/` | List service init scripts |
| `ls /etc/rc.d/` | List boot-time symlinks |
| `/etc/init.d/network restart` | Restart networking |
| `/etc/init.d/firewall restart` | Restart the firewall |
| `/etc/init.d/log restart` | Restart the logging service, if this script exists |
| `/etc/init.d/cron restart` | Restart cron |
| `/etc/init.d/your_service enable` | Enable a service at boot |
| `/etc/init.d/your_service disable` | Disable a service at boot |
| `reboot` | Reboot the device |
| `poweroff` | Shut down the device |

**Warning:** Restarting networking or rebooting a remotely accessed device can disconnect your SSH session. Make sure you have a recovery path first.

Exact init script names depend on installed packages and the OpenWrt release.

## 13. Developer and Troubleshooting Commands

Useful when developing C/C++ services and packages for OpenWrt.

| Command | Description |
|---|---|
| `command -v command_name` | Determine how the shell resolves a command |
| `type command_name` | Show whether a command is a builtin, alias, or executable |
| `ls -l /path/to/file` | Inspect file permissions and ownership |
| `stat /path/to/file` | Inspect file metadata, if available |
| `readlink -f /path/to/file` | Resolve a path, if supported |
| `find /etc -name '*name*'` | Search for files by name |
| `grep -R 'pattern' /etc` | Search configuration files recursively |
| `which strace` | Check whether `strace` is installed |
| `strace -p PID` | Trace system calls of a running process, if available |
| `lsof -p PID` | List files opened by a process, if `lsof` is installed |
| `cat /proc/PID/status` | Inspect process status |
| `cat /proc/PID/maps` | Inspect a process's memory mappings |

### Debug a daemon that exits unexpectedly

Inspect the logs:

```sh
logread -e my_daemon
```

Check whether the process is running:

```sh
ps w | grep my_daemon
```

Inspect its procd service definition:

```sh
ubus call service list '{"name":"my_daemon","verbose":true}'
```

Inspect its init script:

```sh
cat /etc/init.d/my_daemon
```

If available, trace the running process:

```sh
strace -f -p PID
```

This is a useful sequence for debugging service startup, configuration errors, and unexpected process termination.

## 14. Five Commands Worth Memorizing

### 1. Inspect services

```sh
ubus call service list
```

### 2. Inspect system information

```sh
ubus call system board
```

### 3. Follow system logs

```sh
logread -f
```

### 4. Inspect network interfaces

```sh
ubus call network.interface dump
```

### 5. Check storage

```sh
df -h
```

Together, these provide a quick overview of service state, firmware, logs, networking, and storage.

## 15. Quick OpenWrt Debugging Workflow

When investigating an issue on an OpenWrt device, start with this sequence:

```sh
# 1. Identify the device and firmware
ubus call system board

# 2. Check system resources
ubus call system info

# 3. Check service state
ubus call service list

# 4. Check running processes
ps w

# 5. Inspect recent logs
logread -l 100

# 6. Follow logs while reproducing the issue
logread -f

# 7. Check networking
ubus call network.interface dump

# 8. Check storage
df -h

# 9. Check mounted filesystems
mount

# 10. Check kernel messages
dmesg | tail -n 100
```

This sequence is a good starting point for most device-level investigations.
