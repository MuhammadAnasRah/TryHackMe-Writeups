# TryHackMe — Simple CTF

A walkthrough of the **Simple CTF** room on TryHackMe: enumerating a web server with Gobuster, finding a vulnerable CMS Made Simple install, exploiting a blind SQL injection (CVE-2019-9053) to crack a user's password, SSHing in, and escalating to root through a GTFOBins misconfiguration in `vim`.

| | |
|---|---|
| **Platform** | TryHackMe |
| **Room** | [Simple CTF](https://tryhackme.com/room/easyctf) |
| **Difficulty** | Easy |
| **Category** | Linux |
| **Author** | MuhammadAnasRah |
| **Target IP** | `10.130.144.64` |
| **Attack box** | Kali Linux |
| **Connection** | TryHackMe OpenVPN |

---

## Table of Contents

- [Objective](#objective)
- [Setup](#setup)
- [Reconnaissance & Enumeration](#reconnaissance--enumeration)
- [Vulnerability Analysis](#vulnerability-analysis)
- [Exploitation](#exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flags](#flags)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)
- [References](#references)

---

## Objective

Get full root access on a Linux target by chaining together three separate, low-effort mistakes: an exposed admin directory, an outdated CMS with a known SQL injection, and a sudo rule that hands a regular user a root shell through a text editor.

## Setup

Connected to TryHackMe's OpenVPN, then confirmed the tunnel and target were reachable before scanning anything:

```bash
sudo openvpn <your-connection-file>.ovpn
ping -c 4 10.130.144.64
```

```
PING 10.130.144.64 (10.130.144.64) 56(84) bytes of data.
64 bytes from 10.130.144.64: icmp_seq=1 ttl=62 time=141 ms
64 bytes from 10.130.144.64: icmp_seq=2 ttl=62 time=142 ms
64 bytes from 10.130.144.64: icmp_seq=3 ttl=62 time=140 ms
64 bytes from 10.130.144.64: icmp_seq=4 ttl=62 time=139 ms

--- 10.130.144.64 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
```

## Reconnaissance & Enumeration

### Service scan

```bash
nmap -sV 10.130.144.64
```

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

![Nmap service scan results](screenshots/01-nmap-scan.png)

Three open ports, three different angles to look at:

- **21 (FTP)** — vsftpd 3.0.3, worth checking for anonymous login, but not where this room's path went.
- **80 (HTTP)** — Apache on Ubuntu, the obvious place to start digging.
- **2222 (SSH)** — not the default port, but still SSH, and the thing I'd eventually need valid credentials for.

### Directory enumeration

With HTTP open, the next step was finding out what was actually hosted on the webserver beyond the default page:

```bash
gobuster dir -u http://10.130.144.64 -w /usr/share/wordlists/dirb/common.txt -x html,php,txt,bak -t 20
```

That turned up a `/simple` directory, which resolved to a **CMS Made Simple** installation — a full content management system, not just a static page.

## Vulnerability Analysis

| Field | Details |
|---|---|
| **Vulnerability** | Unauthenticated, time-based blind SQL injection |
| **Affected service** | CMS Made Simple ≤ 2.2.9 |
| **CVE** | [CVE-2019-9053](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9053) |
| **Impact** | Leaks the admin's username, email, password hash, and salt straight out of the database — no authentication needed |

A quick search for the CMS name and version number led to a public exploit script ([Exploit-DB #46635](https://www.exploit-db.com/exploits/46635)) targeting exactly this CVE. The script abuses a `moduleinterface.php` endpoint that's vulnerable to a boolean/time-based blind injection: it sends a request with a conditional `SLEEP()` in the query and measures the response time, character by character, to blindly read data out of the `cms_users` table it has no business reading.

## Exploitation

Ran the exploit against the CMS install, pointing it at a small password wordlist to crack the hash it extracts along the way:

```bash
python3 exploit.py -u http://10.130.144.64/simple --crack -w common.txt
```

The script works in four stages — each one brute-forcing one character at a time by timing the server's response:

1. `dump_salt()` — recovers the password salt from `cms_siteprefs`
2. `dump_username()` — recovers the admin's username from `cms_users`
3. `dump_email()` — recovers the admin's email address
4. `dump_password()` — recovers the salted password hash

Once it has the hash, `--crack` runs it against the supplied wordlist (`common.txt` — a short list of the internet's favorite terrible passwords) and reports back the cracked plaintext.

That gave me a working set of credentials for the user **mitch**.

### Logging in

```bash
ssh -p 2222 mitch@10.130.144.64
```

My first attempt used a mistyped username and was rejected outright — included below because that's genuinely what the terminal shows:

```
$ ssh -p 2222 mi9@10.130.144.64
mi9@10.130.144.64's password:
Permission denied, please try again.
mi9@10.130.144.64's password:
Permission denied, please try again.
mi9@10.130.144.64: Permission denied (publickey,password).

$ ssh -p 2222 mitch@10.130.144.64
mitch@10.130.144.64's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)
```

![SSH login as mitch after cracking the CMS password](screenshots/02-ssh-login-mitch.png)

Once in, `pwd` confirmed the landing spot and a quick check of the home directory turned up the user flag.

## Privilege Escalation

With a foothold as `mitch`, the standard first move is checking what that account is allowed to run as root:

```bash
sudo -l
```

```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

A completely unrestricted, password-free `sudo vim`. That's not a minor convenience — it's a direct route to a root shell, and it's exactly the kind of entry [GTFOBins](https://gtfobins.github.io/gtfobins/vim/) maintains a page for. I checked the binary itself first out of curiosity:

```bash
ls -l /usr/bin/vim
# -> symlink to /etc/alternatives/vim -> /usr/bin/vim.basic
```

...then went straight for the GTFOBins technique for vim's sudo escape:

```bash
sudo vim -c '!sh'
```

```
# whoami
root
```

![Escalating to root via sudo vim -c '!sh'](screenshots/03-gtfobins-vim-root.png)

`vim` can shell out to run arbitrary commands via `:!<command>` — and because it was running as root under `sudo` with no password required, that shell-out inherits root. One line, no password, full system access.

## Flags

| Flag | Location | Status |
|---|---|---|
| User flag | `/home/mitch/user.txt` | ✅ Captured (value not published — see note below) |
| Root flag | `/root/root.txt` | ✅ Captured (value not published — see note below) |

> I'm intentionally not publishing the actual flag strings here. Both were confirmed and read during the exploit, but I'd rather keep this writeup useful as a *methodology* reference than hand out answer keys for a room other people are still working through.

## Key Takeaways

- **A single outdated CMS can undo an otherwise fine server.** The underlying box — vsftpd, Apache, OpenSSH — wasn't the problem. One neglected web application sitting in a subdirectory was enough to leak admin credentials.
- **Exploit-DB and public GitHub repos are a real part of the workflow, not a shortcut that skips learning.** Finding CVE-2019-9053, understanding *why* it works (a classic unsanitized parameter feeding straight into a SQL query), and adapting the public script to this target taught me more than writing the injection from scratch would have at this stage.
- **`sudo -l` should be the first thing you run on any new shell.** It costs nothing and immediately tells you whether privilege escalation is going to be a five-minute problem or a much longer one.
- **GTFOBins is essential, not optional, knowledge.** A huge number of "easy" privilege escalations on Linux come down to a single binary with sudo rights that was never meant to be treated as a root shell — vim, find, less, awk, and dozens of others all have the same failure mode.

## Tools Used

- [Nmap](https://nmap.org/) — service enumeration
- [Gobuster](https://github.com/OJ/gobuster) — directory/content discovery
- [Exploit-DB #46635](https://www.exploit-db.com/exploits/46635) (`exploit.py`) — CVE-2019-9053 blind SQL injection exploit
- [GTFOBins](https://gtfobins.github.io/) — privilege escalation reference for the `vim` sudo rule

## References

- [TryHackMe — Simple CTF](https://tryhackme.com/room/easyctf)
- [CVE-2019-9053](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9053)
- [CMS Made Simple — Official Site](https://www.cmsmadesimple.org/)
- [GTFOBins — vim](https://gtfobins.github.io/gtfobins/vim/)

---



