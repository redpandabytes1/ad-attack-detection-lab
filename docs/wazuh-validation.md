# Wazuh deployment and validation

Wazuh 4.14.7 was installed as an all-in-one manager/indexer/dashboard VM at `10.10.10.20`. Windows agents were installed on both Windows endpoints and enrolled under the names `adlab-dc01` and `adlab-client01`. At the end of the deployment work, both agents appeared as **Active** in the manager and dashboard.

The validation distinguished three different outcomes:

1. **Event generated:** Windows recorded the event in its Security log.
2. **Event collected / archived:** the event reached Wazuh and appeared in `wazuh-archives-*`.
3. **Alert generated:** an event matched a Wazuh rule and appeared in `wazuh-alerts-*`.

For the Kerberoasting run, the specific Event 4769 was archived but did not produce a corresponding alert. A separate DC-side Event 4624 for `LAB\bsilverhand`, using NTLM from `10.10.10.248`, generated rule `92652` at level 6. That rule's description says “possible pass-the-hash,” but its condition checks remote IPv4 logon followed by an NTLM authentication package; it does not prove hash reuse or detect Kerberoasting specifically. Separately, for the client-side Pass-the-Hash test around 20:14, the user confirmed that Event 4624 activity appeared in both Wazuh archives and alerts for `adlab-client01`; the exact client-side rule ID and level were not recorded.

The Wazuh built-in ruleset was examined. The chain observed was `60103` (Windows audit success) → `60106` (Windows logon success, including event IDs 4624 and 4769) → `92651` (remote IPv4 address, excluding `127.0.0.1`) → `92652` (authentication package `NTLM`). The Kerberos Event 4769 used an IPv4-mapped IPv6 address (`::ffff:10.10.10.248`) and was Kerberos ticket telemetry, not NTLM. The address-family mismatch is a plausible reason it did not enter the remote-NTLM child-rule chain; it is documented as an analysis, not as a fully isolated root-cause test.

**Sigma vs. Wazuh:** The repository's detection YAML files are separate Sigma-format artifacts. They were not automatically imported or executed by Wazuh merely because they exist. The rule 92652 alert came from Wazuh's packaged ruleset. This project does not claim that its Sigma YAML rules were directly executed by Wazuh.

See [docs/wazuh-validation.md](docs/wazuh-validation.md) for event fields, rule behavior, and evidence status. Remaining unverified publication items are listed in [docs/remaining-confirmations.md](docs/remaining-confirmations.md).

## Deployment

- Manager VM: `adlab-ubuntu-wazuh` (`10.10.10.20`).
- Wazuh version: `4.14.7`.
- Windows agent `001`: `adlab-dc01`.
- Windows agent `002`: `adlab-client01`.
- Final manager CLI and dashboard checks showed both Windows agents as Active.
- The Windows MSI was transferred from Fedora through a shared folder and installed on both Windows guests. The same MSI package was used; each agent was enrolled with its own agent name.

## How results were evaluated

Three stages were kept separate:

1. **Windows event generation:** an event exists in the endpoint's Windows Event Viewer.
2. **Wazuh collection/archive:** the agent forwarded the event and the manager indexed or stored it in `wazuh-archives-*`.
3. **Wazuh alert:** the event matched a Wazuh rule and appeared in `wazuh-alerts-*`.

An archived event is not necessarily an alert. An alert is not necessarily a technique-specific detection.

## Kerberoasting test

### Matching Event 4769

| Field | Value |
|---|---|
| Event Record ID | `47783` |
| Requesting account | `bsilverhand@LAB.LOCAL` |
| Service | `svc_sql` |
| Source address | `::ffff:10.10.10.248` |
| Source port | `58988` |
| Ticket options | `0x40810010` |
| Ticket encryption type | `0x12` (AES256) |
| Session encryption type | `0x12` |
| Failure code | `0x0` |
| Event Viewer time | 22 October 2026, 19:24:20 (DC local display time) |
| XML UTC time | `2026-10-23T02:24:20.2257363Z` |

### Wazuh outcome

- The `svc_sql` Event 4769 was found in Wazuh archives.
- It was not found as a corresponding entry in Wazuh alerts.
- Therefore, collection/archive succeeded, but an alert for this exact Kerberos ticket request was not demonstrated.

## Associated DC-side NTLM event and alert

At approximately the same time, the DC recorded a separate Event 4624:

| Field | Value |
|---|---|
| Event Record ID | `47781` |
| Account | `LAB\\bsilverhand` |
| Logon type | `3` (Network) |
| Logon process | `NtLmSsp` |
| Authentication package | `NTLM` |
| NTLM package | `NTLM V2` |
| Key length | `128` |
| Source address | `10.10.10.248` |
| Source port | `37738` |
| Event Viewer time | 22 October 2026, 19:24:20 |

Wazuh generated alert rule `92652`, level `6`, on `adlab-dc01`, with a description classifying successful remote NTLM authentication as a possible Pass-the-Hash attack. This alert was for the Event 4624 authentication, not the Event 4769 Kerberoasting request. The matching user, source IP, and near-identical timestamps make the events contextually related to the same test activity, but the 4624 alert does not prove Pass-the-Hash.

## Built-in Wazuh rule-chain analysis

The packaged rules were inspected under `/var/ossec/ruleset/rules/`:

- `60103` (level 0): parent rule for Windows audit-success events (`win.system.severityValue` matching `AUDIT_SUCCESS` or `success`), with parent `60001`.
- `60106` (level 3): Windows logon-success classification; includes Event IDs `528`, `540`, `673`, `4624`, and `4769`; parent `60103`.
- `92651` (level 0): requires `60106`, an IPv4-formatted `win.eventdata.ipAddress`, and excludes `127.0.0.1`.
- `92652` (level 6): requires `92651` and `win.eventdata.authenticationPackageName` matching `NTLM`.

The 4624 event has an ordinary IPv4 source address (`10.10.10.248`) and NTLM authentication, which matches the remote-NTLM child-rule conditions. The 4769 event uses an IPv4-mapped IPv6 address (`::ffff:10.10.10.248`) and is Kerberos ticket telemetry, not NTLM. The IPv4-only condition in rule 92651 and the NTLM-only condition in 92652 are plausible reasons the 4769 event did not generate the same alert. This is a rule-path analysis; it was not separately proven by running the event through a Wazuh rule-testing harness.

## Pass-the-Hash test on the client

A representative client Event 4624 was recorded:

| Field | Value |
|---|---|
| Event Record ID | `10135` |
| Target account | `DESKTOP-9J6SCIV\\Administrator` |
| Target Logon ID | `0x201F56` |
| Logon type | `3` |
| Logon process | `NtLmSsp` |
| Authentication package | `NTLM` |
| NTLM package | `NTLM V2` |
| Key length | `128` |
| Source address | `10.10.10.248` |
| Source port | `38280` |
| Event Viewer time | 22 October 2026, approximately 20:14:18 |

Five successful events were observed around that displayed second. They represented distinct logon sessions, not five copies of one identical record. The representative EventRecordID `10135` was selected for detailed documentation.

The client-side Windows event supports the successful NTLM network-logon observation. The exact archive/alert outcome for this client-side Pass-the-Hash event was not conclusively documented in the evidence reviewed for this report. It is therefore left unclaimed here.

## LLMNR/NBT-NS

- Responder successfully captured an NTLMv2 response during the earlier Milestone 6 test.
- tcpdump showed LLMNR traffic on UDP/5355, including client traffic to `224.0.0.252`.
- The tested Hashcat mode `5600` wordlist attempt did not recover the password.
- NBT-NS UDP/137 traffic was not confirmed in the supplied evidence.
- No Zeek/Suricata sensor or Responder-log ingestion was deployed.
- Wazuh detection of LLMNR/NBT-NS poisoning was not tested.

## Conclusion

The lab validated Wazuh agent enrollment and event collection from both Windows endpoints. For Kerberoasting, the matching Event 4769 reached the Wazuh archive but did not generate a corresponding alert. A separate NTLM Event 4624 during that test generated the generic rule `92652` alert. For Pass-the-Hash, the client produced a successful NTLM network-logon Event 4624, and the user confirmed that the corresponding activity around 20:14 appeared in both Wazuh archives and alerts. The exact client-side alert rule ID, level, and description were not captured in the available evidence. LLMNR/NBT-NS was demonstrated with Responder and tcpdump, outside Wazuh's configured endpoint-only collection scope.

## Configuration cleanup

Full JSON event archiving was disabled after the validation period to limit storage growth. The ordinary Wazuh alerting configuration was retained. The final status check reported no failed systemd units, and the manager later showed both Windows agents as Active.
