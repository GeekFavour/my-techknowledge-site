# SSL/TLS Certificate Expiry Checker — Multi-Host Scanner

## Purpose

Expired TLS certificates are one of the most embarrassing and disruptive incidents in
infrastructure operations — they take down HTTPS endpoints silently and often at the
worst possible time. Certificate expiry is entirely predictable, yet it catches teams
off guard when it falls outside normal deployment cycles or when a certificate was
provisioned outside the standard automation pipeline (e.g. manually uploaded to an
Azure Application Gateway, an internal service, or a load balancer).

This Python script scans a configurable list of hostnames and ports, retrieves the
live TLS certificate from each, and reports days remaining to expiry. It is used by:

- **Platform and DevOps teams** running weekly scans across production endpoints
- **Network/security engineers** auditing which certificates are self-signed vs.
  CA-issued
- **SREs** including it in pre-deployment runbooks to catch short-lived certificates
  before a release

It outputs a colour-coded terminal table, a CSV report, and exits non-zero if any
certificate is within the critical renewal window — making it pipeline-compatible.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Python | `>= 3.8` |
| Standard library only | `ssl`, `socket`, `csv`, `datetime` — no `pip install` needed |
| Network access | Outbound TCP to each target host on the specified port |
| Permissions | No elevated privileges required |

---

## Script

