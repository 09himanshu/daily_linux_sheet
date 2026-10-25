# sos report: Troubleshooting Guide

`sos report` ek diagnostic collection tool hai (RHEL, CentOS, Rocky, Fedora, Ubuntu). Ye node ki config, logs aur command outputs ek tarball me collect karta hai, taaki issue ko offline analyze kiya ja sake ya vendor support ko bheja ja sake.

---

## 1. Design (kaise kaam karta hai)

- **Plugin-based**: har component ka apna plugin (kernel, networking, systemd, kubernetes, docker, storage, etc.)
- Run hone pe har enabled plugin:
  - relevant **files** copy karta hai (`/etc/...`, `/var/log/...`)
  - relevant **commands** chalata hai (`ip a`, `lsblk`, `systemctl status`) aur output save karta hai
- Sensitive data (passwords, tokens) by default scrub hota hai
- Output: ek `.tar.xz` archive + `.sha256` checksum, default location `/var/tmp/`

```
sos report
   |
   +-- plugins (kernel, networking, systemd, ...)
   |       |-- copy files
   |       |-- run commands
   |
   +-- scrub secrets
   |
   +-- /var/tmp/sosreport-<host>-<date>-<id>.tar.xz (+ .sha256)
```

---

## 2. Install

```bash
# Ubuntu / Debian
sudo apt install sosreport
```

Version check:

```bash
sos --version
```

> Purane versions me command `sosreport` tha. Naye versions (4.x+) me `sos report` hai. `sosreport` alias abhi bhi chalta hai.

---

## 3. Common usage

```bash
# Full report (interactive prompts ke saath)
sudo sos report

# Batch mode (bina prompts ke, automation ke liye)
sudo sos report --batch

# Case ID ke saath (vendor support ke liye)
sudo sos report --batch --case-id 12345

# Sirf specific plugins
sudo sos report --batch -o systemd,networking,kubernetes

# Kuch plugins skip karna
sudo sos report --batch -n docker,sudo

# Plugin list dekhna
sudo sos report --list-plugins

# Profiles dekhna aur use karna (plugins ka group)
sudo sos report --list-profiles
sudo sos report --batch -p network

# Sirf recent logs (log size kam rakhne ke liye)
sudo sos report --batch --since 20261020

# Saare logs (rotated bhi)
sudo sos report --batch --all-logs

# Alag tmp directory (agar /var/tmp me space kam hai)
sudo sos report --batch --tmp-dir /data/tmp

# Plugin option enable karna
sudo sos report --batch -k kubernetes.all=on
```

### Useful flags (quick table)

| Flag | Kaam |
|------|------|
| `--batch` | Prompts skip |
| `-o <plugins>` | Sirf ye plugins chalao |
| `-n <plugins>` | Ye plugins skip karo |
| `-e <plugins>` | Off-by-default plugins enable karo |
| `-p <profile>` | Profile ke plugins chalao |
| `-k plugin.opt=val` | Plugin option set karo |
| `--case-id` | Case number archive name me add |
| `--since` | Sirf is date ke baad ke logs |
| `--all-logs` | Rotated logs bhi |
| `--tmp-dir` | Working/output directory |
| `--verify` | Collected data ki verification |
| `--upload` | Vendor ko direct upload (support ke saath configured ho to) |

---

## 4. Report ke andar kya hota hai

Archive extract karo:

```bash
cd /var/tmp
tar -xf sosreport-<host>-<date>-<id>.tar.xz
cd sosreport-<host>-<date>-<id>
```

Typical structure:

```
sosreport-xxx/
  sos_commands/        # saare command outputs (plugin-wise)
    systemd/
    networking/
    kubernetes/
    block/ ...
  sos_logs/            # sos ke apne logs (kaun sa plugin fail hua)
  sos_reports/         # sos.html / sos.json (summary index)
  etc/                 # config files
  var/log/             # system logs
  proc/ sys/           # kernel runtime info
  installed-rpms       # package list
  uptime  date  hostname  df  free  ps  lsmod ...
```

Tip: `sos_reports/sos.html` browser me kholo, plugin-wise index mil jata hai.

---

## 5. Troubleshooting workflow

1. **Issue define karo**: kya fail hua, kab se, kaunse node pe
2. **Report collect karo** issue ke time ke paas (jitna jaldi, utna better)
3. **Extract karke pehle basics dekho**: uptime, date, df, free, dmesg
4. **Symptom ke hisaab se plugin directory me jao**
5. **Timeline banao**: kaunsa event pehle hua (logs ke timestamps)
6. **Root cause confirm karo**, fix apply karo, aur postmortem notes likho

### Pehle ye 5 cheezein dekho (quick triage)

```bash
cat uptime                         # reboot to nahi hua?
cat date                           # report ka timestamp
cat df                             # disk full to nahi?
cat free                           # memory pressure?
cat sos_commands/kernel/dmesg*     # kernel errors, OOM, hardware issues
```

---

## 6. Scenario-wise guide

### A. Node `NotReady` / kubelet crash

```bash
# kubelet service status
cat sos_commands/systemd/systemctl_status_kubelet*

# kubelet logs
grep -iE "error|failed|panic" sos_commands/logs/journalctl_--no-pager_--unit_kubelet* | tail -50

# Container runtime
cat sos_commands/systemd/systemctl_status_containerd*   # ya crio / docker

# Disk / memory pressure
cat df
cat free
```

