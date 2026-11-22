# Pass-the-Hash — Lab Writeup

## Date & scope

- Lab activity: October 2026; representative Windows event dated 22 October 2026.
- Target: the author's Windows 11 Enterprise LTSC 2024 client, `adlab-client01` (`DESKTOP-9J6SCIV`, `10.10.10.220`).
- Attacking host: Kali Linux, `10.10.10.248` during the captured run.
- Authorization: personally owned and controlled isolated KVM/QEMU lab.
- MITRE ATT&CK: [T1550.002 — Pass the Hash](https://attack.mitre.org/techniques/T1550/002/).

## Objective

Demonstrate that the local Administrator account on the lab client could be used for remote authentication with its NTLM hash rather than its plaintext password, and inspect the resulting Windows Security telemetry.

## Tools used

- Impacket `secretsdump.py` for the lab client's local account hash retrieval.
- Impacket `wmiexec.py` for remote WMI command execution using hash-based authentication.
- Windows 11 Security Event Log.
- Wazuh 4.14.7 endpoint agent and manager.

## Methodology

1. Used the lab's authorized local Administrator credentials to retrieve local SAM account hash material from the Windows client using Impacket.
2. Kept the extracted hash in private lab evidence. It is not included in this repository.
3. Used `wmiexec.py` with the `-hashes` option to authenticate to `DESKTOP-9J6SCIV` at `10.10.10.220` as the local `Administrator` account.
4. Confirmed that the remote shell opened and checked the remote host identity using `hostname` and `whoami` before exiting.
5. Inspected the client's Security log for successful logon Event 4624 records around the test time.

The hash value and full credential-bearing command are deliberately omitted. The demonstration was conducted only against the author's own lab endpoint.

## Findings and evidence

Five successful Event 4624 records were observed around 22 October 2026, 20:14:18 in the client's Security log. The events represented separate logon sessions; they should not be collapsed into a claim that Windows produced only one event.

The representative event selected for detailed documentation was:

| Field | Observed value |
|---|---|
| Event ID | `4624` |
| Event Record ID | `10135` |
| Event Viewer time | 22 October 2026, 20:14:18 (client local time) |
| Target account | `DESKTOP-9J6SCIV\Administrator` |
| Target Logon ID | `0x201F56` |
| Logon type | `3` (Network) |
| Logon process | `NtLmSsp` |
| Authentication package | `NTLM` |
| NTLM package | `NTLM V2` |
| Key length | `128` |
| Source address | `10.10.10.248` |
| Source port | `38280` |
| Elevated token | `Yes` |

The other four records occurred within the same displayed second and had matching account, logon type, NTLM V2, and source-address characteristics. They had different logon identifiers and/or source ports, consistent with distinct network logon sessions. Event 4624 records successful logon session creation; it does not contain a field that explicitly states “Pass-the-Hash.”

### Wazuh result

Both Windows agents were enrolled and shown as Active in Wazuh. The user confirmed that the client-side Event 4624 activity around 20:14 was present in both `wazuh-archives-*` and `wazuh-alerts-*` for agent `adlab-client01`. This establishes that Wazuh collected and archived the client-side event and generated an alert for activity in the Pass-the-Hash test window.

The exact client-side Wazuh alert rule ID, level, and description were not recorded in the evidence available for this writeup. They are left unspecified rather than guessed. Do not assume that the client-side alert used rule `92652`.

A separate Wazuh alert (rule `92652`, level 6) was observed on the Domain Controller during the Kerberoasting test for an NTLM Event 4624 involving `bsilverhand`. That is a distinct DC-side alert and must not be conflated with the client-side Pass-the-Hash alert.

## Impact

If an attacker obtains a usable NTLM password hash, systems that accept NTLM authentication may allow the attacker to authenticate without knowing the cleartext password. The potential impact depends on the account's privileges, remote-management exposure, local administrator reuse, and network segmentation.

## Remediation

- Reduce or disable NTLM where business compatibility allows, and monitor remaining NTLM usage.
- Use Windows LAPS or an equivalent solution to enforce unique, rotated local Administrator passwords across endpoints.
- Restrict local administrator privileges and remote WMI/SMB administration to approved management paths.
- Apply host firewalls and network segmentation to limit remote administration sources.
- Monitor successful network logons (Event 4624, Logon Type 3) with NTLM context, source-host baselines, account context, and supporting process/service telemetry.
- Treat a generic NTLM network logon as a signal for investigation, not conclusive proof of Pass-the-Hash.

## Lessons learned

- Event 4624 with Logon Type 3 and NTLM V2 is consistent with remote NTLM authentication, but it does not independently prove hash reuse.
- Multiple 4624 events can be generated during one remote operation. Comparing account, source address, timestamp, logon type, authentication package, and logon IDs is more reliable than selecting a record solely by timestamp.
- This was a Kali-to-client remote authentication demonstration against the client's own local Administrator account. It was not a multi-host Windows-to-Windows pivot because the lab had one Windows client.
- The client-side Event 4624 was confirmed in both Wazuh archives and alerts during the Pass-the-Hash test window. The exact client alert rule ID and level should be added if its alert details are exported.
- The separate DC-side rule 92652 alert says “possible pass-the-hash,” but its conditions are based on remote IPv4 logon and NTLM authentication. A hash-reuse claim needs corroborating attack execution evidence.

## References

- [MITRE ATT&CK T1550.002 — Pass the Hash](https://attack.mitre.org/techniques/T1550/002/)
- [MITRE ATT&CK Detection Strategy DET0409 — Pass the Hash (Windows)](https://attack.mitre.org/detectionstrategies/DET0409/)
- [Wazuh — Windows agent package](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
