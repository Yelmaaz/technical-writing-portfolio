
---

## Metadata

| Field | Value |
|---|---|
| **Runbook ID** | OPS-RUN-001 |
| **Service** | [SERVICE_NAME] |
| **Environment(s)** | Production / Staging / Development |
| **Severity Levels** | SEV-1 (complete outage), SEV-2 (degraded), SEV-3 (at risk) |
| **Last Reviewed** | [YYYY-MM-DD] |
| **Owner** | [TEAM OR INDIVIDUAL] |
| **Escalation Contact** | [NAME / CHANNEL / PAGER] |

### Assumptions and Prerequisites

- Responder has SSH access to the affected host(s)
- Responder has sudo privileges or equivalent
- Monitoring and alerting are configured for the service
- A known-good configuration backup exists and is accessible
- Rollback artifacts (previous package version, container image, or config snapshot) are available
- Incident logging channel or system is accessible

---

## 1. Trigger Conditions

This runbook applies when any of the following conditions are observed:

- Monitoring alert fires for service health check failure
- Automated endpoint probe returns a non-200 response or times out
- User-reported service unavailability
- Dependent service raises a connectivity or timeout error
- On-call responder identifies anomalous behaviour during routine review

Do not use this runbook for planned maintenance windows. If a deployment or scheduled restart is in progress, refer to the deployment runbook before proceeding.

---

## 2. Symptom Detection

Before beginning triage, document the initial symptom. Common presentations:

| Symptom | Likely Severity |
|---|---|
| Service completely unreachable, health check failing | SEV-1 |
| Service responding but returning errors or degraded output | SEV-2 |
| Elevated latency, intermittent failures, or resource pressure | SEV-3 |
| Single instance affected, others healthy | SEV-3 |

Record the following before proceeding:

- Time the issue was first observed or reported
- Source of the alert (monitoring system, user report, manual check)
- Estimated user or system impact
- Whether any deployment, configuration change, or infrastructure event preceded the issue

---

## 3. Initial Triage

Before inspecting the service itself, rule out external causes.

### Network connectivity

Confirm the host is reachable:

```bash
ping -c 4 [HOST]
ssh [USER]@[HOST]
```

Confirm the service port is reachable from an external host:

```bash
curl -v http://[HOST]:[PORT]/health
# or
nc -zv [HOST] [PORT]
```

### Dependency availability

Identify upstream dependencies (databases, caches, external APIs) and confirm they are reachable:

```bash
# Example: confirm database port is open
nc -zv [DB_HOST] [DB_PORT]

# Example: confirm DNS resolution is working
dig [DEPENDENCY_HOSTNAME]
```

If a dependency is unavailable, this runbook may not apply. Escalate to the team responsible for the dependency and continue monitoring.

### Concurrent events

Before proceeding, check:

- Is a deployment currently in progress?
- Has infrastructure changed recently (scaling event, certificate renewal, DNS update)?
- Are other services on the same host affected?

```bash
# Check recent systemd activity across all services
sudo journalctl --since "30 minutes ago" -p err
```

If multiple services are failing simultaneously, the issue is likely infrastructure-level. Escalate immediately.

---

## 4. Service Status Verification

Confirm the service state using systemctl:

```bash
sudo systemctl status [SERVICE_NAME]
```

Interpret the output:

| Status | Meaning |
|---|---|
| `active (running)` | Service is running. Issue may be application-level, not service-level. |
| `inactive (dead)` | Service is stopped and was not expected to be. |
| `failed` | Service exited with an error. Exit code and signal are shown. |
| `activating` | Service is attempting to start. May be stuck. |
| `deactivating` | Service is shutting down. May be stuck. |

Check whether the service is enabled to start on boot:

```bash
sudo systemctl is-enabled [SERVICE_NAME]
```

If the service is `disabled`, it will not restart after a reboot. Note this for the incident log but do not re-enable without understanding why it was disabled.

Check the exit code from the last failure:

```bash
sudo systemctl show [SERVICE_NAME] --property=ExecMainStatus
```

A non-zero exit code indicates the process itself failed. An exit code of 0 with a failed state suggests the service was killed externally (OOM killer, signal, or timeout).

---

## 5. Log Inspection

### View recent service logs

```bash
sudo journalctl -u [SERVICE_NAME] -n 100 --no-pager
```

### View logs since the last boot

```bash
sudo journalctl -u [SERVICE_NAME] -b
```

### View logs from a specific time window

```bash
sudo journalctl -u [SERVICE_NAME] --since "2024-01-15 14:00:00" --until "2024-01-15 14:30:00"
```

### Filter by priority

```bash
# Show only errors and above
sudo journalctl -u [SERVICE_NAME] -p err

# Show warnings and above
sudo journalctl -u [SERVICE_NAME] -p warning
```

### Follow logs in real time

```bash
sudo journalctl -u [SERVICE_NAME] -f
```

### What to look for

| Log Pattern | Likely Cause |
|---|---|
| `permission denied` | File or socket permission issue |
| `address already in use` | Port conflict, previous instance still running |
| `no such file or directory` | Missing binary, config file, or dependency |
| `connection refused` / `timeout` | Upstream dependency unavailable |
| `out of memory` / `OOM` | Memory exhaustion, process killed by kernel |
| `segmentation fault` / `core dumped` | Application crash, possible bug or corrupted state |
| `failed to load configuration` | Config file syntax error or missing required value |
| `certificate` / `TLS` / `SSL` errors | Expired or misconfigured certificate |

Note the first error in the log sequence, not just the most recent. The root cause is usually earlier than where the log ends.

---

## 6. Failure Classification

Based on log inspection and triage findings, classify the failure before proceeding to remediation:

| Category | Indicators |
|---|---|
| **Configuration drift** | Service was working previously, config file was recently modified, syntax or value errors in logs |
| **Dependency failure** | Upstream service unreachable, connection timeout or refused errors |
| **Resource exhaustion** | OOM kill, disk full errors, too many open files |
| **Deployment regression** | Failure began after a package update, image push, or code deployment |
| **Permission or security issue** | Permission denied errors, ownership changes, SELinux or AppArmor denials |
| **External dependency outage** | Third-party API, DNS, or network path unavailable |
| **Unknown** | No clear pattern in logs, requires deeper investigation |

Record the classification in the incident log. If the classification is unknown, escalate rather than attempting remediation without a hypothesis.

---

## 7. Safety Checks Before Remediation

Do not proceed to remediation without completing the following checks.

- [ ] **Maintenance window** — confirm whether a maintenance window is active or needs to be opened before making changes
- [ ] **Active user impact** — determine how many users or systems are currently affected and whether remediation will extend or temporarily worsen the impact
- [ ] **Concurrent deployments** — confirm no deployment is currently in progress that may conflict with remediation steps
- [ ] **Rollback artifacts available** — confirm that a previous known-good package version, container image, or configuration file is accessible before making changes
- [ ] **Configuration snapshot** — back up the current config before modifying it, even if it appears corrupted

```bash
sudo cp /etc/[SERVICE_NAME]/[CONFIG_FILE] /etc/[SERVICE_NAME]/[CONFIG_FILE].bak.$(date +%Y%m%d%H%M%S)
```

- [ ] **Peer confirmation** — for SEV-1 incidents, confirm remediation steps with a second responder before executing

---

## 8. Recovery Procedures

Follow the branch that matches the failure classification identified in Section 6.

### 8.1 Low-Risk Recovery (unknown or transient failure)

Attempt this first only if the failure classification is unclear and logs show no obvious cause.

Reload the systemd daemon to pick up any unit file changes:

```bash
sudo systemctl daemon-reload
```

Restart the service:

```bash
sudo systemctl restart [SERVICE_NAME]
```

Monitor logs immediately after restart:

```bash
sudo journalctl -u [SERVICE_NAME] -f
```