Common causes: disk pressure, memory pressure, runtime down, certificate expiry, API server unreachable.

### B. Disk full / disk pressure

```bash
cat df
cat sos_commands/filesys/df_-al*
ls sos_commands/block/
cat sos_commands/lvm2/*       # LVM issues
```

Dhyan do: `/var/lib/containerd`, `/var/lib/kubelet`, `/var/log` ka size.

### C. OOM kill / memory issue

```bash
grep -iE "out of memory|oom-killer|killed process" sos_commands/kernel/dmesg*
grep -iE "oom" var/log/messages
cat free
cat sos_commands/process/ps_auxwww   # sabse zyada memory kaun use kar raha tha
```

### D. Network / CNI problem

```bash
cat sos_commands/networking/ip_-d_address
cat sos_commands/networking/ip_route*
cat sos_commands/networking/ss_*
ls sos_commands/networking/ | grep -i iptables     # iptables / nftables rules
cat etc/cni/net.d/*                                 # CNI config
```

Check karo: interface down, route missing, iptables rules, MTU mismatch, DNS (`etc/resolv.conf`).

### E. Service fail / systemd issue

```bash
cat sos_commands/systemd/systemctl_list-units_--all*
grep -i "failed" sos_commands/systemd/systemctl_list-units_--all*
cat sos_commands/systemd/systemctl_status_<service>*
```

### F. Kernel panic / unexpected reboot

```bash
cat uptime
grep -iE "panic|bug:|oops|call trace|mce|hardware error" sos_commands/kernel/dmesg*
ls var/crash 2>/dev/null          # kdump vmcore info
last -x reboot                    # (ya sos_commands/login/ me dekho)
```

### G. Boot / filesystem / mount issue

```bash
cat etc/fstab
cat proc/mounts
grep -iE "mount|fsck|ext4|xfs" sos_commands/kernel/dmesg*
```

### H. Package / upgrade issue

```bash
cat installed-rpms | grep -iE "kernel|kube|containerd|cri-o"
cat sos_commands/rpm/*
```

---

## 7. Kubernetes / container nodes pe sos chalana

### Node pe SSH ho to

```bash
sudo sos report --batch -o kubernetes,containerd,networking,systemd,kernel,block --case-id <id>
```

### SSH nahi hai (managed / OpenShift style)

```bash
# OpenShift
oc debug node/<node-name>
chroot /host
sos report --batch

# Generic Kubernetes
kubectl debug node/<node-name> -it --image=<debug-image>
chroot /host
sos report --batch
```

> Debug container me `/host` pe node ka root filesystem mount hota hai. `chroot /host` ke baad node ke tools use ho sakte hain (agar sos node pe installed ho).

### Multiple nodes ek saath

```bash
sos collect --cluster-type <type> ...   # sos collect: multi-node collection (cluster profile ke saath)
```

OpenShift me cluster-level data ke liye `oc adm must-gather` bhi chalao. sos node-level, must-gather cluster-level data deta hai.

---

## 8. Sensitive data aur security

- sos default me passwords/keys scrub karta hai, lekin **100% guarantee nahi**
- Hostnames, IPs, MAC addresses obfuscate karne ke liye:

```bash
# Collection ke time
sudo sos report --batch --clean

# Ya existing report pe baad me
sudo sos clean /var/tmp/sosreport-xxx.tar.xz
```

- `sos clean` ek **mapping file** banata hai (real vs obfuscated values). Isko report ke saath **share mat karo**
- Report share karne se pehle:
  - Kisko bhej rahe ho, confirm karo
  - Secure channel use karo (vendor portal, encrypted storage)
  - Kaam ho jane ke baad local copies delete karo
- Report me user names, hostnames, internal IPs, config details ho sakte hain. Isko confidential treat karo

---

## 9. Best practices

- Issue **ke time ya turant baad** report lo (reboot ke baad purane in-memory data/logs kho sakte hain)
- Report lene se pehle disk space check karo (`df -h /var/tmp`)
- Production me heavy plugins se bacho agar node already stressed hai (`-o` se limited plugins chalao)
- `--case-id` hamesha lagao, tracking easy hoti hai
- `.sha256` file ke saath bhejo, integrity verify ho sake
- Report ka summary (`sos_reports/sos.html`) pehle padho
- `sos_logs/` check karo, agar koi plugin timeout/fail hua to data incomplete ho sakta hai
- Same issue ke liye multiple nodes se reports lo, comparison me pattern milta hai

---

## 10. sos vs doosre tools

| Situation | Pehle kya use karo |
|-----------|--------------------|
| Metrics / trends | Prometheus, Grafana |
| App logs | Loki, ELK, `kubectl logs` |
| Live pod debugging | `kubectl describe`, `kubectl debug` |
| Node-level live debugging | `journalctl`, `dmesg`, `crictl`, `strace`, `tcpdump` |
| **Node ka full snapshot / vendor support / RCA evidence** | **`sos report`** |
| OpenShift cluster-level | `oc adm must-gather` |

---

## 11. Quick cheat sheet

```bash
# Collect
sudo sos report --batch --case-id 12345

# Limited + recent logs
sudo sos report --batch -o systemd,kernel,networking --since 20261020

# Obfuscate
sudo sos clean /var/tmp/sosreport-*.tar.xz

# Extract aur dekho
tar -xf sosreport-*.tar.xz && cd sosreport-*/
cat uptime df free
less sos_commands/kernel/dmesg*
firefox sos_reports/sos.html
```
