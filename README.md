# Active Directory Attack & Detection Lab

A home lab simulating an attack against an Active Directory environment, built in VirtualBox with a domain controller, a domain-joined Windows 10 endpoint, a Kali Linux attacker machine, and a Splunk SIEM, all on one internal network.

The goal was to go through a full attack chain myself - compromise a domain account over RDP, then switch hats and hunt the resulting logs in Splunk, then run a couple of MITRE ATT&CK techniques through Atomic Red Team to see what a SIEM actually catches versus what gets missed.

Based on MyDFIR's "Active Directory Home Lab" series, but rebuilt with my own domain, users, and infrastructure, and I ran into a lot of my own problems along the way that aren't in the video (see the troubleshooting section).

---

## Table of Contents

- [Environment Overview](#environment-overview)
- [Network Topology](#network-topology)
- [Phase 1: Active Directory Setup](#phase-1-active-directory-setup)
- [Phase 2: Attack Simulation - RDP Brute-Force](#phase-2-attack-simulation---rdp-brute-force)
- [Phase 3: Detection in Splunk](#phase-3-detection-in-splunk)
- [Phase 4: Threat Emulation with Atomic Red Team](#phase-4-threat-emulation-with-atomic-red-team)
- [Challenges & Troubleshooting](#challenges--troubleshooting)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## Environment Overview

| Host | Role | OS | IP Address |
|---|---|---|---|
| ADDC01 | Domain Controller | Windows Server | 192.168.10.7 |
| Win Target | Domain-joined endpoint | Windows 10 | 192.168.10.100 |
| AD LAB KALI | Attacker machine | Kali Linux | 192.168.10.6 (DHCP) |
| SPLUNK | SIEM / log analysis | Splunk Enterprise (Ubuntu) | 192.168.10.10 |

Domain: `ricks.local`
Network: all four VMs isolated on a single VirtualBox NAT Network (`AD-Project`, 192.168.10.0/24)
Target user: `RICKS\jsanchez` (IT OU)

---

## Network Topology

![Network Topology Diagram](screenshots/01-setup/topology-diagram.png)

---

## Phase 1: Active Directory Setup

Built out a working AD domain before touching anything offensive:

- Promoted ADDC01 to a domain controller for `ricks.local`
- Created OUs (IT, HR, Users) and populated them with domain users
- Joined Win Target to the domain
- Configured the target account (`jsanchez`) with RDP access enabled, for the attack phase later

See `screenshots/01-setup/` for the AD Users and Computers config and domain join.

---

## Phase 2: Attack Simulation - RDP Brute-Force

The idea here was to simulate an outside attacker brute-forcing RDP credentials against a domain-joined endpoint from a separate machine on the same network.

### Setup

- Enabled RDP on Win Target for the `jsanchez` account
- Built a password wordlist on Kali from a trimmed sample of `rockyou.txt`, with the real target password added in so there'd be a guaranteed hit
- Fixed Kali's DNS and `/etc/krb5.conf` so it could actually resolve the `ricks.local` Kerberos realm against ADDC01

### Execution

I ended up using `kerbrute` for the actual brute-force, after `crowbar` and Hydra both gave me trouble (full writeup below in the troubleshooting section - neither one failed for simple reasons).

```bash
kerbrute bruteuser -d ricks.local --dc 192.168.10.7 password.txt jsanchez
```

Result: `[+] VALID LOGIN: jsanchez@ricks.local:********`, followed by a live RDP session into Win Target as jsanchez.

See `screenshots/02-attack/` for the brute-force output and the RDP session.

---

## Phase 3: Detection in Splunk

Once the attack was done, I switched over to Splunk to see what it actually caught.

### Setup note

Universal Forwarders on ADDC01 and Win Target were already sending Windows Security and Sysmon logs into Splunk's `endpoint` index before I ran the attack.

### Hunting the attack

```spl
index=endpoint jsanchez EventCode=4625
| table _time, host, Account_Name, Logon_Type, Failure_Reason
| sort -_time
```

This pulled up 161 failed logon events (Event ID 4625) on `target-pc` over about a 20-minute window - "Unknown user name or bad password," matching the wordlist run - followed by a successful logon (Event ID 4624) a few minutes after the failures stopped.

That's the pattern I was looking for: a burst of failed auth attempts against one account from one source, then a success right after.

See `screenshots/03-detection/` for the failed-logon burst and the matching successful logon.

---

## Phase 4: Threat Emulation with Atomic Red Team

I wanted to go past just the one attack path, so I used Atomic Red Team to run a couple of MITRE ATT&CK techniques directly on Win Target and checked whether Splunk picked each one up.

### T1136.001 - Create Account: Local Account

```powershell
Invoke-AtomicTest T1136.001
```

This creates a local account (`NewLocalUser`) and adds it to the local Administrators group - basically simulating an attacker setting up persistence after getting in.

Detection: showed up in Splunk through Sysmon process-creation logs, with the full `net.exe` -> `net1.exe user NewLocalUser` command chain, tied back to `RICKS\jsanchez`.

### T1059.001 - Command and Scripting Interpreter: PowerShell

```powershell
Invoke-AtomicTest T1059.001
```

A few of the sub-tests under this one (Mimikatz, BloodHound) failed because they needed external payloads/prerequisites I hadn't downloaded, and ran into local privilege restrictions - which actually makes sense on a domain-joined box. The core PowerShell execution sub-test ran fine though.

Detection: Splunk had this one tagged directly with `technique_id=T1059.001, technique_name=PowerShell` via Sysmon.

See `screenshots/04-atomic-redteam/` for the execution evidence and `screenshots/03-detection/` for the matching Splunk results.

---

## Challenges & Troubleshooting

This didn't go smoothly on the first try, and honestly I think that's worth documenting instead of hiding.

| Issue | Root Cause | Fix |
|---|---|---|
| crowbar returned "No results found" instantly | Outdated `/cert-ignore` flag vs. the `/cert:ignore` syntax current FreeRDP actually uses, plus Kali's DNS/Kerberos config pointing at the wrong place | Patched crowbar's source, fixed DNS to point at the DC, added `/etc/krb5.conf` |
| Big Kerberos clock skew between Kali and the DC | DC had no real NTP source | Synced time with `ntpdate` against the DC |
| Hydra's RDP module wasn't reporting the correct password even when it worked | Known bug in Hydra's (self-described) experimental RDP module | Confirmed via BadPwdCount math and verbose output, dropped Hydra for RDP |
| Hydra's SMB brute-force just hung | Windows Firewall silently dropping port 445 | Confirmed with nmap, ruled out SMB as a vector |
| Kept getting failed logins across multiple tools even with the password I thought was right | The account's actual password had drifted from what I assumed it was | Reset it with `Set-ADAccountPassword`, confirmed with a direct GUI login |
| Splunk license expired, blocking all searches | Enterprise Trial license had run out | Web UI license-group change didn't actually persist after restart - fixed it from the CLI instead: `splunk edit licenser-groups Free -is_active 1` |
| No Kerberos pre-auth failure events (4771) anywhere | DC's default audit policy doesn't enable Kerberos Authentication Service auditing | Switched detection strategy to the RDP-layer events (4624/4625) instead, which were on by default |

---

## MITRE ATT&CK Mapping

| Technique ID | Technique Name | Tactic | Status |
|---|---|---|---|
| T1110 | Brute Force | Credential Access | Executed & Detected |
| T1021.001 | Remote Desktop Protocol | Lateral Movement | Executed & Detected |
| T1136.001 | Create Account: Local Account | Persistence | Executed & Detected |
| T1059.001 | PowerShell | Execution | Executed & Detected |

---

## Key Takeaways

- A brute-force attack against a domain account is loud. Even with just default Windows auditing turned on, the 4625/4624 pattern alone is enough to catch it.
- Tooling isn't interchangeable - crowbar, Hydra, and kerbrute all behaved differently against the exact same target, and figuring out why each one failed mattered more than just getting one of them to work.
- Default audit policies leave real gaps. Kerberos-specific events weren't being logged at all in this environment, which by itself is worth flagging as a hardening issue.
- Atomic Red Team is a fast way to actually check whether a given technique generates visibility in your SIEM, instead of just assuming it does.

---

## Tools Used

- VirtualBox - virtualization
- Active Directory Domain Services - identity infrastructure
- Kali Linux - kerbrute, crowbar, Hydra, nmap
- Splunk Enterprise - SIEM / log analysis
- Atomic Red Team - MITRE ATT&CK technique emulation
- Sysmon - endpoint telemetry

---

## Repository Structure

```
screenshots/
├── 01-setup/           AD configuration, topology diagram
├── 02-attack/          Brute-force execution, RDP access, troubleshooting evidence
├── 03-detection/        Splunk queries and detection results
└── 04-atomic-redteam/   Atomic Red Team execution evidence
```