If the service starts and remains stable for five minutes, proceed to post-resolution validation. If it fails again, do not repeat the restart. Move to the appropriate branch below.

### 8.2 Configuration Recovery

Use this branch when logs indicate a configuration error.

Validate the configuration file syntax before making changes. Many services include a built-in check:

```bash
# Examples (use the command appropriate to your service)
[SERVICE_NAME] --test
[SERVICE_NAME] configtest
[SERVICE_NAME] -t
```

If the syntax check fails, identify the reported line and error. Compare against the backup:

```bash
diff /etc/[SERVICE_NAME]/[CONFIG_FILE] /etc/[SERVICE_NAME]/[CONFIG_FILE].bak.[TIMESTAMP]
```

Restore the backup if the current config is corrupted or the error cannot be resolved quickly:

```bash
sudo cp /etc/[SERVICE_NAME]/[CONFIG_FILE].bak.[TIMESTAMP] /etc/[SERVICE_NAME]/[CONFIG_FILE]
```

Validate again, then restart:

```bash
sudo systemctl restart [SERVICE_NAME]
```

### 8.3 Deployment Rollback

Use this branch when the failure followed a deployment event.

Identify the previously installed version:

```bash
# For package-managed services
apt list --installed [PACKAGE_NAME]
dnf history list [PACKAGE_NAME]
pacman -Q [PACKAGE_NAME]

# For container-based services
docker images [IMAGE_NAME]
```

Roll back to the previous version:

```bash
# Debian/Ubuntu
sudo apt install [PACKAGE_NAME]=[PREVIOUS_VERSION]

# RHEL/Fedora
sudo dnf downgrade [PACKAGE_NAME]

# Arch Linux
sudo pacman -U /var/cache/pacman/pkg/[PACKAGE_NAME]-[PREVIOUS_VERSION].pkg.tar.zst
```

Restart the service and monitor logs:

```bash
sudo systemctl restart [SERVICE_NAME]
sudo journalctl -u [SERVICE_NAME] -f
```

### 8.4 Resource Recovery

Use this branch when the failure is caused by resource exhaustion.

**Disk space:**

```bash
# Check disk usage
df -h

# Identify large files in common locations
du -sh /var/log/* | sort -rh | head -20
du -sh /tmp/* | sort -rh | head -20

# Rotate or truncate logs if safe to do so
sudo journalctl --vacuum-size=500M
sudo truncate -s 0 /var/log/[SERVICE_NAME]/[LOG_FILE]
```

**Memory exhaustion:**

```bash
# Check current memory usage
free -h

# Identify top memory consumers
ps aux --sort=-%mem | head -20

# Check if OOM killer was involved
sudo journalctl -k | grep -i oom
```

If a memory leak is suspected, restart the service to reclaim memory, but note that the issue will recur without a fix. Escalate to the development team.

**File descriptor limits:**

```bash
# Check current limits for the service
cat /proc/$(pgrep [SERVICE_NAME])/limits

# Check current usage
ls /proc/$(pgrep [SERVICE_NAME])/fd | wc -l
```

If limits are being hit, increase them in the service unit file under `[Service]`:

```ini
LimitNOFILE=65536
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart [SERVICE_NAME]
```

---

## 9. Escalation Criteria

Stop remediation and escalate immediately if any of the following apply:

- The service has failed to start after two restart attempts with no change in error output
- Logs indicate data corruption, unrecoverable state, or security-related events
- Multiple services or hosts are affected simultaneously
- The failure classification is unknown and no hypothesis can be formed from available logs
- Remediation steps have been exhausted without resolution
- The incident has persisted beyond [ESCALATION_THRESHOLD, e.g. 30 minutes] without improvement

When escalating, provide:

- Time of initial failure and time of escalation
- Failure classification (or "unknown")
- All diagnostic commands run and their output
- Remediation steps attempted and results
- Current service state

Escalation contact: **[NAME / CHANNEL / PAGER POLICY]**

