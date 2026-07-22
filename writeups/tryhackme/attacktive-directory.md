# Attacktive Directory — TryHackMe

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **OS** | Windows (Active Directory) |
| **Status** | ✅ Room explicitly aimed at write-ups/beginners |
| **Key techniques** | Kerberos user enumeration (Kerbrute), AS-REP Roasting, SMB share credential leak, DCSync via a synced backup account |

---

## TL;DR

A guided introduction to core Active Directory attacks, chained end to end:
Kerbrute enumerates valid domain usernames without any credentials, one of
those accounts is AS-REP Roastable and cracks quickly, the cracked
account's accessible SMB share leaks a second, more privileged credential,
and that second account turns out to be synced with DCSync rights — letting
every domain hash (including Administrator's) be pulled directly and reused
via pass-the-hash.

---

## Recon & Enumeration

```bash
nmap -sC -sV <TARGET_IP>
```

Standard AD services (Kerberos, LDAP, SMB) confirmed a domain controller.
SMB/RPC enumeration (`enum4linux`) returned the NetBIOS domain name.

---

## Foothold / Initial Access

With no credentials yet, **Kerbrute** enumerated valid usernames by
abusing Kerberos pre-authentication responses — the KDC replies
differently for a valid vs. invalid username even before any password is
checked, so no domain account is needed to run this enumeration:

```bash
kerbrute userenum -d <domain> --dc <TARGET_IP> <userlist>
```

Two accounts stood out immediately by naming convention (service-style
accounts). One of them had **"Does not require Pre-Authentication"** set —
making it **AS-REP Roastable**: a Kerberos ticket for that account can be
requested with no password at all, and the returned ticket is encrypted
with a key derived from the account's actual password, crackable offline:

```bash
impacket-GetNPUsers <domain>/ -usersfile <userlist> -no-pass -dc-ip <TARGET_IP>
hashcat -m 18200 hash.txt <wordlist>
```

The hash cracked, yielding a working password for the service account.

---

## Privilege Escalation

With valid domain credentials, SMB share enumeration
(`smbclient -L //<TARGET_IP> -U <user>`) found a share reachable to that
account containing a small text file. Its contents were base64-encoded;
decoding revealed a **second account's credentials outright** — a backup
account.

The username itself (a "backup" account) was the cue for the next step:
domain backup accounts are frequently granted **DCSync** rights (the
ability to request replication data from the domain controller, as any
legitimate domain controller would) so that backup software can capture a
full copy of AD data, hashes included. Using `secretsdump.py` with the
backup account's credentials confirmed this — it pulled **every NTLM hash
in the domain**, including the Administrator account's, via the same
DRSUAPI/replication mechanism a real DC uses to sync with its peers:

```bash
secretsdump.py <domain>/<backup_user>:<password>@<TARGET_IP>
```

With the Administrator's NTLM hash in hand, no cracking was even needed —
NTLM hashes can authenticate directly via **pass-the-hash**:

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H <ntlm_hash>
```

Administrator-level access on the domain controller confirmed.

---

## Lessons Learned

- **Kerberos username enumeration needs zero credentials** — Kerbrute's
  pre-auth response timing/error difference is enough to build a valid
  user list before any password is ever tried.
- **AS-REP Roasting only requires one misconfigured account** ("Does not
  require Pre-Authentication") to bootstrap a full domain compromise chain
  from nothing.
- **An account named for its function ("backup") is a strong hint about
  its likely rights** — DCSync-capable accounts are a common, realistic
  requirement for backup software, and exactly the kind of over-privileged
  service account that shows up in real environments too.
- **DCSync doesn't require domain admin membership**, only the specific
  replication rights (`Replicating Directory Changes` / `...All`) — any
  account holding them can pull every hash in the domain.

---

## Remediation

- Disable Kerberos pre-authentication exceptions unless explicitly
  required, and monitor for AS-REP roasting attempts.
- Never store credentials in a plaintext or trivially-decodable (base64)
  file on any share, regardless of the share's access restrictions.
- Grant DCSync/replication rights only to genuine domain controller
  computer accounts; audit and remove them from any standard user or
  service account that doesn't need full replication capability.
- Rotate the `krbtgt` account (twice) and force a domain-wide credential
  reset after any suspected DCSync exposure.

---

**Room:** [TryHackMe — Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)
