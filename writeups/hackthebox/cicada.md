# Cicada — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Easy |
| **OS** | Windows (Active Directory) |
| **Status** | ✅ Retired |
| **Key techniques** | Guest SMB enumeration, password spraying, credential leakage via AD description/scripts, SeBackupPrivilege abuse |

---

## TL;DR

Cicada is a chain of small credential leaks rather than one dramatic
vulnerability. A guest SMB session exposes an HR onboarding document with a
default domain password, which spraying reveals is still active on one
account. That account's access reveals a second password sitting in plain
sight in another user's AD description field, and a third credential inside a
backup script on a share. The final account holds `SeBackupPrivilege`,
letting me dump the SAM/SYSTEM hives directly and pass the Administrator hash.

---

## Recon & Enumeration

```bash
nmap -sC -sV 10.129.231.149
```

Standard AD port set (Kerberos, RPC, LDAP, SMB, WinRM). A guest SMB session
was accepted:

```bash
crackmapexec smb 10.129.231.149 -u guest -p '' --shares
```

This exposed an `HR` share — worth checking first since guest-readable HR
content often contains onboarding material, and onboarding material often
contains default credentials.

---

## Foothold / Initial Access

`Notice from HR.txt` on the share contained a **default domain password**
issued to new accounts. Domain usernames were pulled via RID cycling:

```bash
netexec smb 10.129.231.149 -u guest -p '' --rid-brute
```

Spraying the default password across the resulting username list:

```bash
crackmapexec smb 10.129.231.149 -u users.txt -p '<DEFAULT_PASSWORD>'
```

One account, `michael.wrightson`, still had it set. That account didn't reach
any shares directly, but its authenticated view of the domain revealed more —
**another user's password sitting in their AD `description` field**, a
depressingly common place to stash a "temporary" credential permanently.

That second account reached a `DEV` share containing a PowerShell backup
script, `Backup_script.ps1`, with a **third credential** hardcoded inside it —
for `emily.oscars`, who has WinRM access:

```bash
evil-winrm -i 10.129.231.149 -u emily.oscars -p '<PASSWORD>'
```

User flag retrieved.

---

## Privilege Escalation

`whoami /priv` as Emily showed **`SeBackupPrivilege`** — a privilege meant to
let backup software read files regardless of normal ACLs. In practice, that
means it can read the `SAM` and `SYSTEM` registry hives directly, which is
functionally equivalent to handing out every local password hash on the box:

```powershell
# save SAM and SYSTEM hives using backup-privilege-aware tooling, then download them
```

With both hives on my machine:

```bash
impacket-secretsdump -sam sam -system system local
```

This produced the local Administrator's NTLM hash. Passing it directly:

```bash
evil-winrm -i 10.129.231.149 -u Administrator -H <NT_HASH>
```

Root flag retrieved.

---

## Lessons Learned

- **Guest/anonymous SMB access is worth checking for HR/onboarding
  content specifically** — it's a common place for default credentials to
  live long after they should have been rotated.
- **AD `description` fields are a surprisingly common credential dump.**
  Enumerating them (once authenticated) costs nothing and regularly pays off.
- **`SeBackupPrivilege` is effectively local-admin-equivalent** — any account
  holding it should be treated as high-value, since it bypasses file ACLs
  entirely for hive access.

---

## Remediation

- Never store credentials in `description` fields, onboarding documents, or
  scripts on shares — use a proper secrets manager.
- Rotate default/onboarding passwords immediately and enforce a password
  change on first login.
- Restrict `SeBackupPrivilege` (and `SeRestorePrivilege`) to a small,
  monitored set of accounts — it should never reach a general service or
  low-tier user account.

---

**Machine:** [Hack The Box — Cicada](https://www.hackthebox.com/machines/cicada)
