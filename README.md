# 🔍 Lab 5 — Vulnerability Scanning with Nessus Essentials

Finding weaknesses before an attacker does. This lab walks through standing up Nessus Essentials, running unauthenticated and credentialed scans against a lab server, interpreting CVSS-scored findings, remediating one, and proving the fix worked.

![Nessus](https://img.shields.io/badge/Tenable-Nessus_Essentials-1E88E5)
![Cost](https://img.shields.io/badge/Cost-%240-brightgreen)
![Status](https://img.shields.io/badge/Status-Complete-success)

## 🎥 Demo Video
[Watch the full scan-to-remediation walkthrough →](PASTE_YOUR_LINK_HERE)

---

## Quick Facts

| | |
|---|---|
| **Tool** | Nessus Essentials — free, up to 5 IPs, no credit card |
| **Certs this maps to** | Security+ · CySA+ · PenTest+ |
| **Time invested** | ~3–4 hours |
| **Cost** | $0 |
| **Roles this proves out** | Vulnerability Analyst, Security Engineer, SOC Analyst, Cloud Security Engineer |

## Why This Lab Exists

Every network has vulnerabilities. The real question is whether the security team finds them before someone else does. Vulnerability management is the discipline of continuously finding, scoring, prioritizing, and closing security gaps — and Nessus is the tool most security teams reach for first to do it.

Most junior candidates have *heard* of Nessus. Very few have actually run a scan, read a CVSS score, and closed a finding. This lab is that gap, closed.

**Where this shows up on the job:**
- **Vulnerability Analyst** — scanning, triage, and remediation tracking *is* the role
- **Security Engineer** — CVSS scoring and severity triage drive infrastructure decisions
- **SOC Analyst** — a host with known CVEs is a higher-priority alert
- **Cloud Security Engineer** — this is the same mental model behind Defender for Cloud and AWS Inspector

## The Workflow This Lab Teaches

```
  SCAN  →  FIND  →  SCORE  →  PRIORITIZE  →  REMEDIATE  →  VERIFY
   │                                                          │
   └──────────────────── repeat continuously ◄────────────────┘
```

This loop — not any single scan — is what a real vulnerability management program looks like. This lab runs it once, start to finish, against a lab VM.

---

## Setup Walkthrough

### Phase 1 — Install Nessus Essentials

1. Go to `tenable.com/products/nessus/nessus-essentials`
2. Click **Get Started for Free**
3. Enter your name and email (no credit card needed)
4. Check your inbox for an activation code — save it
5. Download the Nessus installer for your OS

| OS | Install command |
|---|---|
| Windows | Run the `.exe` — installs as a Windows service |
| Ubuntu/Debian | `sudo dpkg -i Nessus-10.x.x-ubuntu1404_amd64.deb && sudo systemctl start nessusd` |
| RHEL/CentOS | `sudo rpm -ivh Nessus-10.x.x-el9.x86_64.rpm && sudo systemctl start nessusd` |
| macOS | Open the `.dmg`, drag to Applications, launch from System Preferences |

6. Open `https://localhost:8834`
7. Choose **Nessus Essentials**, enter your activation code
8. Create an admin username and password
9. Wait 10–20 minutes for the initial plugin download to finish

> Nessus Essentials covers 5 IPs — plenty for a 2–3 VM home lab. Students get 20 IPs free for a year via `academic.tenable.com`.

### Phase 2 — Run an Unauthenticated Discovery Scan

This is what an outside attacker sees: open ports and exposed services, no login required.

10. **New Scan → Basic Network Scan**
11. Name it `Lab Network Discovery`, target your Lab 1 Windows Server's IP (e.g. `10.0.1.4`)
12. **Save**, then hit the play button to launch
13. Let it run 5–10 minutes
14. Click into the scan to watch results populate

> ⚠️ **Only scan systems you own.** Nessus traffic looks identical to an attack from a monitoring tool's perspective. Never scan anything without explicit written permission.

### Phase 3 — Run a Credentialed Scan

This is the real test — logging into the target and inspecting it from the inside. Expect 5–10x more findings than the unauthenticated scan.

15. **New Scan → Basic Network Scan**, name it `Lab Windows Server — Credentialed`
16. Target: your Windows Server VM's IP
17. **Credentials tab → Add → Windows → Password**, enter the admin username, password, and domain (`LAB`)
18. On the Windows Server, enable remote registry access first:

```powershell
# Run in PowerShell on the Windows Server
Set-Service -Name RemoteRegistry -StartupType Automatic
Start-Service RemoteRegistry

# If Nessus runs on a separate machine, also allow it through the firewall:
netsh advfirewall firewall add rule name='Nessus' dir=in action=allow protocol=tcp localport=445
```

19. Launch the scan — expect 15–20 minutes

### Phase 4 — Read the Findings

Every finding breaks down into: **Synopsis**, **Description**, **Solution**, **CVE ID**, **CVSS Score**, **Risk Factor**, and **Plugin Output** (the actual evidence).

**How severity maps to CVSS:**

| Severity | CVSS | What it means | Example |
|---|---|---|---|
| 🔴 Critical | 9.0–10.0 | Remotely exploitable, little/no auth needed — fix now | EternalBlue (MS17-010) |
| 🟠 High | 7.0–8.9 | Serious impact if exploited — fix in 7–14 days | Unpatched RDP vuln |
| 🟡 Medium | 4.0–6.9 | Needs specific conditions to exploit — fix in 30 days | Expired SSL cert, weak cipher |
| 🟢 Low | 0.1–3.9 | Minor impact — fix in routine patch cycles | Missing security headers |
| ⚪ Info | 0 | Not a vulnerability — just system detail | Open port, OS fingerprint |

### Phase 5 — Remediate One Finding and Prove It

20. Pick one Medium or High finding
21. Follow its **Solution** field exactly

**Common Windows remediations found in this lab:**

```powershell
# Install pending Windows updates
Install-Module PSWindowsUpdate -Force
Get-WindowsUpdate -Install -AcceptAll

# Disable TLS 1.0 (frequently flagged)
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Force
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Name 'Enabled' -Value 0 -PropertyType DWORD -Force

# Disable SMBv1 (frequently flagged)
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

22. Re-run the credentialed scan
23. Confirm the finding is gone

This re-scan step — proving the fix actually worked instead of just assuming it did — is the part most people skip and the part that matters most.

### Phase 6 — Export a Report

24. Open the completed scan → **Report → PDF**
25. Choose **Executive Summary** (for stakeholders) or **Detailed Vulnerabilities** (for remediation tracking)
26. **Generate Report**

---

## What I Actually Did

- Installed Nessus Essentials and activated it against a lab VM environment
- Ran an unauthenticated discovery scan against the Lab 1 Windows Server to see what's exposed externally
- Ran a credentialed scan using local admin credentials, enabling Remote Registry and the required firewall rule first
- Compared finding counts between the two scans to see the authenticated-vs-unauthenticated gap firsthand
- Triaged findings by CVSS score and picked one to remediate
- Applied the fix, re-scanned, and confirmed the finding cleared
- Exported both an executive summary and a detailed technical report

## Skills This Proves

| Skill | Why it matters on the job |
|---|---|
| Deploying and configuring a scanner | The architecture — scanner, policy, targets — underlies every enterprise vuln program |
| Running unauthenticated scans | Shows what's visible from outside, the attacker's view |
| Running credentialed scans | The real standard for internal vulnerability management |
| Reading CVSS scores | The universal severity language used in every security conversation |
| Prioritizing findings | Not every "Critical" is equally urgent — exploitability and asset value matter |
| Running the remediate → verify loop | The actual discipline of vulnerability management, not just finding problems |
| Exporting reports for different audiences | Executives want risk. Engineers want CVE IDs. Knowing which to hand over is a real skill |

## Verification Checklist

- [x] Discovery scan completed with at least Info-level findings (even a hardened box shows something)
- [x] Credentialed scan returned meaningfully more findings than the unauthenticated scan
- [x] Remote Registry and firewall prerequisites confirmed before the credentialed scan
- [x] At least one finding remediated and confirmed gone on re-scan
- [x] PDF report generated and reviewed

## Field Notes

- **Credentialed scans are the whole point.** The unauthenticated scan barely scratches the surface — if you only run that, you're not doing vulnerability management, you're doing reconnaissance.
- **Remote Registry has to be running *before* the scan**, not something you fix after a failed credentialed scan comes back thin.
- **Re-scanning after remediation is not optional.** A finding you "fixed" and never re-verified is still an open finding — the whole workflow hinges on proof, not intent.
- **Severity ≠ urgency by itself.** A Critical finding on an isolated test box is lower priority than a High on something internet-facing. CVSS gives you severity; you still have to apply context.

## Related Labs

- **Lab 1** — Active Directory deployment (provides the Windows Server VM scanned in this lab)
- **Lab 3** — Splunk SIEM & Log Analysis
