# Lab 5 — Vulnerability Scanning with Nessus Essentials

Standing up Tenable Nessus, scanning a lab server from both an outsider's and an insider's perspective, scoring what gets found, fixing one issue, and proving the fix actually stuck.

![Nessus](https://img.shields.io/badge/Tenable-Nessus_Essentials-1E88E5)
![Cost](https://img.shields.io/badge/Cost-%240-brightgreen)
![Status](https://img.shields.io/badge/Status-Complete-success)

## 🎥 Demo Video
[Watch me run this lab →](https://www.loom.com/share/efaef436db174650b8ff296b4827adcd)

## Overview

| Field | Value |
|---|---|
| Certification alignment | Security+ · CySA+ · PenTest+ |
| Tools used | Tenable Nessus Essentials (free, up to 5 IPs) |
| Environment | Azure VM — Ubuntu 24.04 LTS, Standard_B2as_v2 |
| Time invested | 3–4 hours |
| Cost | $0 |
| Career relevance | Vulnerability Analyst, Security Engineer, SOC Analyst, Cloud Security Engineer |

## Why This Matters

No network is fully secure — the real differentiator is whether an organization finds its own weak points before someone outside does. That's the entire discipline of vulnerability management: continuously identifying, scoring, and closing gaps before they turn into incidents.

Nessus is the tool most security teams reach for to do exactly that. It's one of the most recognized names in the industry, yet a surprising number of people entering the field have only heard about it in passing rather than actually run a scan themselves. This lab closes that gap — going through the full lifecycle of finding a weakness, understanding how serious it actually is, fixing it, and confirming the fix worked.

**How this connects to real roles:**
- A **Vulnerability Analyst** spends their entire day scanning, triaging results, and tracking fixes — this lab is that job in miniature
- A **Security Engineer** uses CVSS scoring to decide what gets patched first and what can wait
- A **SOC Analyst** treats a host with known unpatched vulnerabilities as a higher-priority alert the moment it shows up in a case
- A **Cloud Security Engineer** applies this same logic inside tools like Microsoft Defender for Cloud or AWS Inspector — Nessus just teaches the underlying concept without the cloud abstraction layer in the way

## How This Lab Works

![Diagram showing Nessus Essentials running unauthenticated and credentialed scans against a Windows Server VM, producing findings scored by CVSS, which get remediated and re-scanned to verify the fix](./screenshots/nessus-lab-diagram.svg)

## The Bigger Picture

Vulnerability management isn't a single scan — it's a loop that never really stops:

```
Scan the environment → Surface findings → Score by severity →
Decide what matters most → Fix it → Re-scan to confirm → repeat
```

This lab runs that loop once, start to finish, on a single lab machine — but it's the same cycle a security team runs continuously across an entire company.

---

## Step-by-Step Setup

### The environment

This lab runs on an **Azure Ubuntu 24.04 LTS VM**, sized **Standard_B2as_v2** — enough CPU and memory headroom to run Nessus comfortably without the scanner itself becoming the bottleneck during a credentialed scan.

### Getting Nessus running

1. Head to `tenable.com/products/nessus/nessus-essentials` and click **Get Started for Free**
2. Enter a name and email — no payment information required at any point
3. An activation code lands in your inbox — hang onto it, you'll need it during setup

**Installing on the Azure Ubuntu VM via the command line:**

```bash
curl --request GET \
  --url 'https://www.tenable.com/downloads/api/v2/pages/nessus/files/Nessus-10.12.4-ubuntu1604_amd64.deb' \
  --output 'Nessus-10.12.4-ubuntu1604_amd64.deb'

sudo dpkg -i Nessus-10.12.4-ubuntu1604_amd64.deb
```

The `curl` command pulls the `.deb` package straight from Tenable's download API rather than going through a browser, which is the more natural path when you're already SSH'd into a headless Ubuntu VM. `dpkg -i` then installs it directly — no desktop environment needed on the VM itself, since the Nessus web UI is what you'll actually interact with afterward.

> If this specific version URL 404s, Tenable has likely shipped a newer release — grab the current `wget`/`curl` command from the Nessus download page and substitute it in.

4. Alternative install methods for other platforms:

| Platform | How to install |
|---|---|
| Windows | Run the `.exe` — it installs and runs as a background service |
| RHEL/CentOS | `sudo rpm -ivh Nessus-10.x.x-el9.x86_64.rpm && sudo systemctl start nessusd` |
| macOS | Open the `.dmg`, drag Nessus into Applications, launch it from System Preferences |

5. Start the Nessus service and browse to `https://<VM-IP-or-localhost>:8834` — Nessus runs over this port
6. Pick **Nessus Essentials**, drop in the activation code from your email
7. Set an admin username and password
8. Sit tight for 10–20 minutes while the plugin library downloads on first launch

> The free tier covers 5 IP addresses, which comfortably covers a small home lab. Students can get a year of Nessus Essentials Plus (20 IPs) at no cost through `academic.tenable.com`.

### Scanning without credentials — the outsider's view

This first scan shows exactly what someone with zero access could see just by probing the network — open ports, exposed services, nothing more.

9. **New Scan → Basic Network Scan**
10. Call it something like `Lab Network Discovery`, point it at the Windows Server VM's IP from Lab 1 (something like `10.0.1.4`)
11. Hit **Save**, then press play to kick it off
12. Give it 5–10 minutes to finish
13. Open the scan to watch results come in live

> ⚠️ Only ever point Nessus at systems you own or have written permission to test. To a network monitoring tool, a Nessus scan is indistinguishable from an actual attack in progress.

### Scanning with credentials — the insider's view

This is where the real value shows up. Logging in during the scan lets Nessus inspect the system from within — patch levels, installed software, registry misconfigurations — and it typically surfaces 5 to 10 times more findings than the outside-only scan.

14. **New Scan → Basic Network Scan** again, this time named something like `Lab Windows Server — Credentialed`
15. Target the same Windows Server VM IP
16. Go to the **Credentials** tab → **Add → Windows → Password**, and supply the admin username, password, and the `LAB` domain name
17. Before launching, flip on Remote Registry on the target machine:

```powershell
# Run this in PowerShell on the Windows Server
Set-Service -Name RemoteRegistry -StartupType Automatic
Start-Service RemoteRegistry

# If Nessus lives on a different machine, open the firewall for it too:
netsh advfirewall firewall add rule name='Nessus' dir=in action=allow protocol=tcp localport=445
```

18. Kick off the scan — this one takes longer, plan for 15–20 minutes

### Making sense of the results

Every finding Nessus surfaces comes with a **Synopsis**, a fuller **Description**, a **Solution**, a **CVE identifier**, a **CVSS score**, a **Risk Factor**, and raw **Plugin Output** as evidence.

**What the severity scale actually means:**

| Severity | CVSS Score | Translation | Real example |
|---|---|---|---|
| Critical | 9.0–10.0 | Exploitable remotely with barely any effort — drop everything | EternalBlue (MS17-010) |
| High | 7.0–8.9 | Serious if exploited — fix within one to two weeks | Unpatched RDP flaw |
| Medium | 4.0–6.9 | Needs specific conditions to be exploitable — fix within a month | Expired SSL cert, weak cipher suite |
| Low | 0.1–3.9 | Minor exposure — bundle into normal patch cycles | Missing security headers |
| Info | 0 | Not actually a vulnerability, just system detail | Detected open port, OS fingerprint |

### Closing the loop — fixing something and proving it

19. Choose one Medium or High finding to actually resolve
20. Apply exactly what the **Solution** field recommends

**A few fixes that come up constantly on a Windows target:**

```powershell
# Apply pending Windows updates
Install-Module PSWindowsUpdate -Force
Get-WindowsUpdate -Install -AcceptAll

# Turn off TLS 1.0 — one of the most common findings
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Force
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Name 'Enabled' -Value 0 -PropertyType DWORD -Force

# Turn off SMBv1 — also flagged constantly
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

21. Re-run the credentialed scan
22. Confirm the finding you fixed is actually gone from the results

Skipping this last step is the most common shortcut people take — and it's the one that matters most. A fix nobody verified is just a guess.

### Documenting the results

23. Open the finished scan — you can either export a report (**Report → PDF**, choosing Executive Summary or Detailed Vulnerabilities) or simply screenshot the key before/after states
24. If the PDF export isn't cooperating, screenshots of the scan results at each stage tell the same story just as clearly — a discovery scan, a credentialed scan with findings, one finding resolved, and a final clean scan cover the full lifecycle without needing the export feature at all

---

## Screenshots — The Remediation Progression

*(Images live in a `/screenshots` folder in this repo — see the note at the bottom for how to wire them up.)*

**1. Unauthenticated scan — the outsider's view**
![Unauthenticated discovery scan results](./screenshots/01-unauthenticated-scan.png)
> **Takeaway:** With no credentials, the scan surfaced only 8 findings — all Medium or Info, mostly SSL certificate issues (self-signed cert, untrusted cert chain, hostname mismatch). This is genuinely all an outside attacker with zero access would see.

**2. Credentialed scan kicking off**
![Credentialed scan in progress](./screenshots/02-credentialed-scan-in-progress.png)
> **Takeaway:** Same target, this time authenticated with local admin credentials over SMB. The scan takes noticeably longer than the unauthenticated pass because Nessus is now inspecting the system from the inside — patch levels, installed software, registry state — not just probing from outside.

**3. Credentialed results — two severe findings surface**
![Credentialed scan results showing two High severity findings](./screenshots/03-credentialed-scan-two-severe-findings.png)
> **Takeaway:** The finding count jumped to 62 — nearly 8x the unauthenticated scan — and two High-severity vulnerabilities appeared that were completely invisible without credentials: **WinVerifyTrust Signature Validation (CVE-2013-3900)** at CVSS 8.8, and a **Windows Package Manager elevation-of-privilege flaw (CVE-2026-68821)** at CVSS 7.3. This is the clearest proof in the whole lab of why credentialed scanning is the real standard.

**4. Full vulnerability list — my executive summary substitute**
![Full list of scanned vulnerabilities standing in for the executive summary export](./screenshots/04-full-vulnerability-list-exec-summary-substitute.png)
> **Takeaway:** Nessus Essentials doesn't support PDF export, so this full vulnerability breakdown stands in as the executive-summary equivalent — the complete picture of severity distribution and finding families from the initial credentialed scan, in one view.

**5. First remediation — one severe finding resolved**
![Scan results after remediating the first severe finding](./screenshots/05-first-remediation-one-severe-resolved.png)
> **Takeaway:** After patching the WinVerifyTrust signature validation issue and re-scanning, the finding count dropped to 60 and that CVE no longer appears. The Windows Package Manager High-severity finding is still present — one down, one to go.

**6. Final verification scan in progress**
![Final verification scan running after remediating all findings](./screenshots/06-final-verification-scan-in-progress.png)
> **Takeaway:** Re-running the scan one more time after resolving the second finding (the WinGet elevation-of-privilege issue) to confirm both severe vulnerabilities are actually gone — not just assumed fixed.

**7. Final scan — both severe findings resolved**
![Final scan results with zero High or Critical findings remaining](./screenshots/07-final-scan-all-severe-resolved.png)
> **Takeaway:** Down to 59 findings, with zero Critical or High severity items remaining — only routine Info-level and a handful of Medium SSL findings left. This closes the loop: both vulnerabilities found, fixed, and independently re-verified.

**How to add these when ready:**
1. Create a `screenshots` folder in the repo root (same approach as the other labs)
2. Upload the seven images using the filenames above
3. Replace your `README.md` with this version — the image links already match

## What I Actually Did in This Lab

- Registered for Nessus Essentials and got it running on an Azure Ubuntu VM
- Ran an unauthenticated scan against the lab VM, returning only 8 low-severity findings — mostly SSL certificate issues
- Ran a credentialed scan with local admin access, which returned 62 findings — nearly 8x the unauthenticated result
- Identified two High-severity vulnerabilities that only appeared once authenticated: WinVerifyTrust Signature Validation (CVE-2013-3900, CVSS 8.8) and a Windows Package Manager elevation-of-privilege flaw (CVE-2026-68821, CVSS 7.3)
- Remediated the WinVerifyTrust finding first, re-scanned, and confirmed it cleared while the second finding remained
- Remediated the Windows Package Manager finding, ran a final verification scan, and confirmed both severe findings were resolved
- Documented the full progression with screenshots at each stage, since PDF export isn't available in Nessus Essentials

## Skills This Demonstrates

| Skill | Why it's relevant on the job |
|---|---|
| Standing up and configuring a scanner | Every enterprise vulnerability program is built on this same scanner/policy/target architecture |
| Running unauthenticated scans | Shows the outside-in perspective an attacker actually has |
| Running credentialed scans | The real standard for meaningful internal vulnerability coverage |
| Reading and applying CVSS scores | The common language every security conversation uses to talk about severity |
| Prioritizing findings | A high score on an isolated test box matters less than a lower score on something internet-facing |
| Running the fix-then-verify loop | The actual discipline behind vulnerability management, not just the scanning part |
| Producing audience-appropriate reports | Leadership wants risk in plain terms; engineers want the CVE and the fix |

## How I Verified It Worked

| Check | Result |
|---|---|
| Discovery scan completed | Returned 8 findings — all Medium/Info, mostly SSL certificate issues |
| Credentialed scan completed | Returned 62 findings, including 2 High-severity vulnerabilities invisible in the unauthenticated pass |
| First remediation verified | Re-scan showed 60 findings, WinVerifyTrust (CVE-2013-3900) resolved, WinGet finding still present |
| Second remediation verified | Final scan showed 59 findings, zero Critical/High remaining |
| Documentation | Full progression captured in screenshots, from initial scan to clean final scan |

## Takeaways

- **The credentialed scan is where the real work happens.** An unauthenticated scan alone tells you almost nothing — it's reconnaissance, not vulnerability management.
- **Remote Registry has to be running before you scan**, not something you troubleshoot after a suspiciously thin result set.
- **A fix you don't re-verify isn't actually a fix** — it's an assumption. The re-scan is what turns "I think I patched it" into "I confirmed it's patched."
- **Severity and urgency aren't the same thing.** A Critical on an isolated test box can wait longer than a High on something exposed to the internet — CVSS gives you the starting point, context does the rest.

## Lab Summary

I scanned a Windows VM on my Azure lab network, first without credentials and then with local admin credentials over SMB, to see the gap between an outsider's view and a real internal assessment. The unauthenticated scan returned only 8 low-impact findings, but the credentialed scan surfaced 62 — including two High-severity vulnerabilities that were completely invisible without authentication: a WinVerifyTrust signature validation flaw (CVE-2013-3900, CVSS 8.8) and a Windows Package Manager elevation-of-privilege issue (CVE-2026-68821, CVSS 7.3). I remediated both, one at a time, re-scanning after each fix to confirm it actually resolved — first watching the WinVerifyTrust finding disappear, then the WinGet finding after the second fix. The final scan came back with zero Critical or High findings, leaving only routine Info-level results and a couple of Medium-severity SSL certificate items. That progression — scan, find, fix, verify — is the complete vulnerability management lifecycle this lab was built to demonstrate, not just a one-time scan result.

## Related Labs

- **Lab 1** — Active Directory deployment (the source of the Windows Server VM used in this lab)
- **Lab 3** — Splunk SIEM & Log Analysis