```python
#!/usr/bin/env python3
"""
ssl-cert-checker.py
Scans a list of hosts for TLS certificate expiry and reports status.

Usage:
    python3 ssl-cert-checker.py
    python3 ssl-cert-checker.py --hosts hosts.txt
    python3 ssl-cert-checker.py --host example.com --port 443

Configuration: edit TARGETS below or pass a --hosts file (one host[:port] per line).
"""

import ssl
import socket
import csv
import sys
import argparse
from datetime import datetime, timezone

# ── CONFIG ────────────────────────────────────────────────
WARN_DAYS  = 30   # Warn if certificate expires within this many days
CRIT_DAYS  = 14   # Critical if certificate expires within this many days
TIMEOUT    = 10   # TCP connection timeout in seconds
OUTPUT_CSV = f"cert_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"

# Default target list — edit or replace with --hosts file
TARGETS = [
    ("example.com",        443),
    ("mail.example.com",   443),
    ("api.example.com",    443),
    # Internal services
    # ("internal-app.corp",  8443),
    # ("vpn.corp",           443),
]
# ── END CONFIG ────────────────────────────────────────────

# ANSI colour helpers
GREEN  = "\033[92m"
YELLOW = "\033[93m"
RED    = "\033[91m"
RESET  = "\033[0m"
BOLD   = "\033[1m"


def get_cert_expiry(host: str, port: int, timeout: int) -> dict:
    """Connect to host:port, retrieve TLS certificate, return parsed info."""
    ctx = ssl.create_default_context()
    ctx.check_hostname = True
    ctx.verify_mode = ssl.CERT_REQUIRED

    try:
        with socket.create_connection((host, port), timeout=timeout) as sock:
            with ctx.wrap_socket(sock, server_hostname=host) as tls:
                cert = tls.getpeercert()

        # Parse expiry date from ASN.1 string
        expiry_str = cert["notAfter"]          # e.g. "Jun 15 12:00:00 2026 GMT"
        expiry_dt  = datetime.strptime(expiry_str, "%b %d %H:%M:%S %Y %Z")
        expiry_dt  = expiry_dt.replace(tzinfo=timezone.utc)
        now        = datetime.now(timezone.utc)
        days_left  = (expiry_dt - now).days

        # Extract Subject CN and SANs
        subject    = dict(x[0] for x in cert.get("subject", []))
        cn         = subject.get("commonName", "unknown")
        issuer     = dict(x[0] for x in cert.get("issuer", []))
        issuer_cn  = issuer.get("organizationName", issuer.get("commonName", "unknown"))
        sans       = [v for _, v in cert.get("subjectAltName", []) if _ == "DNS"]

        return {
            "host":       host,
            "port":       port,
            "cn":         cn,
            "issuer":     issuer_cn,
            "expiry":     expiry_dt.strftime("%Y-%m-%d"),
            "days_left":  days_left,
            "sans_count": len(sans),
            "error":      None,
        }

    except ssl.SSLCertVerificationError as e:
        return {"host": host, "port": port, "error": f"Cert verification failed: {e}",
                "days_left": -1}
    except ssl.SSLError as e:
        return {"host": host, "port": port, "error": f"SSL error: {e}",
                "days_left": -1}
    except (socket.timeout, ConnectionRefusedError, OSError) as e:
        return {"host": host, "port": port, "error": f"Connection failed: {e}",
                "days_left": -1}


def status_label(days: int) -> tuple[str, str]:
    """Return (colour, label) for terminal output based on days remaining."""
    if days < 0:
        return RED, "ERROR  "
    elif days <= CRIT_DAYS:
        return RED, "CRITICAL"
    elif days <= WARN_DAYS:
        return YELLOW, "WARNING "
    else:
        return GREEN, "OK      "


def main():
    parser = argparse.ArgumentParser(description="TLS certificate expiry checker")
    parser.add_argument("--hosts", help="Path to file with one host[:port] per line")
    parser.add_argument("--host",  help="Single host to check")
    parser.add_argument("--port",  type=int, default=443, help="Port (default: 443)")
    args = parser.parse_args()

    targets = list(TARGETS)  # Start with built-in list

    if args.host:
        targets = [(args.host, args.port)]
    elif args.hosts:
        with open(args.hosts) as f:
            for line in f:
                line = line.strip()
                if not line or line.startswith("#"):
                    continue
                parts = line.rsplit(":", 1)
                host = parts[0]
                port = int(parts[1]) if len(parts) == 2 else 443
                targets.append((host, port))

    print(f"\n{BOLD}TLS Certificate Expiry Report — {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}{RESET}")
    print("─" * 90)
    print(f"{'Host':<35} {'Port':<6} {'CN':<30} {'Expiry':<12} {'Days':>6}  {'Status'}")
    print("─" * 90)

    results     = []
    exit_code   = 0
    crit_count  = 0
    warn_count  = 0

    for host, port in targets:
        r = get_cert_expiry(host, port, TIMEOUT)
        colour, label = status_label(r.get("days_left", -1))

        if r.get("error"):
            print(f"{host:<35} {port:<6} {'—':<30} {'—':<12} {'—':>6}  {RED}{label}{RESET}  {r['error']}")
            exit_code = 1
        else:
            days = r["days_left"]
            print(
                f"{host:<35} {port:<6} {r['cn']:<30} {r['expiry']:<12} {days:>6}  "
                f"{colour}{label}{RESET}"
            )
            if days <= CRIT_DAYS:
                crit_count += 1
                exit_code = 1
            elif days <= WARN_DAYS:
                warn_count += 1
                if exit_code == 0:
                    exit_code = 1

        results.append(r)

    print("─" * 90)
    print(f"\nSummary: {len(targets)} hosts checked — "
          f"{RED}{crit_count} critical{RESET}, "
          f"{YELLOW}{warn_count} warnings{RESET}")

    # Write CSV
    with open(OUTPUT_CSV, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=[
            "host", "port", "cn", "issuer", "expiry", "days_left", "sans_count", "error"
        ])
        writer.writeheader()
        writer.writerows(results)

    print(f"Report saved to: {OUTPUT_CSV}\n")
    sys.exit(exit_code)


if __name__ == "__main__":
    main()
```

---

## How It Works

1. **`ssl.create_default_context()`** — Creates a context that validates the certificate
   chain against the system trust store (same as a browser). This catches expired,
   self-signed, and hostname-mismatch errors separately, so you know *why* a cert failed.

