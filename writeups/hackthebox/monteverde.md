# Monteverde — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Medium |
| **OS** | Windows (Active Directory) |
| **Status** | ✅ Retired |
| **Key techniques** | Anonymous LDAP enumeration, password spraying, Azure AD Connect credential extraction |

---

## TL;DR

Monteverde starts with the same low-effort wins as most AD boxes — anonymous
LDAP, a username-as-password spray — but the privilege escalation is what
makes it worth including: the box runs **Azure AD Connect**, the service that
synchronizes an on-prem domain with Azure AD, and its sync account credentials
can be decrypted directly from the local SQL database it depends on. That
account happens to be the domain administrator.

---

## Recon & Enumeration

```bash
nmap -sC -sV -p- -T4 10.129.228.111
```

![nmap service scan](./assets/monteverde/01-nmap.png)

DNS, Kerberos, RPC, LDAP (`MEGABANK.LOCAL`), SMB, WinRM. LDAP allowed an
anonymous bind:

```bash
ldapsearch -x -b "dc=megabank,dc=local" "*" -H ldap://10.129.228.111 | grep userPrincipalName
```

![anonymous LDAP bind leaking usernames](./assets/monteverde/02-ldap-usernames.png)

This is close to free reconnaissance — anonymous LDAP binds routinely leak a
full username list before a single credential is guessed.

---

## Foothold / Initial Access

With usernames in hand, the next check is always whether anyone reused their
username as their password:

```bash
crackmapexec smb 10.129.228.111 -u users.txt -p users.txt --continue-on-success
```

The service account `SABatchJobs` had exactly that. It had read access on
multiple shares, including `users$` — and inside a per-user folder was an
`azure.xml` file (an Azure AD Connect account export) containing a plaintext
password. Password reuse meant it also worked for the domain user `mhope`:

```bash
evil-winrm -i 10.129.228.111 -u mhope -p '<PASSWORD>'
```

User flag retrieved.

---

## Privilege Escalation

`mhope` belonged to an **Azure Admins** group, and `C:\Program Files`
confirmed both SQL Server and Azure AD Connect were installed. Azure AD
Connect needs a highly privileged domain account to perform its sync (often
literally the domain administrator, as a matter of historical default
configuration) — and it has to store that account's credentials *somewhere*
retrievable, since the sync process runs unattended. That "somewhere" is an
encrypted blob in the local `ADSync` SQL database, decryptable using a key
management API the service itself uses at runtime.

Running a known extraction script (querying `mms_server_configuration` and
`mms_management_agent`, then decrypting via the sync engine's own
`mcrypt.dll`) against the local database recovered the sync account's
plaintext credentials — which were, in fact, the domain Administrator's:

```powershell
# script queries ADSync DB config tables, then uses Azure AD Connect's own
# crypto library to decrypt the stored sync-account credential
```

![decrypted Azure AD Connect sync account credentials](./assets/monteverde/03-azuread-connect-creds.png)

Those credentials gave a WinRM session as Administrator and the root flag.

---

## Lessons Learned

- **Azure AD Connect is a single point of catastrophic failure if
  compromised** — the account it uses to sync is often over-privileged by
  default, and the service is *designed* to be able to decrypt its own stored
  credential, which means anyone with local access to the sync server can too.
- **A `.xml` config export found on a share is worth opening in full** — sync
  and integration tooling routinely embeds plaintext credentials in
  configuration exports meant only for internal use.
- **Password reuse between a service account and a real user account is
  still one of the fastest wins on any domain.**

---

## Remediation

- Run Azure AD Connect with a dedicated, minimally-privileged sync account —
  never the domain Administrator.
- Restrict local access to the AD Connect server itself; if an attacker can't
  reach the SQL database, the decryption path doesn't matter.
- Audit shares for configuration exports (`.xml`, `.config`, `.json`)
  containing credentials, and treat any found as an immediate incident.

---

**Machine:** [Hack The Box — Monteverde](https://www.hackthebox.com/machines/monteverde)
