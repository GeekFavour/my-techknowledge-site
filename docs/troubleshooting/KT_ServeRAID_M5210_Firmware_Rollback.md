# KT: ServeRAID M5210 Firmware Upgrade Broke Drive Discovery — Rollback to Recover

**Document type:** Knowledge Transfer / Runbook
**Subject:** Controller firmware upgrade rendered RAID5 array invisible; recovery by firmware rollback
**Server:** Lenovo System x3650 M5, Type **5462**
**Date of incident:** 2026-06-20

---

## 1. Issue Statement

The **firmware of the ServeRAID M5210 controller alone was upgraded** (from package **24.7.0-0056** to **24.21.0-0151**). No hardware, cabling, or drive changes were made.

Immediately after the upgrade and reboot:

- The IMM (Integrated Management Module) showed **all five physical disks as "Critical"**, listed under *"Non-manageable drives to IMM"*.
- The controller reported **"No drives found"** and **no virtual disk**.
- The server **would not boot into Windows Server 2019** (the OS installed on the RAID5 virtual disk).

The disks were healthy and the array was running a normal consistency check ~25 minutes before the upgrade. Only the firmware changed in between. This established the firmware upgrade as the trigger and the data on the platters as very likely intact (a discovery/link problem, not media failure).

**Resolution:** Roll the controller firmware back to the original, known-good **24.7.0-0056** using `storcli` with the `noverchk` flag, then perform a full AC power cycle so the recovered virtual disk re-presents and Windows boots.

---

## 2. Environment

| Item | Value |
|---|---|
| Server | Lenovo System x3650 M5, Type 5462 |
| OS on array | Windows Server 2019 |
| RAID controller | IBM/Lenovo ServeRAID M5210 (PCI Slot 9) |
| Controller chip | Broadcom/LSI MegaRAID SAS-3 **3108 [Invader]**, SAS-12G |
| PCI IDs | Vendor 0x1000, Device 0x5D, SubVendor **0x1014** (Lenovo/IBM), SubDevice **0x454** |
| Cache / battery | 1 GB cache, CacheVault **CVPM02** (Optimal), BBU present |
| OEMID | Lenovo |
| Array | RAID5, 5 × 1.8 TB SATA (FRU 00FN114), Virtual Disk ≈ 7.27 TB |
| Backplane | **IBM-ESXS SAS EXP BP, Rev N551** (SAS expander backplane) |
| IMM | IMM2 (default IP 192.168.70.125) |
| Recovery toolset | Ubuntu 24.04.2 Live USB, `megaraid_sas` 07.727.03.00, `storcli` 007.1022 |

### 2.1 Accessing the IMM (out-of-band management)

The IMM (Integrated Management Module II) is the server's baseboard management controller. It runs independently of the OS, so it is reachable even when the server will not boot — it is how the disks were first seen as "Critical" in this incident.

**Physical connection**
- On the x3650 M5, the IMM has a **dedicated systems-management RJ-45 port on the rear** of the chassis (marked with a wrench/management icon), separate from the regular data NICs.
- Connect an Ethernet cable directly from a laptop to that port (a standard patch cable works; no crossover needed).

**Configure the laptop NIC**
- The IMM default address is **192.168.70.125 / 255.255.255.0**.
- Set the laptop's wired NIC to a **static IP in the same subnet but a different host**, e.g. IP `192.168.70.100`, mask `255.255.255.0` (gateway not required).
- Browse to **`https://192.168.70.125`** and accept the self-signed certificate warning (expected). Default credentials on these servers are user `USERID` / password `PASSW0RD` (the "0" is a zero); newer firmware may force a password change on first login, or the credentials may already have been customized.

---

## 3. Firmware Versions

| | Original (known-good) | Upgraded (broke it) |
|---|---|---|
| Package build | **24.7.0-0056** | **24.21.0-0151** |
| MR Firmware | 4.270.00-4382 | 4.680.00-8561 |
| iMR Firmware | — | 4.680.01-8560 |
| BIOS | — | 6.36.00.3 |
| HII | — | 03.25.05.14 |
| MR NVDATA | — | 3.1705.00-0024 |
| iMR NVDATA | — | 3.1705.01-0018 |
| UEFI Driver | — | 0x06180205 |
| Boot Block | — | 3.07.00.00-0004 |
| CPLD | — | 26514-02A |

