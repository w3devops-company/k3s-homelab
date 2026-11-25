# Ubuntu Linux Cheatsheet & Debugging Guide

A quick reference for daily Ubuntu administration, troubleshooting, and `/var/log` system debugging.

---

## 1. System Info & Hardware

```bash
lsb_release -a                  # Ubuntu version info
uname -a                        # Kernel version & architecture
hostnamectl                     # Hostname & OS details
uptime                          # Uptime & load averages
lscpu                           # CPU info
free -h                         # Memory (RAM & swap) usage
lsblk                           # Block devices / disks / partitions
df -h                           # Disk space usage by filesystem
du -sh /path/to/dir             # Size of a directory
```

---

## 2. Package Management (`apt`, `dpkg`, `snap`)

```bash
sudo apt update                 # Refresh package index
sudo apt upgrade                # Upgrade installed packages
sudo apt install <pkg>          # Install a package
sudo apt remove <pkg>           # Remove package (keep config)
sudo apt purge <pkg>            # Remove package + config
sudo apt autoremove             # Remove unused dependencies
apt search <keyword>            # Search available packages
apt list --installed            # List installed packages

dpkg -i file.deb                # Install local .deb file
dpkg -S /path/to/file           # Find which package owns a file

snap list                       # List installed snaps
sudo snap install <pkg>         # Install a snap
```

---

## 3. Services & `journalctl` (`systemd`)

```bash
systemctl status <svc>          # Check service status
sudo systemctl start <svc>      # Start service
sudo systemctl stop <svc>       # Stop service
sudo systemctl restart <svc>    # Restart service
sudo systemctl reload <svc>     # Reload config without restart
sudo systemctl enable <svc>     # Start automatically on boot
systemctl --failed              # List failed services

journalctl -u <svc> -f          # Follow logs for a service
journalctl -p err -b            # Errors from current boot
journalctl -b -1                # Logs from previous boot (great after a crash)
journalctl --since "1h ago"     # Logs from the last hour
journalctl -k                   # Kernel messages (similar to dmesg / kern.log)
```

---

## 4. Important `/var/log` Files for Debugging

> **Note:** On modern/minimal Ubuntu installs, `rsyslog` may not be installed by default, so files like `/var/log/syslog` might not exist. In those cases, logs are stored in `/var/log/journal/` and queried via `journalctl`.

### Core System Logs

| Log File | Purpose | What to Debug |
| :--- | :--- | :--- |
| `/var/log/syslog` | General system & service activity | Cron jobs, daemon crashes, networking, general errors |
| `/var/log/kern.log` | Kernel messages | Hardware issues, driver errors, OOM killer events, USB/disk disconnects |
| `/var/log/auth.log` | Authentication & authorization | SSH logins, `sudo` usage, failed login attempts, PAM errors |
| `/var/log/boot.log` | System boot messages | Services failing during startup |
| `/var/log/dmesg` | Kernel ring buffer from boot | Early hardware detection and boot-time kernel errors |
| `/var/log/ufw.log` | Uncomplicated Firewall (if enabled) | Blocked/allowed network connections |

### Package & Update Logs

| Log File | Purpose |
| :--- | :--- |
| `/var/log/apt/history.log` | Summary of packages installed, upgraded, or removed via `apt` |
| `/var/log/apt/term.log` | Full terminal output from `apt` runs (useful for failed installs) |
| `/var/log/dpkg.log` | Low-level package state changes |
| `/var/log/unattended-upgrades/` | Automatic security update logs |

### Login & Session History (Binary Logs)

- `/var/log/wtmp` — Successful logins/reboots (read with `last`)
- `/var/log/btmp` — Failed login attempts (read with `sudo lastb`)
- `/var/log/lastlog` — Most recent login per user (read with `lastlog`)

### Common Service / App Subdirectories

