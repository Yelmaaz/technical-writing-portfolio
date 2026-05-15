# Suspicious Linux Activity

> **IMPORTANT:** This playbook is for investigative use only. Do not attempt remediation or containment without completing the Containment Decision Point in Section 4. Premature action can destroy evidence, alert an attacker, or cause unrecoverable data loss.

---

## Metadata

| Field | Value |
|---|---|
| **Playbook ID** | SEC-PLAY-001 |
| **Scope** | Linux endpoints investigated remotely over SSH |
| **Severity Levels** | P1 (confirmed compromise), P2 (likely compromise), P3 (suspicious/inconclusive), P4 (benign anomaly) |
| **Last Reviewed** | [YYYY-MM-DD] |
| **Owner** | [TEAM OR INDIVIDUAL] |
| **Escalation Contact** | [NAME / CHANNEL / PAGER] |
| **Prerequisites** | SSH access to target host, sudo privileges, local investigation log open before connecting |

---

## 1. Trigger Conditions

This playbook applies when any of the following conditions are observed:

| Trigger | Example |
|---|---|
| SIEM or IDS alert | Unusual outbound connection, privilege escalation detected, known malicious IP contacted |
| Authentication anomaly | Multiple failed SSH attempts followed by success, login from unexpected geography |
| User or team report | Unexpected system behaviour, unknown processes, suspicious files discovered |
| Automated scan finding | Vulnerability scanner or EDR flags anomalous process or file |
| Threat intelligence match | IOC (IP, hash, domain) matched against system logs or network traffic |
| Anomalous resource usage | Unexplained CPU or network spike consistent with cryptomining or exfiltration |

Record the trigger source and timestamp before proceeding.

---

## 2. Pre-Investigation Checklist

Complete all items before issuing any investigative commands.

- [ ] **Open a local investigation log** — record every command issued and its output. Do not rely on terminal history alone. A simple timestamped text file is sufficient.

```bash
script -a /path/to/investigation-$(date +%Y%m%d%H%M%S).log
```

- [ ] **Establish a stable SSH session** — use a multiplexed or persistent session to avoid losing state mid-investigation. Consider tmux or screen on the target host immediately after connecting.

```bash
ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=10 [USER]@[HOST]
```

- [ ] **Note your own presence** — your SSH session will appear in logs and process lists. Record your login time and source IP so you can exclude your own activity from findings.

- [ ] **Do not run remediation tools** — antivirus scans, cleanup scripts, or restarts can overwrite volatile evidence before it is captured. Investigation precedes remediation.

- [ ] **Do not alert the attacker** — avoid actions that would be visible to a monitoring attacker: do not kill suspicious processes, do not delete files, do not change passwords yet.

- [ ] **Confirm investigation authorisation** — verify you have explicit authorisation to investigate this host before proceeding.

---

## 3. Initial Containment Decision

Before beginning investigation, make an explicit containment decision. Containment and investigation are in tension: containment protects the environment but may destroy evidence or alert an attacker. This decision must be made consciously, not by default.

Work through the following questions:

**Is the system actively exfiltrating data?**
If yes, the cost of continued connection may outweigh the investigative value of leaving it live. Consider immediate network isolation.

**Will isolation destroy critical evidence?**
Memory-resident malware, active C2 connections, and in-progress processes may leave no recoverable trace after isolation. If volatile evidence has not yet been captured, delay isolation until Section 5 is complete.

**Does containment risk tipping off the attacker?**
An attacker monitoring their implant for connectivity may interpret a sudden network drop as detection and trigger a wipe or secondary payload. In some cases a controlled live investigation is preferable to immediate isolation.

**Is business continuity critical?**
Isolating a production system may cause an outage. Confirm with the relevant business owner before taking the system offline.

**Containment options by severity:**

| Situation | Recommended Action |
|---|---|
| Active exfiltration confirmed | Immediate network isolation after volatile capture |
| Likely compromise, no active exfiltration | Proceed with live investigation, prepare isolation |
| Suspicious but inconclusive | Live investigation only, no containment yet |
| Benign anomaly suspected | Investigation only |

Record the containment decision and rationale in the investigation log before proceeding.

---

## 4. Volatile State Capture

Volatile data exists only while the system is running. It is stored in RAM and will be permanently lost if the system is rebooted or shut down. Capture it before anything else. Order matters — the most time-sensitive data first.

### 4.1 Timestamp and system identity

```bash
date -u
hostname
uname -a
uptime
```

### 4.2 Active network connections

```bash
ss -tulnp
ss -anp
netstat -tulnp   # if ss is unavailable
```