> **Note on the flash log "double image" entries:** the IMM flash log showed *two* APP images and *two* NVDATA images being written. This is **normal** — the 24.21.0-0151 package carries both the **MR** (full MegaRAID: 4.680.00-8561 / NVDATA 3.1705.00-0024) and **iMR** (entry: 4.680.01-8560 / NVDATA 3.1705.01-0018) personalities. The M5210 correctly settled on the MR personality. This was *not* a corrupt or wrong package.

---

## 4. Symptoms & Evidence

### 4.1 IMM (Local Storage → Physical Resource)
- ServeRAID M5210 (PCI Slot 9): **"No drives found."**
- Drive 0–4 under *"Non-manageable drives to IMM"*: all **Critical**. 6 errors total.

### 4.2 IMM RAID Logs — post-upgrade boot (key events, newest first)
| From reboot | Event | Severity | Message |
|---|---|---|---|
| 00:01:57–58 | 185 ×5 | **Critical** | Enclosure PD 00 (c Port 4 – 7/p1) **phy bad for slot 0–4** |
| 00:01:03–04 | 241 | Warning | **Previous configuration completely missing at boot** |
| 00:00:57–58 | 547 | Info | Encl PD 00 Inquiry: **IBM-SAS EXP BP 0 GB** (scsiType=d = expander/SES) |
| 00:00:11 | 261 | Info | Package version **24.21.0-0151** |
| 00:00:07 | 1 | Info | Firmware version **4.680.00-8561** |

For contrast, the **previous (old-firmware) boots** showed Package **24.7.0-0056** / FW **4.270.00-4382**, all five PDs present (slots s0,s1,s2,s4,s3), and a healthy consistency check (Event 408) — i.e. the array was fine until the flash.

### 4.3 Live-Ubuntu diagnosis (storcli + OS)
| Check | Command | Result |
|---|---|---|
| Controller present & healthy | `lspci`/`lshw`; `storcli /c0 show` | MegaRAID SAS-3 3108 [Invader]; **Controller Status = Optimal**; CacheVault Optimal; **no preserved cache** |
| Enclosures | `storcli /c0/eall show` | EID 0 = SAS EXP BP (Port 4-7 & 0-3 x8) **PD=0**; EID 252 = SGPIO **PD=0** |
| Drives | `storcli /c0/eall/sall show` | **"No drive found!"** |
| Virtual disks | `storcli /c0/vall show` | **"No VD's have been configured."** |
| Foreign config | `storcli /c0/fall show` | **"Couldn't find any foreign Configuration"** |
| OS view | `cat /proc/scsi/scsi`, `lsscsi -g`, `dmesg` | Only the **IBM-ESXS SAS EXP BP** enclosure (`/dev/sg0`) + the USB stick. **Zero disks.** |

---

## 5. Root Cause

The upgrade to firmware package **24.21.0-0151** broke the controller's ability to **discover the drives behind the IBM SAS expander backplane**:

- The controller-to-expander link is healthy — the expander enumerates as a SCSI enclosure (`IBM-ESXS SAS EXP BP`, Rev N551).
- But every drive-slot PHY (slots 0–4) reports **"phy bad"**, no disks present, so the controller declares the previous configuration missing.
- Because no disks are visible, there is no VD and no foreign config to import — the RAID metadata (stored on the drives) simply cannot be read.

Since the drives were Online and healthy minutes earlier and **only the firmware changed**, this is a firmware/discovery regression with this specific backplane, not drive or media failure. Therefore rolling the firmware back to the version that demonstrably worked (24.7.0-0056) is the correct remediation, and the on-disk data is expected to be intact.

---

## 6. Step-by-Step Remediation Procedure

> Perform all flashing **on the affected x3650 M5**, booted to the Ubuntu Live USB, using the **Linux** `storcli64` (never the `.exe`). The firmware ROM is OS-independent — flashing from Linux does not affect the Windows install on the array.

### About the Live Ubuntu USB (for those new to it)

A "Live" Ubuntu USB boots a complete, working Ubuntu **into RAM directly from the USB stick** — it does **not** install anything to, or modify, the server's internal disks. This lets us run hardware/diagnostic tools (`storcli`, `lsscsi`, `dmesg`) and flash the controller without disturbing the Windows installation on the RAID array. Key points:

- **Create it:** download the Ubuntu Desktop ISO and write it to a USB stick with Rufus (on Windows), balenaEtcher, or `dd` (on Linux).
- **Boot it:** power on the server, open the boot menu (**F12** on the x3650 M5), select the USB device, and choose **"Try Ubuntu"** — **not** "Install Ubuntu."
- **It's non-persistent:** anything you install during the session (e.g. `sg3-utils`, `lsscsi`) lives in RAM and disappears on reboot. That is fine here — the only durable action is the firmware flash, which writes to the *controller's* flash chip, not to the USB or the array.
- **Why storcli works from it:** Ubuntu's in-box `megaraid_sas` driver binds to the controller at boot, so `storcli` can talk to it in-band exactly as it would under the installed OS.