2. **`getpeercert()`** — Returns the decoded certificate as a Python dict. The
   `notAfter` field is parsed from the ASN.1 date string format used by OpenSSL.

3. **Days calculation** — Uses `datetime` with UTC timezone awareness to avoid
   off-by-one errors from local timezone offsets during DST transitions.

4. **SAN count** — Subject Alternative Names indicate how many hostnames the
   certificate covers; a count of 1 on a wildcard-less cert is a signal that the
   cert is tightly scoped.

5. **Exit code** — Returns `1` if any certificate is in warning or critical state.
   This allows the script to be used as a gate in CI/CD pipelines:
   `python3 ssl-cert-checker.py || exit 1`

---

## Example Output

```
TLS Certificate Expiry Report — 2025-06-01 07:05:11
──────────────────────────────────────────────────────────────────────────────────────────
Host                                Port   CN                             Expiry       Days  Status
──────────────────────────────────────────────────────────────────────────────────────────
example.com                         443    example.com                    2026-01-14    227  OK
api.example.com                     443    *.example.com                  2025-06-20     19  WARNING
mail.example.com                    443    mail.example.com               2025-06-08      7  CRITICAL
internal-app.corp                   8443   —                              —              —   ERROR    Connection failed: [Errno 111] Connection refused
──────────────────────────────────────────────────────────────────────────────────────────

Summary: 4 hosts checked — 1 critical, 1 warnings
Report saved to: cert_report_20250601_070511.csv
```

---

## Using a Hosts File

Create `hosts.txt` with one entry per line:

```
# Production endpoints
example.com:443
api.example.com:443
# Internal services
internal-lb.corp:8443
vpn.corp:443
```

Run:

```bash
python3 ssl-cert-checker.py --hosts hosts.txt
```

---

## Scheduling

**Weekly cron scan with email summary:**

```cron
0 8 * * 1 /usr/bin/python3 /opt/scripts/ssl-cert-checker.py \
  --hosts /opt/scripts/prod-hosts.txt \
  2>&1 | mail -s "[Cert Report] Weekly TLS Scan" ops@example.com
```

**Azure DevOps pipeline gate (runs on every release):**

```yaml
- task: PythonScript@0
  displayName: 'Check TLS certificate expiry'
  inputs:
    scriptPath: 'scripts/ssl-cert-checker.py'
    arguments: '--hosts $(Build.SourcesDirectory)/config/prod-hosts.txt'
```

---

## Best Practices & Notes

- **Internal/private CAs** — For hosts using an internal CA (common in corporate
  networks or Azure internal load balancers), pass `ctx.check_hostname = False` and
  `ctx.verify_mode = ssl.CERT_NONE` and check expiry only. Alternatively, load the
  internal CA bundle:
  ```python
  ctx.load_verify_locations("/etc/ssl/certs/internal-ca.pem")
  ```

- **Azure Application Gateway / Front Door** — Certificates uploaded manually to
  Azure Application Gateway do not auto-renew. Include these listener endpoints
  explicitly in your host list.

- **Let's Encrypt / ACME** — LE certs expire in 90 days. Run the scan weekly and
  treat `WARN_DAYS=21`, `CRIT_DAYS=7` as appropriate thresholds for auto-renewed certs;
  a warning at 21 days gives time to debug a broken renewal before it goes critical.

- **Mutual TLS (mTLS)** — For services requiring client certificates, extend
  `ctx.load_cert_chain(certfile, keyfile)` before wrapping the socket.

- **Port 25 / STARTTLS** — For SMTP/STARTTLS, the connection upgrade requires
  `smtplib.SMTP` with `.starttls()` rather than a raw TLS socket; adapt the
  `get_cert_expiry` function accordingly for mail server checks.

- **Notification integration** — Replace the `mail` command with a call to the
  Microsoft Teams webhook or PagerDuty Events API for alerting in teams that use
  chat-based operations.
