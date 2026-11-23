# Deviations from the planned protoype and Lessons Learned


## 1. Active Directory design

- The domain used was `LAB.LOCAL` (`lab.local`).
- The designed OUs were `Domain Controllers`, `Lab-Admins`, `Lab-ServiceAccounts`, `Lab-Users`, and `Lab-Workstations`.
- The Kerberoasting service account was `svc_sql` (display name `SQL Service`) in `Lab-ServiceAccounts`; the requesting user was `bsilverhand`.
- The SQL service SPN was registered for the lab Domain Controller host on port 1433.

## 2. Operating systems and virtualization

- The Domain Controller ran Windows Server 2025 Standard Evaluation with Desktop Experience.
- The client ran Windows 11 Enterprise LTSC 2024 Evaluation, rather than a generic Windows client edition.
- The lab ran under Fedora Linux with QEMU/KVM and virt-manager.
- A dedicated Ubuntu Server LTS VM (`adlab-ubuntu-wazuh`) was added for Wazuh.
- Guest hostnames were not identical to virt-manager VM names. The documentation now distinguishes VM names, guest hostnames, and Wazuh agent names.

## 3. Kerberoasting encryption and cracking mode

- The plan included RC4 as a common Kerberoasting signature.
- The actual request used Impacket `GetUserSPNs.py` with `-no-rc4` and the captured service ticket used AES256, etype `0x12` (decimal 18).
- Hashcat mode `19700` was used for the AES Kerberos TGS hash. The RC4 mode `13100` was not appropriate for the observed hash.
- The lab wordlist recovered the test service-account password. The recovered plaintext and the raw ticket hash are intentionally omitted from the public repository.
- The result demonstrates that stronger ticket encryption does not compensate for a weak service-account password; offline password guessing remains relevant to the captured ticket material.

## 4. Pass-the-Hash scope

- The successful hash-based remote execution used Impacket `wmiexec.py` against the lab client's local Administrator account.
- The demonstrated path was Kali-to-client. The lab had one Windows client, so it was not a Windows-to-Windows, multi-host pivot.
- Several Event 4624 records appeared around the operation. One representative event was selected, while the other related logon sessions were retained in the evidence set.

## 5. LLMNR/NBT-NS detection scope

- Responder captured an NTLMv2 response and tcpdump showed LLMNR traffic on UDP/5355.
- The tested Hashcat mode `5600` wordlist attempt did not recover the plaintext password.
- The reviewed evidence confirms LLMNR traffic; it does not establish that NBT-NS UDP/137 traffic was captured.
- No Zeek or Suricata sensor was installed, and no network sensor or Responder-log collection was configured in Wazuh.
- LLMNR/NBT-NS detection through Wazuh is therefore explicitly **not tested**, rather than reported as partially successful.

## 6. Wazuh deployment and validation

- Wazuh 4.14.7 was installed on a dedicated Ubuntu Server LTS VM at `10.10.10.20`.
- The Windows agent MSI was downloaded on Fedora and transferred through the Fedora shared folder to both Windows VMs. The same MSI package was used on both endpoints.
- The Windows agents were enrolled with distinct names: `adlab-dc01` and `adlab-client01`.
- Enrollment required troubleshooting of the shared password configuration. Both agents eventually appeared as Active in the Wazuh manager and dashboard.
- JSON event archiving and archive indexing were enabled temporarily for testing and disabled after validation to reduce disk usage.
- The matching Kerberoasting Event 4769 for `bsilverhand` requesting `svc_sql` was found in Wazuh archives but not in Wazuh alerts.
- A separate Event 4624 for NTLM authentication by `bsilverhand` at approximately the same time generated Wazuh rule `92652`, level 6. That rule is a generic remote-NTLM authentication alert with a “possible Pass-the-Hash” description; it is not a Kerberoasting detector and does not prove hash reuse.
- The client-side Pass-the-Hash Event 4624 was documented from Windows. The exact corresponding Wazuh archive/alert result was not established in the evidence reviewed for this report.
- The built-in Wazuh ruleset was inspected. The packaged rule chain was not modified.
- The repository's Sigma YAML files were not automatically loaded into Wazuh. The Wazuh alert described above came from the packaged Wazuh ruleset, not proof that a custom Sigma rule executed.

## 7. Resource constraints

Running all four VMs at once caused severe memory pressure on the Fedora host and terminated active VMs. Tests were therefore performed in smaller combinations: Wazuh plus the relevant Windows target plus Kali. Documentation and evidence preparation were completed with the VMs shut down.

## 8. General lessons

- Event generation, event collection, alert generation, and attack-specific classification are separate claims.
- A generic NTLM alert is not conclusive evidence of Pass-the-Hash.
- A Kerberos service-ticket event is not automatically malicious; account, service, source, encryption type, request frequency, and baseline context matter.
- Network-layer attacks require network-layer telemetry. Endpoint Windows event collection alone is not a packet sensor.
- Keep passwords, hashes, Responder captures, PCAPs, and VM images out of public Git history.
