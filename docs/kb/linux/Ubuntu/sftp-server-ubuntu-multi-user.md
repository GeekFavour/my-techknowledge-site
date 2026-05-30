# Setting Up a Multi-User SFTP Server on Ubuntu with Port-Level Isolation

## Overview

This article explains how to configure a secure SFTP server on Ubuntu Linux that supports multiple users, each isolated to their own directory and accessible only through a dedicated port. This pattern is commonly used when providing file transfer access to different external parties, where isolation and least-privilege access are required.

Each user:

- Is restricted to a specific directory (chroot jail)
- Can only connect via SFTP (no shell access)
- Optionally connects through a unique port, reducing cross-user exposure
- Can be further restricted by source IP address or SSH key authentication

---

## Prerequisites

- Ubuntu Linux server with `openssh-server` installed
- `sudo` / root access
- Basic familiarity with Linux file permissions and `sshd_config`

---

## Architecture

The solution leverages OpenSSH's `Match User` directive inside `/etc/ssh/sshd_config` to apply per-user settings. Combined with `ChrootDirectory`, each user is locked into a specific folder and cannot navigate the broader filesystem.

For port-level isolation, the SSH daemon is configured to listen on multiple ports. Each user is then matched to a specific port using `Match LocalPort`, so even if credentials were compromised, the blast radius is limited to a single port and directory.

```
Server
├── Port 22    → (admin / standard SSH, if required)
├── Port 4444  → sftpuser1  →  /home/sftpuser1/upload/
└── Port 5555  → sftpuser2  →  /home/sftpuser2/upload/
```

---

## Step 1 — Create SFTP User Accounts

For each user that needs SFTP access, create a dedicated system account:

```bash
sudo adduser sftpuser1
```

You will be prompted to set a password and optionally fill in user details. User detail fields are optional — press `Enter` to skip them.

Repeat this for each user you need to onboard.

---

## Step 2 — Set Up the Upload Directory

Each user requires a dedicated upload directory. The directory structure must follow specific ownership rules for `ChrootDirectory` to work correctly.

```bash
# Create the upload directory
sudo mkdir -p /home/sftpuser1/upload

# Set the home directory owner to root (required for chroot)
sudo chown root:root /home/sftpuser1

# Give root write access; others get read + execute only
sudo chmod 755 /home/sftpuser1

# Give the user ownership of the upload subdirectory
sudo chown sftpuser1:sftpuser1 /home/sftpuser1/upload
```

!!! warning
    The `ChrootDirectory` itself **must** be owned by `root` and must not be writable by any other user. If this ownership is incorrect, SSH will refuse the chroot and the user will be unable to log in.

---

## Step 3 — Configure sshd_config

Open the SSH daemon configuration file:

```bash
sudo vim /etc/ssh/sshd_config
```

### 3.1 — Basic Single-User SFTP Restriction

Scroll to the bottom of the file and add a `Match User` block for each SFTP user:

```
Match User sftpuser1
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /home/sftpuser1
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

Repeat this block for each user, adjusting the username and `ChrootDirectory` path accordingly.

!!! info
    `ForceCommand internal-sftp` prevents the user from opening a shell session. Any SSH login attempt will return the message `This service allows sftp connections only.`

---

### 3.2 — Port-Level Isolation for Multiple Users

To restrict each user to a specific port, first declare all required ports near the top of the configuration file (around line 15, where the default `Port 22` entry typically appears):

```
Port 22
Port 4444
Port 5555
```

Then, at the bottom of the file, add `Match` blocks that combine username and port:

```
# User 1
Match User sftpuser1
    ForceCommand internal-sftp
    Match LocalPort 4444
    PasswordAuthentication yes
    ChrootDirectory /home/sftpuser1
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no

# User 2
Match User sftpuser2
    ForceCommand internal-sftp
    Match LocalPort 5555
    PasswordAuthentication yes
    ChrootDirectory /home/sftpuser2
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

!!! tip
    Using distinct ports per user allows firewall rules (e.g., cloud security groups or NSG rules) to be scoped to individual users. You can open only the port assigned to a specific user, rather than exposing a single shared port.

---

### 3.3 — Multiple Users Under a Shared Parent Directory

In cases where multiple SFTP users need their upload directories nested under a shared parent (for example, a common organizational directory), use the following structure:

```
Match User sftpuser1
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /srv/sftp/sftpuser1data
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no

Match User sftpuser2
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /srv/sftp/sftpuser2data
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

!!! warning
    All **parent directories** in the path must be owned by `root`. Only the final upload subdirectory inside each `ChrootDirectory` should be owned by the respective SFTP user. Incorrect ownership on any parent directory will prevent the chroot from working.

---

### 3.4 — Restricting Access by Source IP Address

To allow a user to connect only from a specific IP address, use `AllowUsers` with the `user@IP` notation inside the `Match User` block:

```
Match User sftpuser1
    AllowUsers sftpuser1@203.0.113.10
    ForceCommand internal-sftp
    ChrootDirectory /srv/sftp/sftpuser1data
    AllowTcpForwarding no
    X11Forwarding no
    PasswordAuthentication yes