Note all established outbound connections, particularly those on unusual ports or connecting to external IPs. Cross-reference destination IPs against threat intelligence feeds if available.

### 4.3 Current logged-in users and sessions

```bash
who
w
last -a | head -30
lastb | head -20   # failed login attempts
```

### 4.4 Running processes

```bash
ps auxf           # full process tree with parent-child relationships
ps aux --sort=-%cpu | head -20
ps aux --sort=-%mem | head -20
```

Look for:
- Processes with no associated terminal running as root
- Processes with names that mimic system binaries (e.g. `sshd` running from `/tmp`)
- Processes with unusually high CPU or memory usage
- Orphaned processes with PPID of 1 that were not started by systemd

### 4.5 Deleted binaries still running in memory

A common attacker technique is to execute a binary and immediately delete it from disk. The process remains in memory but the file is gone.

```bash
ls -la /proc/*/exe 2>/dev/null | grep deleted
```

Any result here is a significant indicator of compromise. The process ID can be used to extract further information:

```bash
ls -la /proc/[PID]/
cat /proc/[PID]/cmdline | tr '\0' ' '
cat /proc/[PID]/maps
```

### 4.6 Shell history

Advanced attackers clear their bash history, but automated tools, worms, and less experienced attackers often do not. Capture it early before any session ends or history is rotated.

```bash
cat /home/[USER]/.bash_history
cat /root/.bash_history
history
```

Look for recently run commands involving network tools (`curl`, `wget`, `nc`, `ncat`), privilege escalation attempts, file downloads, or encoded payloads.

### 4.8 Mounted filesystems

```bash
mount
df -h
cat /proc/mounts
```

Note any unexpected mount points, particularly tmpfs mounts in unusual locations or external filesystems.

### 4.9 Loaded kernel modules

```bash
lsmod
```

Rootkits frequently load malicious kernel modules. Compare against a known-good baseline if available. Modules with generic or misspelled names warrant investigation.

---

## 5. Process Inspection

With volatile state captured, inspect processes in more depth.

### 5.1 Process environment variables

Environment variables can reveal attacker infrastructure, C2 addresses, or injected configuration:

```bash
cat /proc/[PID]/environ | tr '\0' '\n'
```

### 5.2 Open files per process

```bash
lsof -p [PID]
```

Look for processes with open file handles in `/tmp`, `/dev/shm`, or other world-writable directories.

### 5.3 Process working directory

```bash
ls -la /proc/[PID]/cwd
```

A legitimate system service running from `/tmp` or a user home directory is anomalous.

### 5.4 Processes listening on unexpected ports

Cross-reference the process list against expected services:

```bash
ss -tulnp | grep -v -E '(sshd|nginx|postgres|your_expected_services)'
```

### 5.5 Container and namespace inspection

If the host runs containers:

```bash
docker ps -a
docker inspect [CONTAINER_ID]
```

Attackers may use containers to isolate malicious workloads or escape detection by host-level monitoring.

---

## 6. Network Socket Analysis

### 6.1 All active connections with process association

```bash
ss -anp
```

### 6.2 Unusual outbound connections

Filter for established outbound connections:

```bash
ss -anp | grep ESTABLISHED
```

For each suspicious destination IP, perform a passive lookup:

```bash
whois [IP]
dig -x [IP]
```

Note: perform lookups locally or via a controlled resolver. DNS lookups from the target host may be logged by an attacker monitoring DNS traffic.

### 6.3 Listening services not associated with known processes

```bash
ss -tulnp | grep -v [PID_OF_KNOWN_SERVICES]
```

### 6.4 Raw socket usage

Raw socket access can indicate packet sniffing or network-level attack tools:

```bash
cat /proc/net/raw
cat /proc/net/raw6
```

### 6.5 Firewall rules

Inspect whether firewall rules have been modified to permit unusual traffic:

```bash
sudo iptables -L -n -v
sudo nft list ruleset   # if nftables is in use
```

### 6.6 Live traffic capture (optional)

If active exfiltration is suspected and the host has not yet been isolated, capture a short traffic sample for later analysis. Store the capture file on external storage.

```bash
sudo tcpdump -i [INTERFACE] -w /external/path/capture-$(date +%Y%m%d%H%M%S).pcap -G 60 -W 1
```

This captures 60 seconds of traffic. Adjust the interface (`-i eth0`, `-i ens3`, etc.) to match the active network interface identified in Section 4.2. Do not run this on the loopback interface.

---

## 7. Authentication and Access Review

### 7.1 Recent successful logins

```bash
last -a | head -50
sudo journalctl _SYSTEMD_UNIT=sshd.service | grep "Accepted" | tail -50
```

