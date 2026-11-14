# Linux `systemctl` & `journalctl` Cheat Sheet and Guide

`systemd` is the central init system and service manager on modern Linux distributions (Ubuntu, Debian, Fedora, CentOS, RHEL, etc.). It manages:
- **`systemctl`**: Controlling services, system states, and unit files.
- **`journalctl`**: Inspecting, querying, and filtering system and service logs.

---

# Part 1: `systemctl` Cheat Sheet & Guide

`systemctl` is used to examine and control the state of the `systemd` system and service manager.

## 1. Quick Reference Table

| Purpose | Command |
|---|---|
| Check status of a service | `systemctl status <service>` |
| Start a service | `sudo systemctl start <service>` |
| Stop a service | `sudo systemctl stop <service>` |
| Restart a service | `sudo systemctl restart <service>` |
| Reload config without stopping service | `sudo systemctl reload <service>` |
| Restart if running, else reload | `sudo systemctl reload-or-restart <service>` |
| Enable service to start on boot | `sudo systemctl enable <service>` |
| Disable service from starting on boot | `sudo systemctl disable <service>` |
| Enable and start immediately in one go | `sudo systemctl enable --now <service>` |
| Disable and stop immediately in one go | `sudo systemctl disable --now <service>` |
| Completely prevent service from starting (manual or automatic) | `sudo systemctl mask <service>` |
| Unmask a masked service | `sudo systemctl unmask <service>` |
| Check if service is currently active | `systemctl is-active <service>` |
| Check if service is enabled on boot | `systemctl is-enabled <service>` |
| Check if service has failed | `systemctl is-failed <service>` |
| List all active running services | `systemctl list-units --type=service --state=running` |
| List all failed units | `systemctl --failed` |
| Reload systemd daemon after editing unit files | `sudo systemctl daemon-reload` |
| View unit file definition | `systemctl cat <service>` |
| Create drop-in override for a unit | `sudo systemctl edit <service>` |
| Reset failed state of a unit | `sudo systemctl reset-failed <service>` |

---

## 2. Managing Service Lifecycles

### Checking Status
```bash
# Detailed status with recent log entries
systemctl status k3s

# Script-friendly checks (returns exit code 0 if true, non-zero if false)
systemctl is-active k3s      # Prints "active" or "inactive"
systemctl is-enabled k3s     # Prints "enabled" or "disabled"
systemctl is-failed k3s      # Prints "failed" or "active"
```

### Starting, Stopping, and Restarting
```bash
# Start
sudo systemctl start k3s

# Stop
sudo systemctl stop k3s

# Full restart (stops and starts)
sudo systemctl restart k3s

# Reload config file without restarting the process
sudo systemctl reload nginx
```

### Enabling and Disabling on Boot
```bash
# Start on boot
sudo systemctl enable k3s

# Do not start on boot
sudo systemctl disable k3s

# Enable and start immediately
sudo systemctl enable --now k3s

# Disable and stop immediately
sudo systemctl disable --now k3s
```

### Masking Units
Masking links the unit file to `/dev/null`. This makes it impossible for any user or service to start it (even manually or as a dependency):
```bash
# Prevent service from ever starting
sudo systemctl mask apache2

# Allow service to be used again
sudo systemctl unmask apache2
```

---

## 3. Listing & Inspecting Units

### Listing Units
```bash
# List all active units
systemctl list-units

# List all services (active and inactive)
systemctl list-units --type=service --all

# List only running services
systemctl list-units --type=service --state=running

# List only failed units
systemctl --failed

# List all installed unit files and their enabled/disabled state
systemctl list-unit-files --type=service
```

### Inspecting Unit Properties
```bash
# View the full unit configuration file (including drop-in overrides)
systemctl cat k3s

# View low-level properties (key=value format)
systemctl show k3s

# View a specific property
systemctl show k3s -p MainPID
systemctl show k3s -p ActiveState
systemctl show k3s -p MemoryCurrent
```

---

## 4. Editing & Overriding Service Unit Files

Systemd stores unit files across several priority directories:
1. `/etc/systemd/system/` (Highest priority: local administrator configurations & overrides)
2. `/run/systemd/system/` (Runtime units created dynamically)
3. `/lib/systemd/system/` or `/usr/lib/systemd/system/` (Package-installed defaults; do not edit directly)

### Safe Editing with `systemctl edit`
Never directly edit files in `/lib/systemd/system/`—package updates will overwrite your changes. Instead, use:

```bash
# 1. Create a drop-in override snippet (recommended)
# This opens an editor and saves to /etc/systemd/system/<service>.d/override.conf
sudo systemctl edit k3s

# Example override: adjust environment or restart behavior
[Service]
Environment="K3S_NODE_NAME=custom-node"
RestartSec=10s
```

```bash
# 2. Edit full copy of the unit file in /etc/systemd/system/
sudo systemctl edit --full k3s
```

