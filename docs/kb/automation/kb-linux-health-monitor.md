# Linux Server Health Monitor — Disk, Memory, CPU & Service Watchdog

## Purpose

Unplanned downtime caused by full disks, runaway memory, or silently crashed services
is one of the most common (and avoidable) problems in day-to-day Linux server operations.
Monitoring dashboards such as Grafana/Prometheus are excellent but require infrastructure
and setup time. This self-contained Bash script runs on any Ubuntu/Debian or RHEL/CentOS
server with zero dependencies and performs a fast local health check:

- Disk usage per mount point (warns at configurable thresholds)
- Memory and swap usage
- CPU load average vs. core count
- Status of critical system services defined in a list
- Logs findings to a file and optionally sends an email alert

It is typically used as a:

- **Cron job** running every 5–15 minutes on application servers, database hosts,
  and build agents
- **Pre-flight check** run manually before deployments or maintenance windows
- **Baseline health script** deployed via Ansible to every managed node

---

## Prerequisites

| Requirement | Details |
|---|---|
| OS | Ubuntu 20.04+ / Debian 11+ / RHEL 8+ / Amazon Linux 2 |
| Bash | `>= 4.0` (standard on all modern distros) |
| `df`, `free`, `uptime` | Pre-installed on all Linux distributions |
| `systemctl` | Required for service checks (systemd-based systems) |
| `mailutils` (optional) | `apt install mailutils` — only needed for email alerts |

---

## Script

```bash
#!/usr/bin/env bash
# ============================================================
# linux-health-monitor.sh
# Local health check: disk, memory, CPU load, and services.
# Usage: ./linux-health-monitor.sh
# Config: edit the variables in the CONFIG section below.
# ============================================================

set -uo pipefail

# ── CONFIG ────────────────────────────────────────────────
DISK_WARN_PCT=75          # Warn when disk usage exceeds this %
DISK_CRIT_PCT=90          # Critical when disk usage exceeds this %
LOAD_WARN_MULTIPLIER=1.5  # Warn if load avg > (cores * multiplier)
ALERT_EMAIL=""            # Set to an address to enable email alerts
LOG_FILE="/var/log/server-health.log"

# List of systemd services to verify are active
CRITICAL_SERVICES=(
  "ssh"
  "cron"
  "rsyslog"
  # Add your own: "nginx" "postgresql" "docker" "azure-monitor-agent"
)
# ── END CONFIG ────────────────────────────────────────────

HOSTNAME=$(hostname -f)
TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
ISSUES=()
REPORT=""

log() {
  local level="$1"; shift
  local msg="[$level] $*"
  REPORT+="${msg}\n"
  echo -e "${msg}"
  echo "${TIMESTAMP} ${msg}" >> "${LOG_FILE}"
}

separator() { log "INFO" "─────────────────────────────────────────"; }

# ── HEADER ────────────────────────────────────────────────
separator
log "INFO" "Server Health Check — ${HOSTNAME}"
log "INFO" "Time: ${TIMESTAMP}"
separator

# ── DISK USAGE ────────────────────────────────────────────
log "INFO" "DISK USAGE"
while IFS= read -r line; do
  pct=$(echo "$line" | awk '{print $5}' | tr -d '%')
  mount=$(echo "$line" | awk '{print $6}')
  used=$(echo "$line" | awk '{print $3}')
  total=$(echo "$line" | awk '{print $2}')

  if (( pct >= DISK_CRIT_PCT )); then
    log "CRIT " "  ${mount}: ${pct}% used (${used} / ${total}) ← CRITICAL"
    ISSUES+=("CRITICAL: Disk ${mount} at ${pct}%")
  elif (( pct >= DISK_WARN_PCT )); then
    log "WARN " "  ${mount}: ${pct}% used (${used} / ${total}) ← WARNING"
    ISSUES+=("WARNING: Disk ${mount} at ${pct}%")
  else
    log "OK   " "  ${mount}: ${pct}% used (${used} / ${total})"
  fi
done < <(df -h --output=source,size,used,avail,pcent,target -x tmpfs -x devtmpfs \
          | tail -n +2)

separator

# ── MEMORY ────────────────────────────────────────────────
log "INFO" "MEMORY & SWAP"
mem_total=$(free -m | awk '/^Mem:/ {print $2}')
mem_used=$(free -m  | awk '/^Mem:/ {print $3}')
mem_pct=$(( mem_used * 100 / mem_total ))
swap_total=$(free -m | awk '/^Swap:/ {print $2}')
swap_used=$(free -m  | awk '/^Swap:/ {print $3}')

log "INFO" "  RAM : ${mem_used}MB / ${mem_total}MB (${mem_pct}%)"
if (( swap_total > 0 )); then
  swap_pct=$(( swap_used * 100 / swap_total ))
  log "INFO" "  Swap: ${swap_used}MB / ${swap_total}MB (${swap_pct}%)"
  if (( swap_pct > 50 )); then
    ISSUES+=("WARNING: Swap usage at ${swap_pct}%")
  fi
else
  log "INFO" "  Swap: none configured"
fi

separator

# ── CPU LOAD ──────────────────────────────────────────────
log "INFO" "CPU LOAD"
cores=$(nproc)
load_1min=$(uptime | awk -F'[,:]' '{print $(NF-2)}' | xargs)
warn_threshold=$(echo "${cores} * ${LOAD_WARN_MULTIPLIER}" | bc)

log "INFO" "  Cores   : ${cores}"
log "INFO" "  Load 1m : ${load_1min} (warn at ${warn_threshold})"

if (( $(echo "${load_1min} > ${warn_threshold}" | bc -l) )); then
  log "WARN " "  Load average is HIGH: ${load_1min} on ${cores} cores"
  ISSUES+=("WARNING: High CPU load ${load_1min} on ${cores} cores")
else
  log "OK   " "  Load is normal"
fi

separator

# ── SERVICES ──────────────────────────────────────────────
log "INFO" "CRITICAL SERVICES"
for svc in "${CRITICAL_SERVICES[@]}"; do
  if systemctl is-active --quiet "${svc}"; then
    log "OK   " "  ${svc}: active"
  else
    status=$(systemctl is-active "${svc}" 2>/dev/null || echo "not-found")
    log "CRIT " "  ${svc}: ${status} ← NOT RUNNING"
    ISSUES+=("CRITICAL: Service ${svc} is ${status}")
  fi
done

separator

# ── SUMMARY ───────────────────────────────────────────────
if (( ${#ISSUES[@]} == 0 )); then
  log "OK   " "✅  All checks passed — no issues found."
else
  log "WARN " "⚠️   ${#ISSUES[@]} issue(s) detected:"
  for issue in "${ISSUES[@]}"; do
    log "     " "    • ${issue}"
  done

  # Send email if configured
  if [[ -n "${ALERT_EMAIL}" ]]; then
    subject="[ALERT] ${HOSTNAME} — ${#ISSUES[@]} health issue(s) at ${TIMESTAMP}"
    echo -e "${REPORT}" | mail -s "${subject}" "${ALERT_EMAIL}"
    log "INFO" "Alert email sent to ${ALERT_EMAIL}"
  fi
fi

separator
exit $(( ${#ISSUES[@]} > 0 ? 1 : 0 ))
```

