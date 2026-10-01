---
title: "Enterprise Network & SIEM | TechPulse Analytics"
date: 2026-04-15 14:00:00 +0100
categories: [Projects, Blue Team]
tags: [siem, wazuh, cisco, packet-tracer, vlan, hardening, cis-benchmark, mitre-attack, sysmon, auditd]
image:
  path: https://github.com/user-attachments/assets/2985db5f-f810-49ad-8aff-d53e4cbd72f0


---

# Summary

This was my cybersecurity engineering project. I designed an enterprise network for a made-up data analytics company, TechPulse Analytics, and then built a working SIEM lab that monitors part of that network. The two pieces are meant to be one system: the SIEM lab is the SOC segment (VLAN 10) of the network I designed.

The network side is a Cisco Packet Tracer topology with nine VLANs, dual firewalls and full redundancy. The SIEM side is three VMs running [Wazuh](https://documentation.wazuh.com) 4.7.5, with a monitored Linux endpoint and a monitored Windows endpoint. I ran attack simulations against both endpoints to see what the SIEM caught, then hardened the Windows machine and measured the result against the CIS benchmark.

The headline number is the Windows CIS score going from 32% to 94% after hardening. The more interesting result, to me, was an attack that the SIEM did not catch, which I get to at the end.

Everything is on [GitHub](https://github.com/ampls-911/enterprise-network-security-siem):

- [Full technical report](https://github.com/ampls-911/enterprise-network-security-siem/blob/main/docs/TechPulse_Full_Technical_Report.md) — the complete device-by-device documentation
- [Compiled report (PDF)](https://github.com/ampls-911/enterprise-network-security-siem/blob/main/report/TechPulse_Report.pdf)
- [Packet Tracer topology](https://github.com/ampls-911/enterprise-network-security-siem/blob/main/packet-tracer/topology.pkt) · [Wazuh agent configs and auditd rules](https://github.com/ampls-911/enterprise-network-security-siem/tree/main/wazuh) · [attack and hardening notes](https://github.com/ampls-911/enterprise-network-security-siem/tree/main/scripts)

# Phase 1 — the network
![Packet Tracer simulation running the connectivity tests](https://github.com/user-attachments/assets/a516faff-cbd6-46df-8097-e45cef759e2a)
_Live traffic simulation across the TechPulse topology_ 

TechPulse is a 35-person company that handles client data, so the design is built around defense in depth: several independent layers, so that one failure does not open up the whole network.

The five layers are:

1. **Perimeter** — dual Cisco 2911 edge routers terminating two ISP links.
2. **Firewall** — dual Cisco ASA, with a DMZ interface between the internet and the inside.
3. **Segmentation** — nine VLANs, one per department or service.
4. **Micro-segmentation** — ACLs on the core switch so that even inside the network, servers are isolated from each other.
5. **Monitoring** — the Wazuh SIEM watching the SOC segment.

## VLANs

Each department and each server sits on its own /24:

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 10 | SOC-Management | 10.0.10.0/24 | Wazuh server and monitored endpoints |
| 20 | Staff-Wired | 10.0.20.0/24 | Wired workstations |
| 25 | Staff-Wireless | 10.0.25.0/24 | Staff WiFi (WPA2-PSK) |
| 30 | App-Server | 10.0.30.0/24 | Application server |
| 35 | DB-Server | 10.0.35.0/24 | Database server, strictly isolated |
| 36 | File-Server | 10.0.36.0/24 | File server, staff only |
| 40 | DMZ-Zone | 10.0.40.0/24 | Web server and email gateway |
| 50 | Guest-IoT | 10.0.50.0/24 | Guest devices, fully isolated |
| 99 | OOB-Management | 10.0.99.0/24 | Out-of-band management |

## The part I think matters most

The segmentation is only as good as the rules that enforce it. Three cases:

- **Database server (VLAN 35):** only the app server can reach it, and only on the MySQL port. Everything else is denied.
- **File server (VLAN 36):** only the staff VLANs, and only over SMB.
- **Guest network (VLAN 50):** no access to any internal resource at all, just the gateway out.

I enforced these twice, once on the ASA and once with ACLs on the core switch SVIs, so a misconfiguration in one place does not expose a server. I verified every rule in Packet Tracer's simulation mode. The one I care about: a staff PC can reach the app server but is blocked from the database, while the app server can reach the database on 3306 and nothing else. That is lateral movement containment, and it is the thing that pays off in Phase 2.

The design also has HSRP for gateway failover, a properly zoned DMZ that cannot initiate connections back inside, separate staff and guest wireless, and a dedicated out-of-band management VLAN.

A few things could not be done in Packet Tracer and would change in a real build: the 2911 image has no IOS IPS (a real deployment would use a Firepower appliance inline), and the ASA 5505 base license limits some DMZ behaviour. I documented these rather than pretending the simulation is production.

# Phase 2 — the SIEM lab

Three VMs on VMware, all on the same NAT network, representing VLAN 10:

| VM | OS | Role |
|---|---|---|
| wazuh-server | Ubuntu Server 22.04 | Wazuh manager, indexer and dashboard |
| wazuh-agent | Ubuntu Desktop 22.04 | Monitored Linux endpoint |
| Windows 10 Pro | Windows 10 | Monitored Windows endpoint |

One setup detail worth repeating because it cost me time: the manager and agents must be on the exact same Wazuh version. An agent installed from the default repo came in newer than the manager and would not report until I pinned it to `4.7.5` to match.

## Detection stack

On Windows I installed [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) with the [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config), so the agent forwards process creation, network connections, file drops, registry changes and WMI activity. I extended File Integrity Monitoring to the user folders and `%TEMP%`, and added the registry Run keys and other persistence locations. The [CIS Windows 10 benchmark](https://www.cisecurity.org/benchmark/microsoft_windows_desktop) loaded automatically through Wazuh's configuration assessment module, which is what gives the baseline score.

On Linux I extended FIM to `/home`, `/tmp`, `/root` and `/var/log`, and wrote 13 auditd rules covering privileged command execution, access to `/etc/shadow` and `/etc/sudoers`, SSH config changes, cron persistence and kernel module loading.

## Attacks and what the SIEM saw

The point of the lab was not to pull off the attacks, which are well known, but to see what reaches the dashboard.

**Windows ([APTSimulator](https://github.com/NextronSystems/APTSimulator)).** Running the full suite as administrator, the SIEM raised 13 alerts across five [MITRE ATT&CK](https://attack.mitre.org) tactics in one go. Three stood out:

| Rule | Level | What it caught | MITRE |
|---|---|---|---|
| 92213 | 15 (critical) | Executable dropped in `%TEMP%` | [T1105](https://attack.mitre.org/techniques/T1105/) Ingress Tool Transfer |
| 89501 | 12 (high) | WMI event consumer persistence | [T1546.003](https://attack.mitre.org/techniques/T1546/003/) |
| 92650 | 12 (high) | PsExec service installed | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) / [T1569.002](https://attack.mitre.org/techniques/T1569/002/) |

The takeaway is that Sysmon is doing the heavy lifting. Without it, most of these simply would not be visible to Wazuh.

**Linux.** I gave the agent account a weak password from the rockyou list and ran a [Hydra](https://github.com/vanhauser-thc/thc-hydra) SSH brute force against it from Kali. Wazuh logged the failed attempts, fired its brute-force rule once a threshold was crossed, and then logged the successful login after the failures, which is exactly the sequence you want to see. I then used a deliberately planted `sudo` misconfiguration to escalate to root via a [GTFOBins](https://gtfobins.github.io/) technique, and the auditd rules caught the privileged command and the read of `/etc/shadow`.

## The attack that got through

The result I keep coming back to is a kernel privilege escalation ([CVE-2026-31431, "CopyFail"](https://copy.fail/)) that corrupts a setuid binary in the page cache without ever changing the file on disk. It got a root shell, and the SIEM showed nothing for it.

That is not a Wazuh bug. FIM decides a file changed by hashing it on disk. If the on-disk bytes never change, there is nothing for a checksum to catch. It is a clean illustration of a blind spot in checksum-based file integrity monitoring: it sees changes to files, not changes to what a running program does in memory. Catching this class of attack needs a different signal, something watching kernel or process behaviour rather than file hashes.

The fix for the vulnerability itself is a kernel update; the short-term workaround is to disable the vulnerable crypto module. But the lesson for the monitoring side is the more useful one, and it is the reason I would not describe any SIEM as complete coverage.

# Hardening results

I hardened the Windows endpoint in two stages and re-scanned against CIS after each:

| Stage | CIS score |
|---|---|
| Default Windows 10 | 32% |
| After my PowerShell script | 41% |
| After HardeningKitty | 94% |

My own script targeted the specific failed checks from the baseline: password policy, account lockout, SMB signing, audit logging, UAC, PowerShell logging and so on. It moved the needle but not far, which surprised me until I looked at how many individual settings the benchmark checks. [HardeningKitty](https://github.com/scipag/HardeningKitty)'s HailMary mode applies the whole CIS finding list at once, and that is what took the score to 94%.

The honest footnote: a 94% on a standalone lab machine is not a 94% on a real managed endpoint. Most of the remaining failures are enterprise settings (domain join, specific update policies) that do not apply to a machine sitting by itself, and a default-deny benchmark score says nothing about whether the box is patched or what is running on it. It is a useful, measurable before-and-after, not a security guarantee.

I did not harden the Linux VM within the project, but the report includes the remediation I would apply: Fail2Ban against the brute force, a real password policy, removing the planted sudo rule, auditing SUID binaries and switching SSH to key-based auth.

# What I took from it

- Segmentation only works if the ACLs are right and tested. The design is the easy part.
- Endpoint telemetry decides what a SIEM can see. On Windows that was Sysmon; on Linux it was auditd. Default logging would have missed most of the attacks.
- A compliance score is a measurement, not a verdict. 32% to 94% is a real improvement, but it is one axis.
- Every detection tool has a blind spot, and the CopyFail result is mine. Knowing where a control is blind is more useful than the controls you can list.

# References

- [Wazuh documentation](https://documentation.wazuh.com)
- [MITRE ATT&CK](https://attack.mitre.org)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [HardeningKitty](https://github.com/scipag/HardeningKitty)
- [APTSimulator](https://github.com/NextronSystems/APTSimulator)
- [SwiftOnSecurity Sysmon config](https://github.com/SwiftOnSecurity/sysmon-config)
- [GTFOBins](https://gtfobins.github.io/)
- [CopyFail — CVE-2026-31431](https://copy.fail/)

*Cybersecurity Engineering project, IT Business School Nabeul, 2025-2026, supervised by Mr. Yassine Khlifi.*
