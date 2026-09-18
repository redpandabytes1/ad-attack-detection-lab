# Active Directory Attack & Detection Lab

## Overview

This project builds an isolated Active Directory lab, executes three real
attack techniques against it (Kerberoasting, Pass-the-Hash, LLMNR/NBT-NS
poisoning), and pairs each attack with a corresponding detection rule —
covering both the offensive and defensive side of the same techniques.

> [!NOTE]
> Some of the credentials are confidential for me and are therefore needed to be sorted out before publishing publicly.

## Planned Features

- Isolated lab network with a Domain Controller, client, and attacker VM
- Domain populated with realistic, exploitable complexity
- Kerberoasting attack executed and documented
- Pass-the-Hash attack executed and documented
- LLMNR/NBT-NS poisoning attack executed and documented
- Sigma detection rule written for each attack
- Each attack mapped to its MITRE ATT&CK technique ID
- Detections validated against real log data (or explicitly scoped
      down, with that decision documented rather than hidden)

## Lab Topology

| Role | Hostname | OS | IP (isolated network) |
|---|---|---|---|
| Domain Controller | WIN-4LJEJ0TK4QI | Windows Server 2025 Standard Evaluation (Desktop Experience) | |
| Windows Client | DESKTOP-9J6SCIV | Windows 11 Enterprise LTSC 2024 Evaluation | |
| Attacker | speedster | Kali Linux 2026.3  | |

## Setup

hypervisor used, VM specs, how the isolated
     network was configured.

## Attacks & Detections

| Attack | MITRE ATT&CK ID | Writeup | Detection Rule |
|---|---|---|---|
| Kerberoasting | TODO | [attacks/kerberoasting.md](attacks/kerberoasting.md) | [detections/kerberoasting.yml](detections/kerberoasting.yml) |
| Pass-the-Hash | TODO | [attacks/pass-the-hash.md](attacks/pass-the-hash.md) | [detections/pass_the_hash.yml](detections/pass_the_hash.yml) |
| LLMNR/NBT-NS Poisoning | TODO | [attacks/llmnr-poisoning.md](attacks/llmnr-poisoning.md) | [detections/llmnr_poisoning.yml](detections/llmnr_poisoning.yml) |

## Legal & Ethical Use

Every attack in this repository was executed against an isolated lab
environment I built and control entirely — no production systems, shared
infrastructure, or third-party targets were involved at any point. This
lab has no network path to anything beyond the virtual machines described
above.

## License

MIT — see [LICENSE](LICENSE)