- `/var/log/nginx/` or `/var/log/apache2/` — Web server access & error logs
- `/var/log/mysql/` or `/var/log/postgresql/` — Database logs
- `/var/log/cloud-init.log` & `/var/log/cloud-init-output.log` — Cloud/VM provisioning issues
- `/var/log/crash/` — Apport crash reports

### Quick Log Inspection Commands

```bash
sudo tail -f /var/log/syslog                # Watch syslog live
zgrep -i "error" /var/log/syslog*           # Search rotated .gz logs
dmesg -T | grep -i -E "killed process|oom"  # Check OOM killer events
```

---

## 5. Files, Navigation & Search

```bash
ls -lah                         # List files (long, hidden, human sizes)
cp -a src/ dst/                 # Copy preserving permissions/timestamps
mv old new                      # Move or rename
rm -rf dir/                     # Force-delete directory recursively
ln -s /target /link             # Create symlink

find /path -name "*.log"        # Find files by name
find /path -type f -size +100M  # Find files larger than 100MB
grep -rn "pattern" /path        # Recursive search with line numbers
tail -f /var/log/syslog         # Watch a file live
less +F file.log                # View file (Ctrl+C to scroll, F to follow)
```

---

## 6. Permissions & Ownership

```bash
chmod 755 script.sh             # rwxr-xr-x (owner: all, group/others: read+exec)
chmod 600 ~/.ssh/id_ed25519     # rw------- (owner only)
chmod +x script.sh              # Make executable
sudo chown user:group file      # Change owner and group
sudo chown -R user:group dir/   # Recursive ownership change
```

---

## 7. Processes & Performance

```bash
ps aux | grep <name>            # Find running processes
pgrep -af <name>                # Search processes by name/cmd
top                             # Live process monitor (or `htop`)
kill <PID>                      # Graceful terminate (SIGTERM)
kill -9 <PID>                   # Force kill (SIGKILL)
pkill -f <pattern>              # Kill processes matching pattern
lsof -i :8080                   # See what's using port 8080
```

---

## 8. Networking & Firewall (`ufw`)

```bash
ip a                            # Show IP addresses & interfaces
ip r                            # Show routing table
ss -tulpn                       # Show listening TCP/UDP ports & processes
ping -c 4 8.8.8.8               # Test connectivity
dig example.com                 # DNS lookup
curl -I https://example.com     # Fetch HTTP headers
scp file user@host:/path        # Copy file over SSH
rsync -avz src/ user@host:/dst/ # Efficient sync over SSH

sudo ufw status verbose         # Firewall status
sudo ufw allow 22/tcp           # Allow SSH
sudo ufw allow 80,443/tcp       # Allow HTTP/HTTPS
sudo ufw enable                 # Enable firewall
```

---

## 9. Users & Groups

```bash
whoami                          # Current user
id                              # UID, GID, and groups
sudo adduser <user>             # Create new user (interactive)
sudo usermod -aG sudo <user>    # Add user to sudo group
sudo passwd <user>              # Change user password
su - <user>                     # Switch user
```

---

## 10. Archives & Compression

```bash
tar -czvf archive.tar.gz dir/   # Create .tar.gz
tar -xzvf archive.tar.gz        # Extract .tar.gz
tar -tf archive.tar.gz          # List contents without extracting
zip -r archive.zip dir/         # Create zip
unzip archive.zip               # Extract zip
```

---

## 11. Key System Directories

- `/etc/` — System and service configuration files
- `/var/log/` — System and application logs
- `/home/` — User home directories
- `/tmp/` — Temporary files (cleared on reboot)
- `/opt/` — Third-party / manually installed software
- `/proc/` & `/sys/` — Virtual filesystems exposing kernel/hardware state

---

## 12. Production Debugging Scenarios

