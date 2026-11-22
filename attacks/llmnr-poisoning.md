# LLMNR/NBT-NS Poisoning — Lab Writeup

## Date & scope

- Lab activity: Milestone 6, September 2026 (the captured troubleshooting session was recorded on 19 September 2026).
- Target: the author's Windows client, `adlab-client01` (`DESKTOP-9J6SCIV`, `10.10.10.220`).
- Attacking host: Kali Linux on the isolated lab network.
- Authorization: personally owned and controlled isolated KVM/QEMU lab.
- MITRE ATT&CK: [T1557.001 — Name Resolution Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001/).

## Objective

Demonstrate the risk of fallback name resolution by observing LLMNR traffic, responding with Responder in the isolated lab, and determining whether an authentication response could be captured. Document the limitations of endpoint-only detection for this network-layer behavior.

## Tools used

- Responder on Kali Linux.
- `tcpdump` for packet observation/capture.
- Hashcat mode `5600` for an offline NTLMv2 password-audit attempt.
- Windows client name-resolution and SMB access behavior.

## Methodology

1. Started Responder on the Kali network interface attached to the isolated lab network.
2. Triggered a name-resolution failure from the Windows client by requesting a non-existent network share/name.
3. Observed LLMNR traffic with tcpdump. The recorded output included UDP/5355 traffic involving the LLMNR multicast address `224.0.0.252` and IPv6 LLMNR multicast traffic.
4. Responder successfully captured an NTLMv2 authentication response from the client. The raw response was retained only as private lab evidence and is not published.
5. Attempted offline recovery using Hashcat mode `5600` and the tested wordlist. The tested wordlist did not recover the plaintext password.

## Findings and evidence

- Responder reported a successful NTLMv2 capture during the lab exercise.
- tcpdump output showed LLMNR traffic on UDP/5355. An example observed IPv4 flow was from the client (`10.10.10.220`) to `224.0.0.252:5355`.
- The tested Hashcat run did not recover the NTLMv2 response with the wordlist used.
- The supplied packet evidence confirms LLMNR activity. NBT-NS traffic on UDP/137 was not conclusively established by the screenshot/output reviewed, so this writeup does not claim that NBT-NS packets were captured.

## Detection assessment

The poisoning behavior is fundamentally network-layer activity. The lab did not deploy Zeek or Suricata, and Wazuh was configured to collect Windows endpoint event channels rather than Responder logs or network sensor telemetry. Therefore:

- No direct Windows Security event identifying the LLMNR poisoning was established.
- Wazuh detection of this poisoning behavior was **not tested**.
- The evidence for this experiment comes from Responder and tcpdump, not Wazuh.

## Impact

An attacker able to answer local name-resolution broadcasts/multicasts may cause a client to connect to an attacker-controlled host. If the client attempts to authenticate to that host, the attacker may capture an NTLM challenge-response that can be subjected to offline password guessing or, in some circumstances, relay attempts. The actual impact depends on protocol protections, credential policies, and whether the captured response can be cracked or relayed.

## Remediation

- Disable LLMNR through Group Policy where it is not required.
- Disable NBT-NS on network adapters where it is not required and validate compatibility before rollout.
- Prefer properly managed DNS and reduce fallback name-resolution exposure.
- Enforce SMB signing and other appropriate protections against relay paths.
- Use unique, strong passwords and reduce NTLM exposure.
- Monitor UDP/5355 (LLMNR), UDP/137 (NBT-NS), and suspicious SMB authentication flows with network telemetry. A network sensor should distinguish authorized name-resolution infrastructure from unexpected responders.

## Lessons learned

- tcpdump and Responder supplied direct evidence of LLMNR traffic and a captured NTLMv2 response.
- The tested wordlist did not crack the captured response; a capture is not equivalent to password recovery.
- LLMNR/NBT-NS poisoning should not be represented as a normal Windows Event Log detection when the relevant evidence is network traffic.
- A future extension could add Zeek or Suricata and forward selected network events to Wazuh. That extension was not part of this project validation.

## References

- [MITRE ATT&CK T1557.001 — Name Resolution Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001/)
- [Wazuh — Log data collection](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/index.html)
