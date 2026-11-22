# Active Directory Attack & Detection Lab

A self-contained Active Directory security lab built with QEMU/KVM to reproduce three credential-access and authentication techniques, examine the Windows telemetry they generate, and validate what a Wazuh SIEM collects and alerts on.

> **Scope and safety:** This project was conducted in a privately controlled, isolated virtual network for learning and detection engineering. The commands and observations are documented for that lab only. Do not use them against systems or accounts without explicit authorization.

## Project goals

- Build a small Active Directory domain with a Domain Controller, a domain-joined Windows client, and deliberately configured test accounts.
- Reproduce Kerberoasting, Pass-the-Hash, and LLMNR/NBT-NS poisoning in the isolated lab.
- Identify the Windows events and network telemetry produced by each experiment.
- Forward Windows endpoint events to Wazuh and distinguish event collection from rule-generated alerts.
- Record limitations and false-positive risks rather than claiming detections that were not demonstrated.

## Lab environment

The virtual machines were hosted on Fedora Linux using QEMU/KVM and virt-manager. They communicated over the isolated libvirt network `adlab-isolated` (`10.10.10.0/24`). The VMs were not intended to be reachable from the host's normal LAN.

| Role              | VM name              | Guest hostname / identity | OS                                         | Lab IP                                   |
| ----------------- | -------------------- | ------------------------- | ------------------------------------------ | ---------------------------------------- |
| Attacker          | `kali-learning-box`  | `speedster`               | Kali Linux 2026.3                          | `10.10.10.248` during the captured tests |
| Domain Controller | `adlab-dc01`         | `WIN-4JLEJOTK4QI`         | Windows Server 2025 Standard Evaluation    | `10.10.10.10`                            |
| Domain client     | `adlab-client01`     | `DESKTOP-9J6SCIV`         | Windows 11 Enterprise LTSC 2024 Evaluation | `10.10.10.220`                           |
| SIEM              | `adlab-ubuntu-wazuh` | `ubu_speedster`           | Ubuntu Server LTS 24.04                    | `10.10.10.20`                            |

**Naming note:** VM names, guest OS hostnames, and Wazuh agent names are separate identifiers. The Wazuh agent names used were `adlab-dc01` and `adlab-client01`.

## Lab Topology

![Active Directory Attack and Detection Lab Topology](docs/screenshots/lab-topology.png)

See [docs/lab-topology.md](docs/lab-topology.md) for the addressing, Active Directory layout, and telemetry flow.

## Active Directory design

- Domain: `LAB.LOCAL` (DNS name `lab.local`).
- Organizational units observed: `Domain Controllers`, `Lab-Admins`, `Lab-ServiceAccounts`, `Lab-Users`, and `Lab-Workstations`.
- Kerberoasting test account: `svc_sql` (display name `SQL Service`), placed in `Lab-ServiceAccounts` and configured with an MSSQL service principal name.
- Requesting domain user used in the Kerberoasting test: `bsilverhand`.
- Other lab user names observed during directory inspection included Ben Silverhand, Bill NotGates, Johnny Diamondhead, Mark Altman, Sam Zuckerberg, and Thomson Vercetti.

The lab's account names and structure were chosen for a small fictional organization. Passwords, password hashes, enrollment secrets, and other credential material are deliberately not published.

## Experiments and outcomes

| Experiment | MITRE ATT&CK | Demonstrated outcome | Detection / telemetry result |
|---|---|---|---|
| Kerberoasting | [T1558.003](https://attack.mitre.org/techniques/T1558/003/) | Requested a TGS for `svc_sql`; the AES256 ticket hash was recovered offline in the lab. | DC Event 4769 for `bsilverhand` → `svc_sql` was present in Wazuh archives. No corresponding alert for that 4769 was observed. A separate Event 4624 at the same time triggered Wazuh rule 92652 for NTLM remote logon. |
| Pass-the-Hash | [T1550.002](https://attack.mitre.org/techniques/T1550/002/) | Used the client's local Administrator NTLM hash with Impacket `wmiexec.py`; a remote shell was obtained. | Client Event 4624 showed a successful NTLMv2 network logon from Kali. The user confirmed the event activity around 20:14 appeared in both Wazuh archives and alerts for `adlab-client01`; the exact client-side alert rule ID/level was not recorded. |
| LLMNR/NBT-NS poisoning | [T1557.001](https://attack.mitre.org/techniques/T1557/001/) | Responder captured an NTLMv2 response after a name-resolution failure; the tested wordlist did not recover the password. | tcpdump showed LLMNR traffic on UDP/5355. NBT-NS traffic was not confirmed in the supplied capture. Wazuh network-layer detection was not configured or tested. |

Detailed reports: [Kerberoasting](attacks/kerberoasting.md), [Pass-the-Hash](attacks/pass-the-hash.md), and [LLMNR/NBT-NS poisoning](attacks/llmnr-poisoning.md).


## MITRE ATT&CK mapping

| Experiment | Technique | Tactic |
|---|---|---|
| Kerberoasting | [T1558.003 — Kerberoasting](https://attack.mitre.org/techniques/T1558/003/) | Credential Access |
| Pass-the-Hash | [T1550.002 — Pass the Hash](https://attack.mitre.org/techniques/T1550/002/) | Lateral Movement |
| LLMNR/NBT-NS poisoning | [T1557.001 — Name Resolution Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001/) | Credential Access / Adversary-in-the-Middle context |

See [docs/mitre-attack-mapping.md](docs/mitre-attack-mapping.md).

## Repository layout

```text
.
├── README.md
├── attacks/
│   ├── kerberoasting.md
│   ├── pass-the-hash.md
│   └── llmnr-poisoning.md
├── detections/
│   ├── kerberoasting.yml
│   ├── pass-the-hash.yml
│   └── llmnr-poisoning.yml
└── docs/
    ├── lab-topology.md
    ├── mitre-attack-mapping.md
    ├── wazuh-validation.md
    ├── deviations-and-lessons.md
    ├── mitre-references.md
    └── screenshots/
```

## Authorization

All experiments were performed against my own isolated virtual machines and fictional Active Directory domain. No third-party systems were in scope.
