# Administrator — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Medium |
| **OS** | Windows (Active Directory) |
| **Status** | ✅ Retired |
| **Key techniques** | ACL chaining (GenericAll/ForceChangePassword), password-manager cracking, targeted Kerberoasting, DCSync |

---

## TL;DR

Administrator is a five-hop ACL chain, an assumed-breach box that starts with
one low-privileged account and ends in Domain Admin purely through
permission abuse — no exploit code anywhere in the path. Each account grants
just enough control over the next one to keep the chain moving: password
resets, a cracked password-manager database, a targeted Kerberoast, and
finally DCSync rights.

---

## Recon & Enumeration

```bash
nmap -sC -sV -p- 10.129.10.95
```

![nmap service scan](./assets/administrator/01-nmap.png)

FTP, DNS, Kerberos, RPC, LDAP — `administrator.htb`. With the provided starting
credentials (`olivia`), BloodHound collection was the obvious first move on an
ACL-heavy box like this:

```bash
bloodhound-python -d administrator.htb -c All -u olivia -p '<PASSWORD>' -ns 10.129.10.95
```

---

## Foothold / Initial Access

![BloodHound collection run](./assets/administrator/03-bloodhound-collection.png)

BloodHound showed `olivia` holds **`GenericAll`** over the user `michael` —
full control of the object, including the ability to reset his password
without knowing the old one:

```bash
net rpc password "michael" "<NEW_PASSWORD>" -U "administrator.htb"/"olivia"%"<PASSWORD>" -S 10.129.10.95
```

![BloodHound showing olivia's GenericAll over michael](./assets/administrator/02-bloodhound-genericall.png)

Logging in as Michael over WinRM didn't turn up anything directly useful, but
the same graph showed **Michael** can force a password change on `benjamin` —
the chain continues one hop at a time rather than granting everything at once:

```bash
bloodyAD -u michael -p '<PASSWORD>' -d administrator.htb --host 10.129.10.95 set password benjamin '<NEW_PASSWORD>'
```

Benjamin turned out to be a member of a **share moderators** group with FTP
access, where a `backup.psafe3` file was waiting — a Password Safe database.
Password Safe databases are crackable offline like any other KDF-protected
container:

```bash
hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt
```

Opening the cracked database in `pwsafe` revealed several more account
passwords. Spraying them against the domain found one valid hit — `emily` —
giving WinRM access and the user flag.

---

## Privilege Escalation

BloodHound showed **Emily** holds `GenericWrite` over the user `ethan`.
`GenericWrite` on an account without a registered SPN enables a **targeted
Kerberoast**: temporarily give the account an SPN, request a service ticket
for it, and crack the returned hash offline — turning a write permission into
a password-cracking opportunity that wouldn't otherwise exist:

```bash
python3 targetedKerberoast.py -d administrator.htb -u emily -p '<PASSWORD>'
hashcat ethan.txt /usr/share/wordlists/rockyou.txt
```

![BloodHound showing ethan's GetChangesAll (DCSync) rights](./assets/administrator/04-dcsync-rights.png)

Ethan's cracked password revealed the final link: BloodHound showed **Ethan**
holds **`GetChangesAll`** (the DCSync right) on the domain. A DCSync attack
pulls every account's password hash straight from Active Directory's
replication protocol:

```bash
impacket-secretsdump administrator.htb/ethan:'<PASSWORD>'@10.129.10.95
```

That returned the Administrator NTLM hash, used directly for a shell and the
root flag.

---

## Lessons Learned

- **ACL chains rarely hand over full control in one hop** — each account in
  this box only unlocked the *next* account, not Domain Admin directly. That
  pattern is realistic: real-world privilege escalation is almost always
  incremental.
- **Password-manager database files (`.psafe3`, `.kdbx`) found on a share are
  a jackpot** — they're crackable offline and typically contain several
  accounts' worth of credentials at once.
- **`GenericWrite` without an SPN is a targeted-Kerberoast setup**, not just
  a "write some attributes" permission — it's worth checking specifically for
  this on every account with `GenericWrite`.

---

## Remediation

- Treat `GenericAll`/`GenericWrite`/`ForceChangePassword` grants over any
  account as sensitive — map the full downstream chain (BloodHound) before
  assuming a permission is low-risk.
- Never store password-manager database exports on shared, network-reachable
  locations.
- Restrict `GetChangesAll`/`GetChanges` (DCSync rights) to Domain Controllers
  and a minimal admin tier — audit membership the same way you'd audit Domain
  Admins.

---

**Machine:** [Hack The Box — Administrator](https://www.hackthebox.com/machines/administrator)
