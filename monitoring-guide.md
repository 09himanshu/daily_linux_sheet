# 📊 Linux System Monitoring and Log Monitoring Guide

A detailed, daily-use reference for monitoring server health and analysing logs: commands, log locations, tools, alerting, and a troubleshooting playbook.

---

## 📋 Table of Contents
- [Overview](#-overview)
- [Key Metrics and Healthy Ranges](#-key-metrics-and-healthy-ranges)
- [System Monitoring Commands](#-system-monitoring-commands)
- [Log Monitoring](#-log-monitoring)
- [Monitoring Tools Stack](#-monitoring-tools-stack)
- [Prometheus and Grafana Basics](#-prometheus-and-grafana-basics)
- [Centralized Logging](#-centralized-logging)
- [Kubernetes and Docker](#-kubernetes-and-docker)
- [Alerting Best Practices](#-alerting-best-practices)
- [Troubleshooting Playbook](#-troubleshooting-playbook)
- [Quick Reference](#-quick-reference)

---

## 🧭 Overview

| Term | Meaning | Answers |
|---|---|---|
| **Monitoring** | Collecting numbers (metrics) over time and alerting on them | "Is something wrong?" |
| **Logging** | Recording events as text lines | "What exactly happened?" |
| **Tracing** | Following one request across services | "Where is it slow?" |
| **Observability** | Metrics + Logs + Traces together | "Why is it wrong?" |

**Typical flow:**
```
Metric alert fires (CPU high)  →  Check dashboard  →  Dig into logs  →  Find root cause  →  Fix
```

**Two useful frameworks:**
- **USE method** (for resources like CPU, memory, disk): **U**tilization, **S**aturation, **E**rrors
- **Four Golden Signals** (for services, from Google SRE): **Latency, Traffic, Errors, Saturation**
- **RED method** (for request-driven services): **R**ate, **E**rrors, **D**uration

---

## 📏 Key Metrics and Healthy Ranges

> These are rules of thumb, not absolute values. Always compare against your own baseline.

| Resource | What to watch | Warning sign |
|---|---|---|
| **CPU** | Utilization %, load average, `iowait`, `steal` | Sustained > 80-85%, load average higher than CPU count |
| **Memory** | `available` memory, swap usage, OOM kills | Available very low, swap being actively used, OOM events |
| **Disk space** | Used %, inode usage | > 80% warning, > 90% critical |
| **Disk I/O** | `%util`, await (latency), queue size | `%util` near 100%, high await |
| **Network** | Bandwidth, packet errors/drops, connections | Errors or drops > 0, connection spikes |
| **Processes** | Zombie count, runaway processes | Many zombies, one process hogging CPU/memory |
| **Service** | Up/down, response time, error rate | Port not listening, 5xx rising, latency spike |

### Understanding Load Average
```bash
uptime
# 10:30:01 up 12 days, load average: 1.20, 0.90, 0.75
#                                     │     │     └── last 15 min
#                                     │     └──────── last 5 min
#                                     └────────────── last 1 min
nproc    # number of CPU cores
```
- Load average **below core count** is healthy. Above core count means processes are waiting.
- Example: 4 cores, load `4.0` = fully busy. Load `8.0` = overloaded (queueing).
- High load with **low CPU usage** usually points to **disk I/O wait** (check `iowait`).

---

## 🖥️ System Monitoring Commands

### Quick Health Snapshot
```bash
uptime                 # uptime + load average
w                      # who is logged in + load
free -h                # memory summary
df -h                  # disk space
top -bn1 | head -15    # one-shot top (good for scripts)
```

### CPU
```bash
top                    # live process view
htop                   # nicer top (install: apt install htop)
mpstat -P ALL 1        # per-core CPU usage every 1 sec (sysstat)
vmstat 1 5             # CPU, memory, I/O, run queue (1 sec interval, 5 times)
pidstat 1              # per-process CPU usage (sysstat)
ps aux --sort=-%cpu | head -10     # top 10 CPU consuming processes
```

**Useful `top` keys:**

| Key | Action |
|---|---|
| `1` | Show per-CPU core usage |
| `M` | Sort by memory |
| `P` | Sort by CPU |
| `k` | Kill a process (asks PID) |
| `c` | Toggle full command line |
| `q` | Quit |

**Reading the CPU line in `top`:**

| Field | Meaning |
|---|---|
| `us` | User processes |
| `sy` | Kernel/system |
| `id` | Idle |
| `wa` | **I/O wait** (CPU waiting for disk) |
| `st` | **Steal** (VM stolen by hypervisor; relevant on cloud) |

### Memory
```bash
free -h                            # total/used/free/available
vmstat -s                          # detailed memory stats
cat /proc/meminfo                  # raw memory info
ps aux --sort=-%mem | head -10     # top 10 memory consuming processes
smem -r | head                     # per-process memory (PSS), if installed
swapon --show                      # swap usage
```
- Look at the **`available`** column in `free -h`, not `free`. Linux uses spare memory for cache, which is normal.
- If `si`/`so` (swap in/out) in `vmstat` are constantly non-zero, the system is **swapping** (memory pressure).

**Check for OOM (Out Of Memory) kills:**
```bash
dmesg -T | grep -i -E "out of memory|oom|killed process"
journalctl -k | grep -i oom
```

### Disk Space
```bash
df -h                              # filesystem usage
df -i                              # inode usage (can be full even when space is free)
du -sh /var/log                    # size of a directory
du -sh /* 2>/dev/null | sort -rh | head      # biggest top-level directories
du -ah /var 2>/dev/null | sort -rh | head -20   # biggest files/dirs under /var
ncdu /                             # interactive disk usage (apt install ncdu)
lsblk                              # block devices layout
```

**Disk full but you deleted files?** A process may still hold the deleted file open:
```bash
sudo lsof +L1                      # files deleted but still open
```
Restart that process to release the space.

### Disk I/O
```bash
iostat -xz 1                       # extended disk stats every 1 sec (sysstat)
iotop -o                           # processes doing I/O right now (needs root)
vmstat 1                           # check 'wa' column (I/O wait) and 'bi/bo'
```

**Key `iostat -x` columns:**

| Column | Meaning |
|---|---|
| `%util` | How busy the device is (near 100% = saturated) |
| `await` | Average time per I/O request in ms (high = slow disk) |
| `r/s`, `w/s` | Reads/writes per second |

### Network
```bash
ss -tulnp                          # listening ports + owning process
ss -s                              # socket summary
ss -tan state established | wc -l  # count established connections
ip -s link                         # interface stats (errors, drops)
ip addr                            # IP addresses
iftop                              # live bandwidth per connection
nload                              # live bandwidth per interface
sar -n DEV 1 5                     # network throughput history/live (sysstat)
ping -c 4 host                     # reachability + latency
traceroute host                    # path to host
mtr host                           # ping + traceroute combined
curl -o /dev/null -s -w "%{http_code} %{time_total}s\n" http://localhost:8080/
```

### Processes
```bash
ps aux                             # all processes
ps -ef --forest                    # process tree
ps -p <PID> -o pid,user,%cpu,%mem,etime,cmd    # details of one process
pgrep -a nginx                     # find processes by name
pstree -p                          # tree view with PIDs
lsof -p <PID>                      # files opened by a process
lsof -i :3000                      # who is using port 3000
ps aux | awk '$8 ~ /Z/'            # zombie processes
```

### Historical Data with `sar` (sysstat)
```bash
sudo apt install sysstat           # install, then enable data collection
sar -u 1 5                         # CPU, live
sar -r 1 5                         # memory, live
sar -q 1 5                         # load average + run queue
sar -d 1 5                         # disk
sar -n DEV 1 5                     # network
sar -u -f /var/log/sysstat/sa24    # CPU history for the 24th (Debian/Ubuntu path)
```

### Services (systemd)
```bash
systemctl status nginx             # service state + recent logs
systemctl is-active nginx          # active / inactive
systemctl --failed                 # list all failed services
systemctl list-units --type=service --state=running
systemd-analyze blame | head       # what slowed down boot
```

### Kernel and Hardware Messages
```bash
dmesg -T | tail -50                # kernel ring buffer with readable timestamps
dmesg -T -l err,crit,alert,emerg   # only errors and above
lscpu                              # CPU info
lsmem                              # memory info
```

---

## 📜 Log Monitoring

### Common Log Locations

| Path | Contents |
|---|---|
| `/var/log/syslog` (Debian/Ubuntu) | General system messages |
| `/var/log/messages` (RHEL/CentOS) | General system messages |
| `/var/log/auth.log` (Debian/Ubuntu) | Logins, sudo, SSH |
| `/var/log/secure` (RHEL/CentOS) | Logins, sudo, SSH |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dmesg` | Boot-time kernel messages |
| `/var/log/boot.log` | Boot process messages |
| `/var/log/dpkg.log`, `/var/log/apt/` | Package install/remove history (Debian/Ubuntu) |
| `/var/log/yum.log`, `/var/log/dnf.log` | Package history (RHEL family) |
| `/var/log/cron` or syslog | Cron job activity |
| `/var/log/nginx/access.log`, `error.log` | Nginx logs |
| `/var/log/apache2/` or `/var/log/httpd/` | Apache logs |
| `/var/log/mysql/`, `/var/log/postgresql/` | Database logs |
| `/var/log/wtmp`, `/var/log/btmp` | Login history / failed logins (binary, read with `last`, `lastb`) |

### Syslog Severity Levels

| Level | Name | Meaning |
|---|---|---|
| 0 | `emerg` | System unusable |
| 1 | `alert` | Immediate action needed |
| 2 | `crit` | Critical condition |
| 3 | `err` | Error |
| 4 | `warning` | Warning |
| 5 | `notice` | Normal but significant |
| 6 | `info` | Informational |
| 7 | `debug` | Debug messages |

### Reading Logs: Basics
```bash
tail -n 100 file.log               # last 100 lines
tail -f file.log                   # follow live
tail -F file.log                   # follow, and survive log rotation (preferred)
head -n 50 file.log                # first 50 lines
less file.log                      # scroll (/word to search, G end, g start, q quit)
cat file.log | wc -l               # count lines
zcat file.log.2.gz | less          # read compressed rotated log
zgrep "ERROR" /var/log/app/*.gz    # search compressed logs
```

### Searching Logs with grep
```bash
grep "ERROR" app.log                       # lines containing ERROR
grep -i "error" app.log                    # case-insensitive
grep -v "DEBUG" app.log                    # exclude DEBUG lines
grep -E "ERROR|FATAL|CRITICAL" app.log     # multiple patterns
grep -C 3 "Exception" app.log              # 3 lines of context around match
grep -A 5 "Exception" app.log              # 5 lines after match
grep -B 5 "Exception" app.log              # 5 lines before match
grep -c "ERROR" app.log                    # count matching lines
grep -rn "timeout" /var/log/myapp/         # recursive with line numbers
tail -f app.log | grep --line-buffered "ERROR"    # live filtered stream
```

### Time-Based Filtering
```bash
grep "2026-10-24 10:" app.log                             # all lines from the 10 o'clock hour
sed -n '/2026-10-24 10:00/,/2026-10-24 11:00/p' app.log   # time range
awk '$2 >= "10:00:00" && $2 <= "10:30:00"' app.log        # range using the time column
```

### journalctl (systemd logs)
```bash
journalctl                               # everything (oldest first)
journalctl -n 100                        # last 100 entries
journalctl -f                            # follow live
journalctl -u nginx                      # logs of one service
journalctl -u nginx -f                   # follow one service
journalctl -u nginx --since "1 hour ago"
journalctl --since "2026-10-24 10:00" --until "2026-10-24 11:00"
journalctl -p err                        # errors and above
journalctl -p warning..alert             # severity range
journalctl -b                            # current boot
journalctl -b -1                         # previous boot (great after a crash/reboot)
journalctl -k                            # kernel messages
journalctl -o json-pretty -n 5           # JSON output
journalctl _PID=1234                     # logs from a specific PID
journalctl --disk-usage                  # space used by journal
sudo journalctl --vacuum-time=7d         # keep only last 7 days
sudo journalctl --vacuum-size=500M       # cap total size
```

### Security-Related Logs
```bash
last                                     # successful login history
lastb                                    # failed login attempts (needs root)
lastlog                                  # last login per user
who -a                                   # current sessions
grep "Failed password" /var/log/auth.log | tail            # failed SSH logins
grep "Accepted" /var/log/auth.log | tail                   # successful SSH logins
grep "sudo" /var/log/auth.log | tail                       # sudo usage
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head
                                                           # top IPs failing SSH login
```
(On RHEL/CentOS use `/var/log/secure` instead of `auth.log`.)

### Log Analysis One-Liners (Web Server Access Logs)
> Assumes the default combined log format (client IP is field 1, status code is field 9).

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head       # top client IPs
awk '{print $9}' access.log | sort | uniq -c | sort -nr              # count by HTTP status
awk '$9 ~ /^5/' access.log | tail                                    # recent 5xx errors
awk '$9 == 404 {print $7}' access.log | sort | uniq -c | sort -nr | head   # top 404 URLs
awk '{print $7}' access.log | sort | uniq -c | sort -nr | head       # most requested URLs
awk '{print $4}' access.log | cut -d: -f1-2 | sort | uniq -c         # requests per hour
grep -c "POST" access.log                                            # count POST requests
```

### Useful Log Tools
```bash
lnav /var/log/syslog         # powerful log viewer with filtering/SQL (apt install lnav)
multitail a.log b.log        # watch several logs at once
```

### Log Rotation (logrotate)
Without rotation, logs fill the disk.

```bash
/etc/logrotate.conf          # main config
/etc/logrotate.d/            # per-application configs
```

**Sample config (`/etc/logrotate.d/myapp`):**
```conf
/var/log/myapp/*.log {
    daily                 # rotate every day
    rotate 14             # keep 14 old logs
    compress              # gzip old logs
    delaycompress         # compress from the 2nd rotation onwards
    missingok             # no error if log is missing
    notifempty            # skip if empty
    copytruncate          # truncate in place (for apps that can't reopen logs)
}
```

```bash
sudo logrotate -d /etc/logrotate.conf    # dry run (debug, no changes)
sudo logrotate -f /etc/logrotate.conf    # force rotation now
```

### Central Syslog Config (rsyslog)
```bash
/etc/rsyslog.conf            # main config
/etc/rsyslog.d/              # drop-in configs
sudo systemctl restart rsyslog
logger "test message from CLI"     # write a test line into syslog
```

---

## 🧰 Monitoring Tools Stack

| Category | Tools | Notes |
|---|---|---|
| **Quick CLI** | `top`, `htop`, `vmstat`, `iostat`, `sar`, `glances`, `btop` | No setup, good for live debugging |
| **Metrics + Dashboards** | **Prometheus + Grafana** | De facto standard in cloud-native / Kubernetes |
| **Host metrics agent** | **node_exporter** (Prometheus), Telegraf, Netdata | Exposes CPU, memory, disk, network |
| **Alerting** | **Alertmanager**, Grafana Alerting, PagerDuty, Opsgenie | Routes alerts to Slack, email, on-call |
| **Centralized logs** | **ELK/EFK** (Elasticsearch, Logstash/Fluentd, Kibana), **Grafana Loki** | Search logs from all servers in one place |
| **Log shippers** | Filebeat, Fluent Bit, Fluentd, Promtail, Vector | Collect and forward logs |
| **Traditional monitoring** | Nagios, Zabbix, Icinga | Agent/check-based, still widely used |
| **Cloud native** | AWS CloudWatch, Azure Monitor, GCP Cloud Monitoring | Built into the cloud provider |
| **Tracing** | Jaeger, Tempo, OpenTelemetry, AWS X-Ray | Follow requests across services |
| **Uptime checks** | Blackbox exporter, UptimeRobot, Pingdom | External availability probes |

**Typical modern setup:**
```
Servers/Pods ──► node_exporter / app metrics ──► Prometheus ──► Grafana (dashboards)
                                                      │
                                                      └──► Alertmanager ──► Slack / Email / PagerDuty

Servers/Pods ──► Fluent Bit / Promtail ──► Loki (or Elasticsearch) ──► Grafana / Kibana
```

---

## 📈 Prometheus and Grafana Basics

### Default Ports

| Component | Port |
|---|---|
| Prometheus | `9090` |
| node_exporter | `9100` |
| Alertmanager | `9093` |
| Grafana | `3000` |
| Loki | `3100` |

### Install node_exporter (quick idea)
```bash
# download the release tarball from the Prometheus GitHub releases page, then:
tar xvf node_exporter-*.tar.gz
./node_exporter &              # runs on :9100
curl localhost:9100/metrics | head
```
(In production, run it as a systemd service under a dedicated user.)

### Prometheus Scrape Config (`prometheus.yml`)
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["server1:9100", "server2:9100"]
```

### Common PromQL Queries
```promql
# CPU usage % per instance
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage % per filesystem
(1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100

# Network received bytes/sec
rate(node_network_receive_bytes_total[5m])

# Is the target up? (1 = up, 0 = down)
up
```

### Sample Alert Rule
```yaml
groups:
  - name: node-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU above 85% for 10 minutes."

      - alert: DiskAlmostFull
        expr: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk almost full on {{ $labels.instance }}"

      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} is down"
```
The `for:` clause means the condition must hold for that long before alerting, which avoids noise from short spikes.

---

## 🗄️ Centralized Logging

**Why:** On many servers/pods, SSH-ing into each one to read logs doesn't scale, and logs disappear when containers die.

### Loki + Promtail (lightweight)
```
App logs ──► Promtail ──► Loki ──► Grafana (Explore)
```
Common **LogQL** queries:
```logql
{app="nginx"}                              # all nginx logs
{app="nginx"} |= "error"                   # lines containing "error"
{app="nginx"} != "healthcheck"             # exclude healthchecks
{app="nginx"} |~ "5[0-9]{2}"               # regex match
rate({app="nginx"} |= "error" [5m])        # error rate per second
```

### ELK / EFK (full-text search, heavier)
```
App logs ──► Filebeat / Fluent Bit ──► Logstash (optional) ──► Elasticsearch ──► Kibana
```
Good for powerful search, aggregations, and large-scale log analytics.

### Good Logging Practices
- Use **structured logs** (JSON) so fields are searchable.
- Always include **timestamp (UTC), level, service name, request/trace ID**.
- Never log passwords, tokens, or personal data.
- Use proper levels (`INFO` vs `ERROR`); too much `DEBUG` in production creates noise and cost.
- Set **retention** policies (for example 14 to 30 days hot, then archive).

---

## ☸️ Kubernetes and Docker

### Docker
```bash
docker ps                                  # running containers
docker logs -f --tail 100 <container>      # follow last 100 lines
docker logs --since 1h <container>         # last hour
docker stats                               # live CPU/memory per container
docker top <container>                     # processes inside a container
docker inspect <container>                 # full details
docker system df                           # disk used by images/containers/volumes
```

### Kubernetes Logs
```bash
kubectl logs <pod>                         # pod logs
kubectl logs -f <pod>                      # follow
kubectl logs <pod> -c <container>          # specific container in a pod
kubectl logs <pod> --previous              # logs of the crashed/previous container (CrashLoopBackOff)
kubectl logs <pod> --since=1h --tail=200
kubectl logs -l app=myapp --all-containers=true     # by label, all containers
kubectl logs -n <namespace> <pod>
```

### Kubernetes Health and Resource Usage
```bash
kubectl top nodes                          # node CPU/memory (needs metrics-server)
kubectl top pods -A --sort-by=memory       # heaviest pods
kubectl get pods -A -o wide                # pods, nodes, status
kubectl get pods -A | grep -v Running      # pods not in Running state
kubectl describe pod <pod>                 # events, probes, restarts, OOMKilled, etc.
kubectl get events -A --sort-by=.lastTimestamp     # recent cluster events
kubectl get nodes                          # node readiness
kubectl describe node <node>               # capacity, allocated resources, conditions
```

### Common Pod Problems

| Status | Likely cause | First check |
|---|---|---|
| `CrashLoopBackOff` | App keeps crashing | `kubectl logs --previous`, `describe pod` |
| `OOMKilled` | Memory limit exceeded | `describe pod` (Last State), raise limits or fix leak |
| `ImagePullBackOff` | Wrong image/tag or registry auth | `describe pod` events, image name, pull secret |
| `Pending` | No resources / scheduling issue | `describe pod` events, `kubectl describe node` |
| `Evicted` | Node under pressure (disk/memory) | `describe node`, node conditions |
| Readiness probe failing | App not ready or wrong probe | `describe pod`, probe path/port |

---

## 🚨 Alerting Best Practices

- **Alert on symptoms, not causes.** "Users see errors/latency" matters more than "CPU is 70%".
- **Every alert must be actionable.** If nobody needs to do anything, it should not page.
- Use **severity levels**:
  - `critical` = page someone now (service down, data loss risk)
  - `warning` = look during working hours (disk 80%, rising errors)
  - `info` = dashboard only
- Use `for:` durations and thresholds to **avoid flapping/noisy alerts**.
- Alert on **trends** for slow-burn issues (disk filling in 4 hours: `predict_linear`).
- Attach a **runbook link** in each alert: what it means, how to check, how to fix.
- **Review alerts regularly.** Delete or tune noisy ones (alert fatigue is a real reliability risk).
- Monitor the monitoring: have a **dead man's switch** alert that fires when the alerting pipeline itself stops working.

**What to always monitor on a server:**
- [ ] CPU, memory, swap
- [ ] Disk space and inodes
- [ ] Disk I/O latency
- [ ] Network errors and bandwidth
- [ ] Critical services up/down (systemd units, ports)
- [ ] Application error rate and latency
- [ ] Certificate expiry
- [ ] Backup/snapshot job success
- [ ] Log volume spikes and error spikes

---

## 🛠️ Troubleshooting Playbook

### 1. Server is slow
```bash
uptime                           # load average vs nproc
top                              # who is using CPU/memory? check wa (I/O wait)
vmstat 1 5                       # r (run queue), b (blocked), si/so (swap), wa
iostat -xz 1                     # is disk saturated? (%util, await)
free -h                          # memory pressure / swap
dmesg -T | tail -30              # kernel errors (OOM, disk errors)
```
- High `us/sy` = CPU-bound process → find with `ps aux --sort=-%cpu | head`
- High `wa` = disk bottleneck → `iotop -o`, `iostat -x`
- High swap in/out = memory shortage → find big processes, add RAM or fix leak

### 2. Disk is full
```bash
df -h                                        # which filesystem?
df -i                                        # or are inodes exhausted?
du -sh /* 2>/dev/null | sort -rh | head      # drill down directory by directory
sudo lsof +L1                                # deleted-but-open files holding space
sudo journalctl --vacuum-time=7d             # shrink the systemd journal
sudo apt clean                               # clear package cache (Debian/Ubuntu)
```
Common culprits: `/var/log`, Docker images/volumes (`docker system df`), old backups, core dumps.

### 3. Service is down
```bash
systemctl status <service>                   # state + last log lines
journalctl -u <service> -n 100 --no-pager    # recent logs
ss -tulnp | grep <port>                      # is it listening?
sudo systemctl restart <service>
journalctl -u <service> -f                   # watch it start
```

### 4. Process got killed unexpectedly (suspect OOM)
```bash
dmesg -T | grep -i -E "killed process|out of memory"
journalctl -k | grep -i oom
free -h
```

### 5. Port already in use
```bash
sudo ss -tulnp | grep :3000
sudo lsof -i :3000
sudo fuser -k 3000/tcp                       # kill whatever holds the port
```

### 6. Suspicious logins / possible brute force
```bash
sudo lastb | head                            # failed logins
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head
last | head                                 # recent successful logins
who -a                                       # who is on right now
```

### 7. High error rate on a web app
```bash
tail -f /var/log/nginx/error.log
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr   # status code spread
awk '$9 ~ /^5/' /var/log/nginx/access.log | tail -20                    # recent 5xx
journalctl -u myapp --since "30 min ago" -p err
```

---

## ⚡ Quick Reference

| Goal | Command |
|---|---|
| Load average | `uptime` |
| Live processes | `top` / `htop` |
| Memory | `free -h` |
| Disk space | `df -h` |
| Biggest directories | `du -sh /* \| sort -rh \| head` |
| Disk I/O | `iostat -xz 1` |
| CPU per core | `mpstat -P ALL 1` |
| CPU/memory/IO overview | `vmstat 1 5` |
| Listening ports | `ss -tulnp` |
| Process on a port | `sudo lsof -i :PORT` |
| Failed services | `systemctl --failed` |
| Kernel errors | `dmesg -T -l err,crit,alert,emerg` |
| Service logs live | `journalctl -u <svc> -f` |
| Errors in last hour | `journalctl -p err --since "1 hour ago"` |
| Follow a log file | `tail -F file.log` |
| Search a log | `grep -i "error" file.log` |
| Failed SSH logins | `sudo lastb \| head` |
| OOM events | `dmesg -T \| grep -i oom` |
| Pod logs | `kubectl logs -f <pod>` |
| Crashed pod logs | `kubectl logs <pod> --previous` |
| Pod/node resource usage | `kubectl top pods` / `kubectl top nodes` |
| Container resource usage | `docker stats` |

---

*💡 Tip: Learn the CLI tools first (they work on any server, even when your monitoring stack is down), then add Prometheus + Grafana + Loki for continuous visibility and alerting.*
