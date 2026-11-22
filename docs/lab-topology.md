# Lab Topology and Environment

## Overview

The Active Directory Attack & Detection Lab was hosted on Fedora Linux using QEMU/KVM and virt-manager. A dedicated isolated libvirt network, `adlab-isolated`, connected the attacker, Windows Domain Controller, Windows client, and Wazuh manager. The network used the `10.10.10.0/24` range and was intentionally separated from the host's normal LAN.

## VM inventory

| Role              | VM name              | Guest hostname / agent name                                    | Operating system                           | IP address                       |
| ----------------- | -------------------- | -------------------------------------------------------------- | ------------------------------------------ | -------------------------------- |
| Attacker          | Kali VM              | Kali Linux                                                     | Kali Linux 2026.3                          | `10.10.10.248` in captured tests |
| Domain Controller | `adlab-dc01`         | Guest hostname `WIN-4JLEJOTK4QI`; Wazuh agent `adlab-dc01`     | Windows Server 2025 Standard Evaluation    | `10.10.10.10`                    |
| Domain client     | `adlab-client01`     | Guest hostname `DESKTOP-9J6SCIV`; Wazuh agent `adlab-client01` | Windows 11 Enterprise LTSC 2024 Evaluation | `10.10.10.220`                   |
| SIEM              | `adlab-ubuntu-wazuh` | `adlab-ubuntu-wazuh`                                           | Ubuntu Server LTS 24.04                    | `10.10.10.20`                    |

**Important:** A virt-manager VM name is not necessarily the hostname inside the guest. Similarly, the Wazuh agent name is a separately configured identifier.

## Network plan

| Property | Value |
|---|---|
| libvirt network | `adlab-isolated` |
| Subnet | `10.10.10.0/24` |
| Domain Controller | `10.10.10.10` |
| Wazuh manager | `10.10.10.20` |
| Windows client | `10.10.10.220` |
| Kali attacker (captured run) | `10.10.10.248` |
| Default gateway | None configured for the isolated lab segment |

The DC provided Active Directory and DNS services for `LAB.LOCAL`. The client used the DC for domain DNS and was joined to the domain. Kali was attached to the isolated network for the controlled experiments. The Wazuh manager and both Windows agents shared the same lab segment for telemetry collection.

## Active Directory layout

Domain: `LAB.LOCAL` (`lab.local`).

Observed OUs:

```text
lab.local
├── Domain Controllers
├── Lab-Admins
├── Lab-ServiceAccounts
├── Lab-Users
└── Lab-Workstations
```

The service account used in the Kerberoasting test was `svc_sql` (display name `SQL Service`) in `Lab-ServiceAccounts`. The requesting domain user was `bsilverhand`. Other test users observed during directory inspection included Ben Silverhand, Bill NotGates, Johnny Diamondhead, Mark Altman, Sam Zuckerberg, and Thomson Vercetti.

## Telemetry architecture

- Windows Security, System, and Application event channels were collected by the Wazuh Windows agents.
- Wazuh manager received agent events on the isolated segment.
- JSON archive logging and archive indexing were temporarily enabled during validation.
- Event archives and alerts were inspected separately (`wazuh-archives-*` and `wazuh-alerts-*`).
- Full event archiving was disabled after validation to reduce storage growth; standard alerting was retained.

## Resource and operating constraints

The Fedora host had limited memory for four concurrently running VMs. Running Kali, the DC, the client, and the Wazuh VM simultaneously caused severe host resource pressure and terminated the active VMs. The tests were therefore conducted in smaller combinations, with the Wazuh manager and only the target Windows endpoint plus Kali powered on for each attack where possible.

## Security boundary

All systems described here were owned and controlled lab VMs. The isolated network was used to prevent test traffic from reaching unrelated networks. No third-party systems were authorized or targeted.