### Phase 0 — Pre-flight checks
1. Confirm controller is seen and Optimal, and that there is **no preserved/pinned cache** and the CacheVault/BBU is healthy:
   ```bash
   sudo ./storcli64 /c0 show all | grep -iE "Controller Status|Preserved|BBU|CacheVault|Package|Firmware"
   ```
2. Confirm there is **no foreign config and no VD** to accidentally damage (expected at this point):
   ```bash
   sudo ./storcli64 /c0/fall show
   sudo ./storcli64 /c0/vall show
   ```

### Phase 1 — (Optional but free) Cold reseat
A full power drain can clear a transient PHY/expander negotiation issue:
1. Shut down, **pull both power cords**, hold power button ~15 s to drain.
2. Reseat the M5210 card, **both ends of the internal mini-SAS cables**, the backplane connectors, and all five drives.
3. Boot the Live USB and re-check `sudo ./storcli64 /c0/eall/sall show`. If all five disks reappear, the firmware rollback may be unnecessary.

### Phase 2 — Obtain the correct firmware (24.7.0-0056)
**Critical:** download the **M5200-series** package (chip SAS3108), **not** the ServeRAID **M1215** package (chip SAS3008). Flashing the M1215 image onto an M5210 can brick the controller.

- Correct file: **`*sraidmr_5200*-24.7.0-0056*`** — contains **`mr3108.rom`**.
- Wrong file (do not use): `lnvgy_fw_sraidmr_1215-…` (ServeRAID M1215).
- Linux/VMware build: IBM legacy support page *"BIOS and Firmware Update v24.7.0-0056 for Linux and VMware – Lenovo x86 Servers"* (`ibm_fw_sraidmr_5200-24.7.0-0056_linux_32-64.bin`).
- If only the **Windows** package is available, the ROM inside is byte-identical and can be extracted on Linux:
  ```bash
  sudo apt install -y p7zip-full
  7z x lnvgy_fw_sraidmr_5200-24.7.0-0056_windows_32-64.exe -o/tmp/m5210fw
  find /tmp/m5210fw -iname '*.rom'        # -> image/mr3108.rom
  ```

### Phase 3 — Validate the ROM matches this controller
1. `cat ctlr-info.txt` — must map this controller's SubDevice Id to the ROM:
   ```
   0454,ServeRAID M5210,mr3108.rom,No FILE,
   ```
   `0454` = the M5210's SubDevice Id (`0x454`) → **confirms `mr3108.rom` is the correct image**.
2. `cat readme.txt` — confirms M5200-series package, supports ServeRAID M5210, version 24.7.0-0056.
   - Readme also confirms: **rollback is only supported via `storcli`** (not via the installer / `install.bat` / `lsiMRupdate`), and Lenovo only *verified* rollback to adjacent levels (24.7.0-0052, 24.2.1-0052).

### Phase 4 — Flash the rollback
1. Place `storcli64` (Linux) and `mr3108.rom` together. If `chmod +x` fails on a FAT/NTFS USB ("Operation not permitted"), copy both to a native path:
   ```bash
   mkdir -p ~/fw && cp storcli64 mr3108.rom ~/fw/ && cd ~/fw
   chmod +x storcli64
   ```
2. Confirm storcli sees the controller (shows current 24.21.0-0151):
   ```bash
   sudo ./storcli64 /c0 show | grep -iE "Firmware|Package"
   ```
3. Flash with the version check bypassed (required for a downgrade):
   ```bash
   sudo ./storcli64 /c0 download file=mr3108.rom noverchk
   ```
   - Add `force` **only if** `noverchk` alone is refused.
   - **Never add `nosigchk`** — the signature check is the safety net against a wrong-controller image.
   - Do **not** interrupt power during the flash.

### Phase 5 — Activate, verify, import
1. **Full AC power cycle** (not just a reboot): shut down, pull power, wait ~30 s, power on. A full AC cycle is required to activate the new image.
2. Boot the Live USB and verify drives/VD returned:
   ```bash
   sudo ./storcli64 /c0/eall/sall show
   sudo ./storcli64 /c0/vall show
   sudo ./storcli64 /c0/fall show
   ```