---

## 10. Post-Resolution Validation

Do not close the incident until all of the following have been confirmed.

### Service state

```bash
sudo systemctl status [SERVICE_NAME]
```

Expected: `active (running)`. Note the process uptime — a service that is running but restarting repeatedly will show a very short uptime.

### Endpoint response

```bash
curl -v http://[HOST]:[PORT]/health
```

Expected: HTTP 200 with a valid response body. If the service exposes multiple endpoints, test the primary user-facing path in addition to the health check.

### Dependency connectivity

Confirm the service can reach its upstream dependencies:

```bash
# Re-run dependency checks from Section 3
nc -zv [DB_HOST] [DB_PORT]
```

### Log stability

Monitor logs for five minutes after restart to confirm no new errors are appearing:

```bash
sudo journalctl -u [SERVICE_NAME] -f
```

A service that appears healthy but is logging recurring warnings may be in a degraded state that will lead to another failure.

### Metric normalisation

If monitoring dashboards are available, confirm that key metrics (request rate, error rate, latency, resource usage) have returned to baseline values.

### User impact confirmation

Confirm with the original reporter or monitoring system that the service is functional from the user perspective, not just from the server side.

---

## 11. Root Cause Classification

After resolution, record the final root cause category for trend analysis:

- [ ] Configuration drift
- [ ] Dependency failure
- [ ] Resource exhaustion
- [ ] Deployment regression
- [ ] Permission or security issue
- [ ] External dependency outage
- [ ] Unknown / requires deeper investigation

If the root cause is unknown or requires deeper investigation, open a follow-up task before closing the incident.

---

## 12. Incident Documentation Template

Complete this template for every incident, regardless of severity or duration.

```
INCIDENT REPORT
===============
Incident ID:          [INC-YYYY-MM-DD-NNN]
Service:              [SERVICE_NAME]
Environment:          [prod / staging / dev]
Severity:             [SEV-1 / SEV-2 / SEV-3]

DETECTION
---------
Alert time:           [YYYY-MM-DD HH:MM UTC]
Detected by:          [monitoring system / user report / manual]
Observed symptoms:    

Impacted systems:     
Estimated users affected:

DIAGNOSTICS
-----------
Commands run and output:


REMEDIATION
-----------
Safety checks completed:   [yes / no — note any skipped and reason]
Actions taken:


RESOLUTION
----------
Resolution time:      [YYYY-MM-DD HH:MM UTC]
Total duration:       
Root cause category:  
Root cause summary:   

POST-RESOLUTION
---------------
Validation steps completed:   [yes / no]
Follow-up actions required:   [yes / no]
Follow-up task ID:            
```

---

## 13. Known False Positives

The following conditions may trigger alerts or appear as failures but do not require remediation.

| Condition | Explanation | How to Confirm |
|---|---|---|
| Delayed health check during startup | Many services take 10–60 seconds to become ready after a restart. Monitoring may fire before the service is fully initialised. | Check uptime with `systemctl status`. If the service is `active (running)` and uptime is increasing, wait for the health check to pass. |
| Transient dependency timeout | Short-lived network interruptions can cause a single failed health check without a sustained outage. | Check whether the service recovered on its own within one or two check intervals. |
| Monitoring lag after recovery | Some monitoring systems have a delay between service recovery and alert resolution. The service may be healthy before the alert clears. | Validate directly with `curl` or `systemctl status` rather than relying on the alert state. |
| Expected restart during deployment | Deployments typically involve a controlled service restart. This will appear as a brief outage in monitoring. | Confirm against the deployment schedule or change log before treating as an incident. |
| OOM kill during scheduled batch job | Memory-intensive scheduled tasks may cause temporary resource pressure that resolves on completion. | Check cron schedules and compare the OOM timestamp against expected job windows. |

---

## 14. Revision History

| Date | Author | Changes |
|---|---|---|
| [YYYY-MM-DD] | [NAME] | Initial version |
| | | |