### 7.2 Recent failed login attempts

```bash
lastb | head -50
sudo journalctl _SYSTEMD_UNIT=sshd.service | grep "Failed" | tail -50
sudo grep "Failed password" /var/log/auth.log | tail -50   # Debian/Ubuntu
sudo grep "Failed password" /var/log/secure | tail -50     # RHEL/Fedora
```

### 7.3 Impossible travel patterns

Review login timestamps and source IPs for the same account. A successful login from one country followed minutes later by a login from a different country indicates credential compromise or session hijacking.

```bash
last -a | grep [USERNAME]
```

### 7.4 SSH authorised keys modifications

```bash
find /home -name "authorized_keys" -exec ls -la {} \;
find /root -name "authorized_keys" -exec ls -la {} \;
```

Check modification timestamps. An attacker establishing persistence via SSH will add a public key to an authorised_keys file.

```bash
cat /home/[USER]/.ssh/authorized_keys
cat /root/.ssh/authorized_keys
```

### 7.5 SSH agent forwarding anomalies

If SSH agent forwarding is in use, an attacker with access to the target host can use the forwarded agent to authenticate to other hosts as the original user. Check for active forwarded agents:

```bash
env | grep SSH_AUTH_SOCK
ls -la $SSH_AUTH_SOCK
```

### 7.6 Sudo usage

```bash
sudo grep "sudo" /var/log/auth.log | tail -50
sudo journalctl | grep "sudo" | tail -50
```

Look for sudo commands issued by unexpected users or at unusual times.

### 7.7 New or modified user accounts

```bash
cat /etc/passwd | grep -v "nologin\|false"
cat /etc/shadow | awk -F: '$2 != "!" && $2 != "*" {print $1}'   # accounts with passwords set
lastlog | grep -v "Never"
```

### 7.8 Privilege boundary crossings

Unexpected use of service accounts for interactive login, or non-admin accounts successfully running privileged commands, warrants investigation:

```bash
sudo journalctl | grep -E "(su|sudo|su -)" | grep -v "pam_unix"
```

---

## 8. Persistence Mechanism Checks

Attackers establish persistence to survive reboots and maintain access. Check all common persistence locations.

### 8.1 Cron jobs

```bash
crontab -l
sudo crontab -l
cat /etc/crontab
ls -la /etc/cron.*
for user in $(cut -f1 -d: /etc/passwd); do echo "=== $user ==="; crontab -u $user -l 2>/dev/null; done
```

### 8.2 Systemd units

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service
find /etc/systemd /usr/lib/systemd /home -name "*.service" -newer /bin/bash
```

New or recently modified service unit files in user home directories are particularly suspicious.

### 8.3 Shell profile modifications

```bash
cat /etc/profile
cat /etc/bashrc
cat /etc/bash.bashrc
cat /home/[USER]/.bashrc
cat /home/[USER]/.bash_profile
cat /home/[USER]/.profile
cat /root/.bashrc
cat /root/.bash_profile
```

Look for unexpected commands, base64-encoded strings, or network calls embedded in shell profiles.

### 8.4 SUID and SGID binaries

SUID binaries run with the permissions of the file owner regardless of who executes them. An attacker may plant or modify a SUID binary to maintain root access.

```bash
find / -perm /4000 -type f 2>/dev/null   # SUID
find / -perm /2000 -type f 2>/dev/null   # SGID
```

Compare results against a known-good baseline. Any SUID binary outside of standard system paths (`/bin`, `/usr/bin`, `/usr/sbin`) warrants investigation.

### 8.5 Package integrity verification

Verify that installed system binaries match their expected checksums:

```bash
# Debian/Ubuntu
sudo debsums -c

