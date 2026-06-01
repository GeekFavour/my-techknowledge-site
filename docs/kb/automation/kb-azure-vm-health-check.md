# Azure VM Bulk Health Check & Status Report

## Purpose

In Azure environments with tens or hundreds of Virtual Machines across multiple resource groups
and subscriptions, manually checking VM power states, OS disk health, and boot diagnostics
through the portal is slow and error-prone.

This script automates a daily or on-demand health snapshot of all VMs in a subscription —
reporting power state, provisioning status, OS type, VM size, and location — and outputs
both a terminal table and a timestamped CSV report. It is commonly used by:

- Infrastructure engineers running morning health checks before business hours
- On-call SREs triaging alerts that reference VM states
- Platform teams generating asset inventory reports for audits or change management

---

## Prerequisites

| Requirement | Details |
|---|---|
| Azure CLI | `>= 2.50.0` — install via `apt install azure-cli` or `brew install azure-cli` |
| `jq` | JSON processor — `apt install jq` |
| `column` | Usually pre-installed on Ubuntu; part of `util-linux` |
| Azure RBAC | `Reader` role on the target subscription is sufficient |
| Login | `az login` (interactive) or a Service Principal via `az login --service-principal` |

---

## Script

```bash
#!/usr/bin/env bash
# ============================================================
# azure-vm-health-check.sh
# Generates a health summary of all VMs in an Azure subscription.
# Usage: ./azure-vm-health-check.sh [SUBSCRIPTION_ID]
# ============================================================

set -euo pipefail

SUBSCRIPTION="${1:-$(az account show --query id -o tsv)}"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
OUTPUT_CSV="vm_health_${TIMESTAMP}.csv"

echo "🔍 Fetching VM health for subscription: ${SUBSCRIPTION}"
echo "-----------------------------------------------------------"

az account set --subscription "${SUBSCRIPTION}"

# Write CSV header
echo "Name,ResourceGroup,Location,Size,OS,PowerState,ProvisioningState" > "${OUTPUT_CSV}"

# Fetch all VMs with instance view (includes power state)
az vm list \
  --show-details \
  --query "[].{
    Name:name,
    ResourceGroup:resourceGroup,
    Location:location,
    Size:hardwareProfile.vmSize,
    OS:storageProfile.osDisk.osType,
    PowerState:powerState,
    ProvisioningState:provisioningState
  }" \
  -o json | jq -r '.[] |
    [
      .Name,
      .ResourceGroup,
      .Location,
      .Size,
      .OS,
      (.PowerState // "unknown"),
      .ProvisioningState
    ] | @csv' | while IFS=',' read -r name rg loc size os power prov; do

  # Strip surrounding quotes from jq @csv output
  name=$(echo "$name" | tr -d '"')
  power=$(echo "$power" | tr -d '"')
  prov=$(echo "$prov" | tr -d '"')

  # Colour-code power state for terminal output
  if [[ "$power" == *"running"* ]]; then
    state_display="\e[32m${power}\e[0m"   # Green
  elif [[ "$power" == *"deallocated"* ]]; then
    state_display="\e[33m${power}\e[0m"   # Yellow
  else
    state_display="\e[31m${power}\e[0m"   # Red
  fi

  printf "%-30s %-20s %-12s %b\n" \
    "$name" \
    "$(echo "$rg" | tr -d '"')" \
    "$(echo "$loc" | tr -d '"')" \
    "$state_display"

  # Append to CSV (plain, no ANSI codes)
  echo "${name},$(echo "$rg" | tr -d '"'),$(echo "$loc" | tr -d '"'),$(echo "$size" | tr -d '"'),$(echo "$os" | tr -d '"'),${power},${prov}" \
    >> "${OUTPUT_CSV}"
done

echo ""
echo "-----------------------------------------------------------"
echo "✅ Report saved to: ${OUTPUT_CSV}"
echo "Total VMs processed: $(tail -n +2 "${OUTPUT_CSV}" | wc -l)"

# Summary counts
running=$(grep -ic "VM running" "${OUTPUT_CSV}" || true)
deallocated=$(grep -ic "VM deallocated" "${OUTPUT_CSV}" || true)
echo "  Running     : ${running}"
echo "  Deallocated : ${deallocated}"
```

---

## How It Works

1. **`az account set`** — Pins the CLI context to the target subscription so all queries
   are scoped correctly even if the operator has multiple subscriptions.

2. **`az vm list --show-details`** — The `--show-details` flag makes a second call per VM
   to retrieve the live power state (running, deallocated, stopped, etc.). Without it,
   you only get the provisioning-time metadata.

3. **JQ query** — Projects only the fields needed, reducing payload size and avoiding
   `null` dereferencing issues.

4. **`@csv` output** — Produces RFC-4180-compliant CSV rows that are safe for Excel/Sheets
   import with no post-processing.

5. **ANSI colour codes** — Make the terminal output scannable at a glance; they are
   stripped from the CSV so the file is clean.

---

## Example Output

```
🔍 Fetching VM health for subscription: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
-----------------------------------------------------------
prod-web-01             rg-production        eastus       VM running
prod-web-02             rg-production        eastus       VM running
dev-jumpbox-01          rg-dev               westeurope   VM deallocated
staging-api-01          rg-staging           eastus       VM stopped
-----------------------------------------------------------
✅ Report saved to: vm_health_20250601_080012.csv
Total VMs processed: 4
  Running     : 2
  Deallocated : 1
```

---

## Scheduling (Cron / Azure Automation)

**Run daily at 07:00 from a Linux jump host:**

```cron
0 7 * * * /opt/scripts/azure-vm-health-check.sh >> /var/log/vm-health.log 2>&1
```

**Run as an Azure Automation Runbook (PowerShell equivalent):**
Replace `az vm list` with `Get-AzVM -Status` if porting to a PowerShell runbook inside
Azure Automation, which avoids the need for a dedicated runner host.

---

## Best Practices & Notes

- **Service Principal for CI/CD** — Never use interactive `az login` in automated pipelines.
  Create a Service Principal (`az ad sp create-for-rbac`) and store credentials in Azure Key
  Vault or a pipeline secret store.

- **Scope to resource group for large subscriptions** — Add `--resource-group <rg>` to
  `az vm list` if you have 200+ VMs; the `--show-details` flag issues one extra API call
  per VM and can approach ARM throttling limits (12,000 reads per hour).

- **Extend with disk/NIC checks** — Pipe the CSV output into a second script that calls
  `az vm show --query storageProfile.osDisk.managedDisk` to verify disk encryption and SKU.

- **Alert on unexpected states** — Add a check after the loop:
  ```bash
  if grep -q "VM stopped" "${OUTPUT_CSV}"; then
    echo "⚠️  WARNING: Unexpected stopped VMs detected" | mail -s "VM Alert" ops@example.com
  fi
  ```

- **Tag filtering** — Add `--query "[?tags.environment=='production']"` to filter to a
  specific environment without changing resource groups.
