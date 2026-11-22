# MITRE ATT&CK Mapping

The technique IDs below were checked against the current MITRE ATT&CK Enterprise pages on 2 November 2026.

| Lab experiment | Technique ID | Official technique name | Tactic / context | What the lab demonstrated |
|---|---|---|---|---|
| Kerberoasting | [T1558.003](https://attack.mitre.org/techniques/T1558/003/) | Steal or Forge Kerberos Tickets: Kerberoasting | Credential Access | Requested a TGS for the SPN-backed `svc_sql` account and attempted offline password recovery. |
| Pass-the-Hash | [T1550.002](https://attack.mitre.org/techniques/T1550/002/) | Use Alternate Authentication Material: Pass the Hash | Lateral Movement | Used a local Administrator NTLM hash with Impacket `wmiexec.py` to authenticate remotely to the lab client. |
| LLMNR/NBT-NS poisoning | [T1557.001](https://attack.mitre.org/techniques/T1557/001/) | Adversary-in-the-Middle: Name Resolution Poisoning and SMB Relay | Adversary-in-the-Middle; credential collection/relay context | Responder captured an NTLMv2 response after LLMNR name-resolution traffic was observed. NBT-NS was not confirmed in the supplied packet evidence. |

## Detection context

- MITRE ATT&CK T1558.003 describes Kerberoasting as obtaining TGS tickets for SPNs and attempting to crack service-account ticket material offline. See [Detection Strategy DET0157](https://attack.mitre.org/detectionstrategies/DET0157/) for an example of combining Event 4769, encryption-type information, request volume, and service-account baselines.
- MITRE ATT&CK T1550.002 describes use of a stolen password hash to authenticate without the cleartext password. See [Detection Strategy DET0409](https://attack.mitre.org/detectionstrategies/DET0409/) for correlation-oriented detection context.
- MITRE ATT&CK T1557.001 describes name-resolution poisoning using LLMNR/NBT-NS/mDNS responses. This lab captured LLMNR evidence; it did not validate NBT-NS traffic or deploy a network IDS.

## Scope note

Technique mapping classifies the behavior demonstrated in the controlled lab. A technique ID in a YAML file or report does not itself mean that Wazuh detected the technique. Wazuh rule matches, Sigma rules, Windows event collection, and network-sensor detections are separate artifacts and must be reported separately.