# RHEL/Fedora/Arch
sudo rpm -Va | grep -E "^..5"   # RHEL
sudo pacman -Qk 2>/dev/null | grep warning   # Arch
```

A modified system binary (particularly `ls`, `ps`, `netstat`, `find`, or `ssh`) is a strong indicator of rootkit installation.

### 8.6 Immutable file attributes

Attackers may use `chattr +i` to make malicious files immutable, preventing even root from deleting or modifying them. Check suspicious files for this attribute:

```bash
lsattr /path/to/suspicious/file
lsattr /tmp /var/tmp /dev/shm
```

An `i` flag in the output indicates the file is immutable. This is rarely set on legitimate files outside of specific system configurations.

### 8.7 Unexpected binary replacement

Check critical system binaries for unexpected modification times:

```bash
ls -la /bin/ls /bin/ps /usr/bin/netstat /usr/bin/find /usr/sbin/sshd
stat /usr/sbin/sshd
```

Compare against package manager records:

```bash
dpkg -S /usr/sbin/sshd   # Debian/Ubuntu
rpm -qf /usr/sbin/sshd   # RHEL/Fedora
pacman -Qo /usr/sbin/sshd   # Arch
```

---

## 9. Filesystem Integrity Review

### 9.1 Recently modified files

```bash
find / -mtime -1 -type f 2>/dev/null | grep -v -E "(/proc|/sys|/dev|/run)"
find /tmp /var/tmp /dev/shm -type f 2>/dev/null
```

### 9.2 Executables in world-writable directories

```bash
find /tmp /var/tmp /dev/shm -type f -executable 2>/dev/null
find / -writable -type f -executable 2>/dev/null | grep -v -E "(/proc|/sys)"
```

### 9.3 Hidden files and directories

```bash
find / -name ".*" -type f 2>/dev/null | grep -v -E "(/proc|/sys|/home/[^/]+/\.[^/]+$)"
find /tmp /var/tmp /dev/shm -name ".*" 2>/dev/null
```

### 9.4 Large or unusual files in unexpected locations

```bash
find /tmp /var/tmp /dev/shm / -size +10M -type f 2>/dev/null | grep -v -E "(/proc|/sys|/usr|/var/lib)"
```

### 9.5 Timestomping detection

Attackers often alter file timestamps (a technique called timestomping) to make malicious files blend in with legitimate ones or to appear older than they are. Look for files with timestamps that do not make sense in context:

```bash
# Find files with modification times in 1970 (a common timestomping artifact)
find / -mtime +19000 -type f 2>/dev/null | grep -v -E "(/proc|/sys)"

# Check inode change time (ctime) vs modification time (mtime) for discrepancies
stat /path/to/suspicious/file
```

The `ctime` (inode change time) cannot be altered by userspace tools, while `mtime` can. A file with a very old `mtime` but a recent `ctime` is a strong indicator of timestomping.

### 9.6 Files with no associated package

On package-managed systems, identify files that do not belong to any installed package:

```bash
# Debian/Ubuntu
find /usr /bin /sbin -type f | xargs dpkg -S 2>&1 | grep "no path found"

# RHEL/Fedora
rpm -Va --nofiles 2>/dev/null
```

---

## 10. Evidence Preservation

Preserve evidence before any remediation or containment action. Evidence collected after containment may be challenged for chain-of-custody reasons.

### 10.1 Chain of custody notes

For each piece of evidence collected, record:

- Timestamp of collection (UTC)
- Identity of the collector
- Method used to collect
- Hash of the collected artifact
- Storage location

```bash
# Example: hash a captured file
sha256sum /path/to/captured/file >> investigation-evidence-log.txt
```

### 10.2 Memory capture

If memory forensics may be required, capture a memory image before any process termination or system restart:

```bash
# Using LiME (Linux Memory Extractor) if available
sudo insmod lime.ko "path=/external/path/memory.lime format=lime"