```bash
# 3. Always reload systemd after manual edits to unit files
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

---

## 5. System State & Power Management

```bash
# Reboot the machine
sudo systemctl reboot

# Power off / shut down
sudo systemctl poweroff

# Suspend system
sudo systemctl suspend

# View default boot target (e.g. multi-user.target, graphical.target)
systemctl get-default

# Change default boot target (e.g. boot to terminal without GUI)
sudo systemctl set-default multi-user.target

# Switch target immediately without rebooting
sudo systemctl isolate multi-user.target
```

---

# Part 2: `journalctl` Cheat Sheet & Guide

`journalctl` is used to query, search, and analyze logs collected by `systemd-journald`.

## 1. Quick Reference Table

| Purpose | Command |
|---|---|
| Follow logs live (like `tail -f`) | `journalctl -f` |
| View logs for a specific service | `journalctl -u <service>` |
| Follow service logs live | `journalctl -u <service> -f` |
| View last $N$ lines of a service | `journalctl -u <service> -n <N>` |
| View logs without pager (for piping/grep) | `journalctl -u <service> --no-pager` |
| View logs from the current boot only | `journalctl -b` |
| View logs from the previous boot | `journalctl -b -1` |
| Filter by priority/severity (e.g. errors) | `journalctl -u <service> -p err` |
| View kernel messages (like `dmesg`) | `journalctl -k` |
| Filter logs by relative time | `journalctl --since "15 minutes ago"` |
| Filter logs by exact time window | `journalctl --since "2026-11-13 10:00" --until "2026-11-13 11:30"` |
| Check journal disk usage | `journalctl --disk-usage` |
| Vacuum logs to limit disk usage | `sudo journalctl --vacuum-size=500M` |
| Vacuum logs older than a time period | `sudo journalctl --vacuum-time=7d` |

---

## 2. Filtering Logs

### By Service / Unit (`-u`)
```bash
# View k3s logs
journalctl -u k3s

# View Docker logs
journalctl -u docker

# Combine multiple services
journalctl -u k3s -u docker
```

### By Boot (`-b`)
```bash
# Current boot only
journalctl -b

# Previous boot
journalctl -b -1

# List all boot IDs
journalctl --list-boots
```

### By Time (`--since` / `--until`)
```bash
# Relative time offsets
journalctl -u k3s --since "10m ago"
journalctl -u k3s --since "2 hours ago"
journalctl --since "yesterday"
journalctl --since "today"

# Specific dates and timestamps
journalctl --since "2026-11-13 10:30:00" --until "2026-11-13 11:15:00"
```

### By Priority / Severity (`-p`)
Syslog log levels: `0=emerg`, `1=alert`, `2=crit`, `3=err`, `4=warning`, `5=notice`, `6=info`, `7=debug`.

```bash
# Show only errors, criticals, alerts, and emergencies
journalctl -u k3s -p err

# Show warnings and errors across the system
journalctl -p warning -b
```

### By Process ID, Binary, or User
```bash
journalctl _PID=12345
journalctl /usr/local/bin/k3s
journalctl _UID=1000
```

---

## 3. Formatting & Output Options

```bash
# Disable pager (crucial for grep / awk / scripts)
journalctl -u k3s -b --no-pager | grep -i "eviction"

# Open directly at end of logs
journalctl -u k3s -e

# Reverse chronological order (newest first)
journalctl -u k3s -r -n 50

# Explanatory details from message catalog
journalctl -xeu k3s

# ISO-8601 timestamps
journalctl -o short-iso

# Detailed view with all internal fields/metadata
journalctl -o verbose

# JSON format
journalctl -o json-pretty
```

---

## 4. Log Disk Maintenance & Vacuuming

```bash
# Check current disk space consumed by logs
journalctl --disk-usage

# Retain maximum 500 MB of logs
sudo journalctl --vacuum-size=500M

# Retain logs from the last 7 days only
sudo journalctl --vacuum-time=7d

# Retain only the last 2 boot sessions
sudo journalctl --vacuum-files=2
```

---

# Part 3: Real-World Troubleshooting Workflows

### 1. Kubernetes / K3s Service Debugging
```bash
# Live stream k3s logs
journalctl -u k3s -f

# Check k3s eviction thresholds and startup config
journalctl -u k3s -b | grep -A 10 "evictionHard:"

# Check for recent eviction manager events
journalctl -u k3s -n 100 --no-pager | grep -i "eviction manager"

# Check all k3s errors since boot
journalctl -u k3s -b -p err --no-pager

# Restart k3s cleanly and follow startup
sudo systemctl restart k3s && journalctl -u k3s -f
```

### 2. Diagnosing a Failed Service
```bash
# 1. Check current status and error snippet
systemctl status <service-name>

# 2. View full error explanation with catalog help
journalctl -xeu <service-name>

# 3. Clear failed status after resolving root cause
sudo systemctl reset-failed <service-name>
```

### 3. System Crash / Kernel Panic
```bash
# Check kernel messages from previous boot
journalctl -k -b -1
```
