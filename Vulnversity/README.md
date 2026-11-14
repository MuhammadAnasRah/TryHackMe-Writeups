# TryHackMe — Vulnversity

A learning-focused write-up of my work on the **Vulnversity** room from TryHackMe. The room gave me practical experience with web reconnaissance, file-upload weaknesses, Linux enumeration, and privilege-escalation concepts.

> **Lab scope:** This write-up describes work performed in the authorized TryHackMe environment. It is intended as a learning record, not as a guide for testing systems without permission. Exploit payloads, callback details, target-specific values, and flag strings are intentionally not reproduced.

## Room information

| Field | Details |
|---|---|
| Platform | TryHackMe |
| Room | [Vulnversity](https://tryhackme.com/room/vulnversity) |
| Difficulty | Easy |
| Category | Linux |
| Author | Muhammad Anas Rah |

## Objective

The room focuses on three areas:

- Active reconnaissance
- Web application testing
- Linux privilege escalation

My main goal was to understand how information gathered during enumeration can reveal weaknesses in a web application and how Linux permission misconfigurations can affect system security.

## Environment

| Item | Details |
|---|---|
| Attacking machine | Kali Linux |
| Lab network | TryHackMe VPN |
| Tools recorded | Nmap, Gobuster, Netcat |
| | Burp Suite |

The target address was assigned dynamically by the TryHackMe lab, so it is not included here.

## 1. Connecting to the lab

I connected to the TryHackMe network using the provided VPN configuration and started the Vulnversity machine. I then checked connectivity to confirm that my machine could reach the lab target.

This was a useful reminder to verify the lab connection before beginning enumeration.

## 2. Reconnaissance and enumeration

### Network scanning

I used **Nmap** to identify reachable services and collect service-version information.

The scan reported six open ports. The services identified in my notes included FTP, SSH, HTTP, a web proxy, and NetBIOS-related services.

This stage helped me build an initial picture of the target and decide which services needed further investigation.

### Web directory discovery

I used **Gobuster** to enumerate directories on the web service. This revealed an `internal` directory containing a file-upload feature.

That discovery changed the direction of my investigation: instead of treating the website as a simple static page, I examined how it handled uploaded files.

## 3. Vulnerability analysis

The upload feature accepted a file type that could be interpreted by the server as executable, rather than treating every upload as a passive document.

The important security issue was not simply that uploads were allowed. It was that file validation and server-side handling could allow an uploaded file to be processed in an unsafe way.

**Security lesson:** File-upload functionality should use strict allow-lists, verify file content rather than relying only on extensions, store uploads outside executable web directories, and apply least-privilege permissions.

## 4. Initial access — high-level summary

Within the TryHackMe lab, I tested the upload weakness and obtained an initial shell. I used a listener during the exercise to observe the connection.

The important learning point was understanding how an unsafe upload configuration can become a route to command execution, and why upload handling needs to be treated as a security-sensitive feature.

## 5. Linux enumeration and privilege escalation

After obtaining the initial foothold, I explored the available user environment and identified the account `bill`.

I then reviewed file permissions and investigated files with the SUID permission bit. During this process, I identified an unusual SUID configuration involving `systemctl`.

In the context of the lab, this misconfiguration provided a path to elevated privileges. I studied how the service configuration was involved and completed the room's privilege-escalation objective.


### Defensive takeaways

- Audit SUID files and confirm that each one genuinely requires elevated privileges.
- Pay particular attention to unusual or custom permission changes.
- Restrict who can create, modify, enable, or start privileged services.
- Keep system components updated and remove unnecessary privileges.
- Review logs and investigate unexpected service activity.

## 6. Challenges and how I approached them

| Challenge | How I approached it |
|---|---|
| Finding the relevant upload location | Continued web enumeration after the initial network scan. |
| Understanding the upload weakness | Considered how the server handled the uploaded file, rather than focusing only on the file extension. |
| Identifying the privilege-escalation path | Reviewed the user's environment and examined SUID permissions. |
| Understanding the service-related misconfiguration | Researched the relevant Linux concepts and adapted my understanding to the authorized lab. |


## 7. Key learnings

This room helped me practise:

- Network reconnaissance and service identification
- Web directory enumeration
- Recognizing risky file-upload configurations
- Linux user and permission enumeration
- Understanding SUID permissions
- Recognizing the security impact of privileged service management

The most valuable part for me was learning about SUID-related privilege escalation. Reconnaissance was already familiar, but the Linux permission and service-management concepts were newer, and I had to spend additional time understanding how they fit together.

## 8. Conclusion

Vulnversity was a useful introduction to the relationship between web application weaknesses and Linux privilege escalation. It reinforced the importance of thorough enumeration: the information gathered at each stage helped guide the next part of the investigation.

I finished the room with a clearer understanding of unsafe file uploads, Linux permissions, and why privileged executables need careful configuration. It also showed me areas I want to study further, especially Linux privilege management and secure service configuration.

## Tools and references

**Tools recorded during the room**

- [Nmap](https://nmap.org/)
- [Gobuster](https://github.com/OJ/gobuster)
- [Netcat](https://netcat.sourceforge.net/)
- Kali Linux
- Burp Suite (listed in my environment notes)

**References**

- [TryHackMe — Vulnversity](https://tryhackme.com/room/vulnversity)
- [Reference material consulted during the lab](https://gist.github.com/A1vinSmith/78786df7899a840ec43c5ddecb6a4740)

---