```

Replace `203.0.113.10` with the actual source IP address that should be permitted.

!!! note
    You can list multiple `AllowUsers` lines within the same `Match` block to permit access from more than one IP address.

---

### 3.5 — SSH Key-Based Authentication

For stronger security, disable password authentication and require SSH key pairs instead:

```
Match User sftpuser1
    ForceCommand internal-sftp
    AuthorizedKeysFile /home/sftpuser1/.ssh/authorized_keys
    ChrootDirectory /srv/sftp/sftpuser1data
    AllowTcpForwarding no
    X11Forwarding no
    PasswordAuthentication no
    PubkeyAuthentication yes
```

!!! info
    After configuring this, place the user's **public key** into the `authorized_keys` file at the path specified in `AuthorizedKeysFile`. Ensure the file and directory have correct permissions:

    ```bash
    chmod 700 /home/sftpuser1/.ssh
    chmod 600 /home/sftpuser1/.ssh/authorized_keys
    chown -R sftpuser1:sftpuser1 /home/sftpuser1/.ssh
    ```

---

## Step 4 — Apply the Configuration

After editing `sshd_config`, restart the SSH service to apply changes:

```bash
sudo systemctl restart sshd
```

---

## Step 5 — Verify the Configuration

### Test: Shell Access Should Be Denied

Attempt a regular SSH login as the SFTP user:

```bash
ssh sftpuser1@<server-ip>
```

Expected output:

```
This service allows sftp connections only.
```

This confirms the user cannot open a shell session.

### Test: SFTP Access Should Work

Connect via the SFTP client:

```bash
sftp sftpuser1@<server-ip>
```

Expected output:

```
Connected to <server-ip>
sftp>
```

For port-restricted users, specify the port explicitly:

```bash
sftp -P 4444 sftpuser1@<server-ip>
```

---

## Firewall / Network Security Group Configuration

!!! warning
    When deploying on a cloud platform (e.g., Azure, AWS, GCP), you must open the required ports in the associated network security group or firewall rules attached to the server.

    - **Do not** open port 22 to all source addresses if it is not needed for all users.
    - Open only the specific port assigned to each user, and scope the source IP range to that user's known addresses where possible.
    - This ensures network-level isolation in addition to the SSH-level restrictions.

---

## Best Practices

| Practice | Reason |
|---|---|
| Use separate ports per user | Limits blast radius if credentials are compromised |
| Chroot each user to their own directory | Prevents cross-user file access |
| Disable shell access with `ForceCommand internal-sftp` | Reduces attack surface |
| Restrict source IPs where possible | Adds another layer of access control |
| Prefer SSH key authentication over passwords | Eliminates password-based brute force risk |
| Keep `ChrootDirectory` owned by root | Required for chroot to function correctly |
| Disable unnecessary SSH features | `PermitTunnel no`, `AllowTcpForwarding no`, `X11Forwarding no` reduce attack vectors |

---

## Troubleshooting

### User cannot log in at all

- Verify `ChrootDirectory` and all parent directories are owned by `root` with `755` permissions.
- Check the SSH service logs: `sudo journalctl -u sshd` or `sudo tail -f /var/log/auth.log`

### SFTP works on port 22 but not on custom port

- Confirm the custom port is declared in `sshd_config` with a `Port` directive.
- Ensure the port is open in the server's firewall and any cloud security group.
- Restart `sshd` after any configuration change.

### User can connect but cannot upload files

- Verify the upload subdirectory (inside `ChrootDirectory`) is owned by the user, not root.
- Check that write permissions are set on the subdirectory.

### SSH key authentication fails

- Confirm `authorized_keys` file permissions: `600` on the file, `700` on the `.ssh` directory.
- Confirm the file and directory are owned by the correct user.
- Check `AuthorizedKeysFile` path in `sshd_config` matches the actual file location.

---

## Summary

| Configuration | Use Case |
|---|---|
| Basic `Match User` + `ChrootDirectory` | Single user, simple SFTP isolation |
| Multiple `Port` + `Match LocalPort` | Per-user port isolation for multiple users |
| Shared parent directory with per-user subdirectories | Organizational directory layout |
| `AllowUsers user@IP` | Restrict to known source addresses |
| `PubkeyAuthentication yes` + `PasswordAuthentication no` | Key-based auth, no passwords |

This setup provides a robust, layered approach to SFTP access management — combining directory isolation, port segregation, IP restrictions, and authentication controls to meet a range of security requirements.