3. Expected outcomes:
   - Disks Online + VD auto-presents (Auto Enhanced Import is on) → boot to Windows.
   - Disks show **Foreign** → import non-destructively:
     ```bash
     sudo ./storcli64 /c0/fall import preview   # confirm it rebuilds the expected RAID5
     sudo ./storcli64 /c0/fall import
     ```
4. Boot Windows. If it does not boot even though the VD is healthy, check the server UEFI (F1) boot order still points at the RAID VD — a firmware flash can reshuffle it.

### Phase 6 — Fallback if direct rollback fails
The jump 24.21.0-0151 → 24.7.0-0056 is large and **not on Lenovo's verified rollback list**. If `storcli` errors or the flash succeeds but drives still don't return:
- Perform a **staged downgrade**: 24.21.0-0151 → intermediate (e.g. **24.16.0-0108**) → 24.7.0-0056, each via `storcli … noverchk` + power cycle.
- If healthy hardware still yields **no drives after rollback**, the cause is physical (backplane / the Port 4-7 mini-SAS cable / drives), not firmware — stop flashing and pursue reseat or professional data recovery.

---

## 7. Issues Encountered & How They Were Resolved

| # | Symptom / Error | Cause | Resolution |
|---|---|---|---|
| 1 | `storcli … download file=mr3108.rom` → *"The image file has older version than or same as that on the controller. The controller is not flashed"* | storcli version guard blocks downgrades | Add **`noverchk`** to bypass the version check |
| 2 | `lsiMRupdate.64 -a0 -list` / `-h` → *"Could not open controller info file"* | Wrong invocation; tool needs the info file as argument | Use `storcli` for rollback (readme confirms installer/`lsiMRupdate` cannot roll back) |
| 3 | Downloaded `sraidmr_1215` package | M1215 = SAS3008, a **different controller** | Use the **`sraidmr_5200`** (M5210/SAS3108) package containing `mr3108.rom` |
| 4 | Lenovo product page listed the M5200 package for **Windows only** | Legacy Linux build not surfaced in the filtered product view | Get the Linux `.bin` from IBM legacy pages, **or** extract identical `mr3108.rom` from the Windows `.exe` via `7z` |
| 5 | `chmod +x storcli64` → *"Operation not permitted"* | Binary on FAT/NTFS USB (no Unix perms) | Copy `storcli64` + ROM to a native FS (`~/fw`); often runs anyway due to mount-level exec perms |

---

## 8. Critical Do's and Don'ts

**Do**
- Flash from the Live Ubuntu **on the server that holds the controller**, using the **Linux** `storcli64`.
- Use **`noverchk`** for the downgrade; verify CacheVault/BBU healthy and no preserved cache first.
- Perform a **full AC power cycle** after flashing.
- Verify the ROM via `ctlr-info.txt` (SubDevice `0454` → `mr3108.rom`) before flashing.

**Don't**
- **Never** `initialize`, `clear foreign config`, or **create a new VD** while disks are absent or "Unconfigured Good" — this destroys recoverable RAID metadata.
- **Never** use `nosigchk`.
- Don't flash the **M1215** (`sraidmr_1215`) package onto the M5210.
- Don't interrupt power during a flash.

---

## 9. Command Reference

```bash
# Identify controller / state
sudo ./storcli64 /c0 show
sudo ./storcli64 /c0 show all | grep -iE "Status|Package|Firmware|CacheVault|Preserved"
lspci | grep -i raid
cat /proc/scsi/scsi ; sudo lsscsi -g

# Inspect drives / VD / foreign config
sudo ./storcli64 /c0/eall show
sudo ./storcli64 /c0/eall/sall show
sudo ./storcli64 /c0/vall show
sudo ./storcli64 /c0/fall show

# Extract ROM from Windows package (Linux)
7z x lnvgy_fw_sraidmr_5200-24.7.0-0056_windows_32-64.exe -o/tmp/m5210fw

# Flash rollback (downgrade)
sudo ./storcli64 /c0 download file=mr3108.rom noverchk      # add 'force' only if refused

# After full AC power cycle: re-verify and import if needed
sudo ./storcli64 /c0/eall/sall show
sudo ./storcli64 /c0/fall import preview
sudo ./storcli64 /c0/fall import
```

---

## 10. References

- ServeRAID M5200 Series firmware **24.7.0-0056** (Linux/VMware) — IBM legacy support page; file `ibm_fw_sraidmr_5200-24.7.0-0056_linux_32-64.bin`.
- ServeRAID M5200 Series firmware **24.21.0-0151** package contents (MR + iMR images).
- storcli `download … noverchk` — documented mechanism to flash a firmware version that is the same as or older than the one on the controller.

---

*End of document.*
