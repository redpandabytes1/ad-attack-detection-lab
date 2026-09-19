# MITRE ATT&CK Mapping

This table maps the three attack scenarios demonstrated in the lab to their corresponding MITRE ATT&CK techniques.

| Attack | Tactic | Technique ID | Technique Name |
|---|---|---|---|
| Kerberoasting | Credential Access | T1558.003 | Steal or Forge Kerberos Tickets: Kerberoasting |
| Pass-the-Hash | Defense Evasion / Lateral Movement | T1550.002 | Use Alternate Authentication Material: Pass the Hash |
| LLMNR/NBT-NS Poisoning | Credential Access / Collection  | T1557.001 | Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay |

## Lab Notes

### Kerberoasting

The lab demonstrated Kerberoasting against the intentionally SPN-backed
`svc_sql` service account. The original lab procedure expected an RC4
Kerberos service ticket, but the Windows Server 2025 domain controller
issued an AES256 service ticket after the RC4-dependent request was rejected.
The successful TGS was etype 18 and was cracked offline with Hashcat mode
19700.

### Pass-the-Hash

The lab demonstrated reuse of a local Administrator NTLM hash to authenticate
to the Windows client without supplying the plaintext password. The
demonstration used Impacket's `secretsdump.py` and `wmiexec.py` within the
isolated lab network.

### LLMNR/NBT-NS Poisoning

The lab demonstrated LLMNR/NBT-NS poisoning using Responder and captured an
NTLMv2 authentication response. Packet capture confirmed LLMNR traffic on
UDP/5355. Direct Windows Security Event Log detection of the poisoning stage
was not treated as reliable; network-layer monitoring was identified as the
appropriate detection layer.

## References

- MITRE ATT&CK — T1558.003: Kerberoasting
- MITRE ATT&CK — T1550.002: Pass the Hash
- MITRE ATT&CK — T1557.001: LLMNR/NBT-NS Poisoning and SMB Relay
