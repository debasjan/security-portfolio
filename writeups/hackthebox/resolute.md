# Resolute — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Medium |
| **OS** | Windows (Active Directory) |
| **Status** | ✅ Retired |
| **Key techniques** | Anonymous RPC/LDAP enumeration, password spraying, PowerShell transcript credential leak, DnsAdmins abuse |

---

## TL;DR

Resolute chains a familiar early-AD pattern — anonymous enumeration, a
password left in an LDAP field, a lockout-safe spray — into an escalation path
I hadn't used elsewhere in this set: abusing the **DnsAdmins** group's ability
to load an arbitrary DLL into the DNS service, turning a restart of a routine
Windows service into `NT AUTHORITY\SYSTEM` on the domain controller.

---

## Recon & Enumeration

```bash
nmap -sC -sV 10.129.96.155
```

![nmap service scan](./assets/resolute/01-nmap.png)

LDAP (`megabank.local`) and RPC. Anonymous RPC bind pulled a username list:

```bash
rpcclient -U "" 10.129.96.155 -N
```

![anonymous RPC bind enumerating usernames](./assets/resolute/02-rpc-user-enum.png)

LDAP, still anonymous, was worth searching for anything stashed in an
attribute field — a habit that keeps paying off across this whole machine set:

```bash
ldapsearch -x -H ldap://10.129.96.155 -b "dc=megabank,dc=local" | grep password
```

That search turned up a password sitting in a user's `description` field.
Before spraying it, I checked the account lockout policy — spraying blind on
an unknown policy risks locking out real accounts:

```bash
ldapsearch -x -H ldap://10.129.96.155 -b "dc=megabank,dc=local" -s sub "*" | grep lock
```

No lockout threshold was configured, meaning a spray was safe to run broadly.

---

## Foothold / Initial Access

```bash
crackmapexec smb 10.129.96.155 -u users.txt -p '<PASSWORD_FROM_DESCRIPTION>'
```

The password worked for `melanie`, giving WinRM access and the user flag.

---

## Privilege Escalation

Windows logs more than people expect by default, including **PowerShell
transcripts** — and Resolute's `PSTranscripts` folder had one sitting in
plain reach, containing a command someone had typed with credentials embedded
directly on the command line. That's a recurring theme across this whole
portfolio: transcript/history files are one of the highest-value places to
look on any Windows foothold, because they capture exactly the kind of
mistake a rushed admin makes once and then forgets about.

![PowerShell transcript leaking a credential on the command line](./assets/resolute/03-ps-transcript-leak.png)

The leaked credentials belonged to `ryan`, a member of **DnsAdmins**.

![ryan's membership in the DnsAdmins group](./assets/resolute/04-dnsadmins-group.png) This
group can specify a plugin DLL for the DNS Server service to load — a
legitimate extensibility feature that, combined with write access to the
registry key controlling it, becomes a privileged code-execution primitive:
building a malicious DLL, pointing the DNS service at it, and restarting the
service loads that DLL as `NT_AUTHORITY\SYSTEM`.

```
msfvenom -p windows/x64/exec cmd='net user administrator <NEW_PASSWORD> /domain' -f dll > da.dll
```

To avoid tripping a security product on a normal file copy, the DLL was
served over an Impacket SMB server rather than transferred directly:

```bash
sudo impacket-smbserver share ./
```

`dnscmd` set the registry path to the hosted DLL, and restarting the DNS
service (a right DnsAdmins members are commonly, if unintentionally, granted
on this kind of box) loaded it — running the payload as SYSTEM and rewriting
the Administrator's password. From there:

```bash
impacket-psexec megabank.local/administrator@10.129.96.155
```

Root flag retrieved.

---

## Lessons Learned

- **PowerShell transcript logs are one of the highest-value artifacts on any
  Windows box** — they capture exactly the credentials-on-the-command-line
  mistake that's otherwise invisible.
- **DnsAdmins is a much more powerful group than its name suggests** — DLL
  loading via the DNS service is a well-known SYSTEM-level escalation, not a
  documentation footnote.
- **Checking the account lockout policy before spraying isn't optional** —
  it's the difference between a safe recon step and taking down real
  accounts.

---

## Remediation

- Disable or restrict PowerShell transcription in locations reachable by
  low-privileged users, and never type credentials directly on a command line.
- Treat DnsAdmins membership as Tier-0 privileged — it should be as tightly
  controlled and audited as Domain Admins.
- Enforce a sane account lockout policy; "no lockout" turns every future
  spraying attempt into a free, safe attack for anyone on the network.

---

**Machine:** [Hack The Box — Resolute](https://www.hackthebox.com/machines/resolute)