---

## How It Works

1. **Disk check** — Uses `df -h` with `--output` to select exactly the columns needed.
   Excludes `tmpfs` and `devtmpfs` mounts (memory-backed filesystems) to avoid false
   positives. Thresholds are configurable per environment.

2. **Memory check** — `free -m` reports in megabytes for integer arithmetic in Bash.
   Swap tracking is included because swap exhaustion is a common precursor to OOM kills.

3. **CPU load** — Compares the 1-minute load average against `nproc` (physical + virtual
   cores). A multiplier of 1.5 means a 4-core machine alerts above load 6.0 — typical
   for sustained overload rather than brief spikes.

4. **Service watchdog** — `systemctl is-active --quiet` returns exit code 0 only if the
   unit is in the `active (running)` state. `failed`, `inactive`, and `activating` all
   trigger the alert path.

5. **Exit code** — Returns `1` if any issue was found, `0` if clean. This makes it
   composable in CI pipelines or monitoring wrappers that check exit codes.

---

## Example Output

```
[INFO] ─────────────────────────────────────────
[INFO] Server Health Check — prod-app-01.internal
[INFO] Time: 2025-06-01 07:00:03
[INFO] ─────────────────────────────────────────
[INFO] DISK USAGE
[OK   ]   /: 42% used (42G / 100G)
[WARN ]   /data: 78% used (390G / 500G) ← WARNING
[INFO] ─────────────────────────────────────────
[INFO] MEMORY & SWAP
[INFO]   RAM : 6240MB / 8192MB (76%)
[INFO]   Swap: 512MB / 2048MB (25%)
[INFO] ─────────────────────────────────────────
[INFO] CPU LOAD
[INFO]   Cores   : 4
[INFO]   Load 1m : 1.23 (warn at 6.0)
[OK   ]   Load is normal
[INFO] ─────────────────────────────────────────
[INFO] CRITICAL SERVICES
[OK   ]   ssh: active
[OK   ]   cron: active
[CRIT ]   rsyslog: failed ← NOT RUNNING
[INFO] ─────────────────────────────────────────
[WARN ]   ⚠️  2 issue(s) detected:
[     ]     • WARNING: Disk /data at 78%
[     ]     • CRITICAL: Service rsyslog is failed
[INFO] ─────────────────────────────────────────
```

---

## Scheduling

**Run every 10 minutes via cron (as root or a privileged service account):**

```cron
*/10 * * * * /opt/scripts/linux-health-monitor.sh >> /var/log/server-health.log 2>&1
```

**Deploy to all managed nodes via Ansible:**

```yaml
- name: Deploy health monitor script
  copy:
    src: linux-health-monitor.sh
    dest: /opt/scripts/linux-health-monitor.sh
    mode: '0755'

- name: Schedule health monitor cron
  cron:
    name: "Server health check"
    minute: "*/10"
    job: "/opt/scripts/linux-health-monitor.sh >> /var/log/server-health.log 2>&1"
    user: root
```

---

## Best Practices & Notes

- **Rotate the log file** — Add a `/etc/logrotate.d/server-health` entry:
  ```
  /var/log/server-health.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
  }
  ```

- **Adjust thresholds per role** — Database servers often warrant a lower disk warn
  threshold (`DISK_WARN_PCT=65`) due to write-ahead logs growing fast. Build agents
  may tolerate higher load (`LOAD_WARN_MULTIPLIER=3`).

- **Integrate with Azure Monitor** — The script's exit code can feed into a custom
  script extension or Azure Arc health probe. Log output to `/var/log/server-health.log`
  and stream it to a Log Analytics workspace via the Azure Monitor Agent.

- **Extend the service list** — Common additions:
  `nginx`, `apache2`, `postgresql`, `mysql`, `docker`, `kubelet`,
  `azure-monitor-agent`, `walinuxagent` (Azure Linux Agent).

- **Run as non-root for read-only checks** — All checks except service status work
  without root. Add the monitoring user to the `systemd-journal` group:
  `usermod -aG systemd-journal monitor_user`