# Using /proc/kcore (limited but available without additional tools)
sudo dd if=/proc/kcore of=/external/path/kcore.img
```

Store memory images on external storage, not on the target host.

### 10.3 Log preservation

Copy logs before they rotate or are overwritten:

```bash
sudo cp /var/log/auth.log /external/path/auth.log.$(date +%Y%m%d%H%M%S)
sudo journalctl --since "7 days ago" > /external/path/journal-$(date +%Y%m%d%H%M%S).log
```

### 10.4 Process and connection snapshot

```bash
ps auxf > /external/path/ps-snapshot-$(date +%Y%m%d%H%M%S).txt
ss -anp > /external/path/ss-snapshot-$(date +%Y%m%d%H%M%S).txt
```

### 10.5 Volatile state export

Export all volatile state captured in Section 4 to external storage or a secure investigation system before proceeding to containment.

---

## 11. Confidence Assessment

Based on findings, assign a confidence level before deciding on next actions.

| Level | Definition | Example Findings |
|---|---|---|
| **P4 — Benign Anomaly** | Findings have a plausible legitimate explanation | Unusual process belongs to a legitimate scheduled task, login from new IP explained by VPN change |
| **P3 — Suspicious / Inconclusive** | Findings are anomalous but no confirmed malicious activity | Unexpected listening port with no associated process, recently modified binary with no package change |
| **P2 — Likely Compromise** | Multiple indicators consistent with compromise, no alternative explanation | Deleted binary still running, new SSH key added, outbound connection to known malicious IP |
| **P1 — Confirmed Compromise** | Definitive evidence of malicious activity | Active C2 connection, rootkit detected, confirmed data exfiltration, attacker-controlled account |

Assign a confidence level and record the supporting evidence in the investigation log. Do not escalate to P1 or P2 without documented supporting findings.

---

## 12. Escalation Decision

Escalate immediately if any of the following apply:

- Confidence level is P1 or P2
- Evidence suggests active data exfiltration
- Multiple hosts appear to be affected (lateral movement indicators)
- Findings are beyond the investigator's ability to interpret
- Attacker appears to be actively present on the system
- Evidence of destructive capability (wiper malware, ransomware indicators)

When escalating, provide:

- Playbook ID and investigation log
- Trigger condition and timestamp
- Confidence level and supporting evidence summary
- Volatile state snapshots
- Containment status and recommendation

Escalation contact: **[NAME / CHANNEL / PAGER POLICY]**

---

## 13. Recommended Actions by Confidence Level

| Confidence Level | Recommended Action |
|---|---|
| **P4 — Benign Anomaly** | Close investigation with documented findings. Note trigger for future tuning. |
| **P3 — Suspicious / Inconclusive** | Increase monitoring on the host. Schedule follow-up review in 24–48 hours. Do not remediate. |
| **P2 — Likely Compromise** | Escalate immediately. Prepare for containment. Do not remediate without IR team involvement. |
| **P1 — Confirmed Compromise** | Escalate immediately. Execute containment plan. Activate incident response procedure. Preserve all evidence before remediation. |

---

## 14. Investigation Log Template

Complete this template throughout the investigation. Do not wait until the end.

```
INVESTIGATION LOG
=================
Log ID:               [INV-YYYY-MM-DD-NNN]
Playbook:             SEC-PLAY-001
Target host:          [HOSTNAME / IP]
Environment:          [prod / staging / dev]
Investigator:         [NAME]
Investigation start:  [YYYY-MM-DD HH:MM UTC]

TRIGGER
-------
Source:               [SIEM / user report / scan / other]
Alert or description: 
Timestamp of trigger: [YYYY-MM-DD HH:MM UTC]

CONTAINMENT DECISION
--------------------
Decision:             [isolate / live investigation / defer]
Rationale:            
Authorised by:        

VOLATILE STATE SUMMARY
----------------------
Active suspicious connections:    
Suspicious processes identified:  
Deleted binaries in memory:       
Unexpected kernel modules:        

FINDINGS BY SECTION
-------------------
Process inspection:       
Network analysis:         
Authentication review:    
Persistence checks:       
Filesystem review:        

EVIDENCE COLLECTED
------------------
Artifact                  Hash (SHA256)             Storage location
---------                 -------------             ----------------


CONFIDENCE ASSESSMENT
---------------------
Level:                [P1 / P2 / P3 / P4]
Supporting evidence:  

ESCALATION
----------
Escalated:            [yes / no]
Escalated to:         
Escalation time:      [YYYY-MM-DD HH:MM UTC]

OUTCOME
-------
Resolution:           
Investigation close:  [YYYY-MM-DD HH:MM UTC]
Follow-up actions:    
```

---

## 15. Known False Positives

| Condition | Explanation | How to Confirm |
|---|---|---|
| Monitoring agent processes | Security agents (EDR, SIEM forwarders, vulnerability scanners) often run as root, open unusual ports, and make outbound connections. They may appear suspicious in isolation. | Verify the process name and path against your organisation's approved tooling list. |
| Deployment pipeline activity | CI/CD pipelines may connect to hosts, install packages, and modify files at unexpected times. | Cross-reference against deployment schedule or pipeline logs. |
| Legitimate cron jobs running as root | System maintenance scripts may run at night, consume high CPU, and write to unusual locations. | Review crontab entries against known scheduled tasks before flagging. |
| Package manager activity | Automated updates may modify binaries, run as root, and make outbound connections to package mirrors. | Check package manager logs for scheduled update activity. |
| SSH login from new IP after VPN change | A user logging in from a new IP after changing VPN provider or working remotely may trigger impossible travel alerts. | Confirm with the user directly before treating as an indicator. |
| Temporary files created by legitimate applications | Some applications write executables or scripts to `/tmp` during normal operation. | Identify the parent process and verify it is a known application. |
| High CPU from legitimate cryptographic operations | Backup tools, compression utilities, and database operations can cause CPU spikes that resemble cryptomining. | Correlate with scheduled jobs and verify the process binary. |

---

## 16. Revision History

| Date | Author | Changes |
|---|---|---|
| [YYYY-MM-DD] | [NAME] | Initial version |
| | | |