### Scenario 1: High CPU / Sluggish Server
**Symptoms:** Slow SSH, high load average, unresponsive APIs.
```bash
uptime                                          # Compare load average to CPU count (`nproc`)
top -b -n 1 | head -n 20                        # Snapshot of top CPU consumers
ps aux --sort=-%cpu | head -n 15                # Top 15 processes by CPU
pidstat 1 5                                     # Per-process CPU stats every 1s (sysstat pkg)
vmstat 1 5                                      # Check CPU wait (wa = disk I/O bottleneck)
```
> **Tip:** High load with low CPU usage usually means **I/O wait (`wa`)** or processes stuck in uninterruptible sleep (`D` state: `ps aux | awk '$8 ~ /D/'`).

### Scenario 2: Out of Memory (OOM) / Process Disappeared
**Symptoms:** App suddenly died or restarted without an application-level error log.
```bash
free -h                                         # Check available RAM and swap
dmesg -T | grep -i -E "oom|killed process"      # Confirm if Linux OOM killer terminated it
journalctl -k | grep -i "out of memory"         # Same check via journald
ps aux --sort=-%mem | head -n 15                # Top memory-consuming processes
```

### Scenario 3: Disk Full (`No space left on device`)
**Symptoms:** Writes fail, databases crash, logs stop updating.
```bash
df -h                                           # Check disk space usage
df -i                                           # Check inode usage (100% inodes = full even with free GBs)
sudo du -xh --max-depth=1 / | sort -rh | head   # Find largest top-level directories
sudo find /var/log -type f -size +100M -exec ls -lh {} +  # Find huge log files

# Classic production gotcha: file deleted, but process still holds it open
sudo lsof +L1                                   # List deleted files still held open by processes
# Fix: restart the process holding the file, or truncate instead of rm:
# sudo truncate -s 0 /var/log/huge.log
```

### Scenario 4: Systemd Service Fails to Start or Crash-Loops
**Symptoms:** `systemctl status` shows `failed` or `activating (auto-restart)`.
```bash
systemctl status <svc> --no-pager -l            # Check exit code and recent lines
journalctl -u <svc> -n 100 --no-pager           # Last 100 log lines of the service
journalctl -u <svc> --since "15 min ago"        # Logs from the failure window
systemd-analyze verify <svc>.service            # Validate unit file syntax
```

### Scenario 5: Service Unreachable / Port or Network Issue
**Symptoms:** `Connection refused`, timeouts, or `Address already in use`.
```bash
# 1. Is the service listening on the right interface (0.0.0.0 vs 127.0.0.1)?
sudo ss -tulpn | grep :<port>

# 2. Check if firewall is blocking it
sudo ufw status numbered
sudo iptables -L -n -v

# 3. Test locally vs remotely
curl -v http://127.0.0.1:<port>/health
nc -zv <target-host> <port>                     # Test TCP reachability

# 4. Check DNS resolution and live traffic
dig +short <domain>
sudo tcpdump -i any port <port> -nn -c 20       # Capture 20 packets on that port
```

### Scenario 6: SSH Login Failures / Brute-Force Activity
**Symptoms:** Users locked out, `Permission denied (publickey)`, or suspicious traffic.
```bash
sudo tail -n 50 /var/log/auth.log               # Recent SSH & sudo auth events
sudo grep "Failed password" /var/log/auth.log | tail -n 20
sudo lastb | head -n 20                         # Recent failed login attempts
sudo sshd -t                                    # Validate /etc/ssh/sshd_config syntax
```
> **Tip:** Public-key auth fails silently if permissions are too open. Ensure `~/.ssh` is `700` and `~/.ssh/authorized_keys` is `600`.

### Scenario 7: Unexpected Reboot or Kernel Panic
**Symptoms:** Server came back online on its own; uptime reset.
```bash
last reboot | head -n 10                        # History of reboots & crashes
who -b                                          # Exact last boot time
journalctl -b -1 -n 100 --no-pager              # Last 100 logs before the previous reboot
journalctl -b -1 -p err                         # Errors right before the reboot
```

