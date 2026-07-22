# Internal — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | SMB version fingerprinting, public SMB RCE (CVE-2009-3103) |

---

## TL;DR

Internal is a short, single-vulnerability box: an outdated Windows Server
2008 R2 SMB implementation is directly vulnerable to a known remote code
execution CVE, and a Metasploit module targeting it lands a
`NT AUTHORITY\SYSTEM` shell with no separate privilege escalation phase at
all.

---

## Recon & Enumeration

```bash
sudo nmap -p- -sCV <TARGET_IP>
```

SMB fingerprinted as a version of Windows Server 2008 R2. Running a
targeted vulnerability scan against the SMB port confirmed a match against
**CVE-2009-3103**, an older SMB2 remote code execution flaw:

```bash
sudo nmap -p 445 -sCV --script vuln <TARGET_IP>
```

---

## Foothold / Initial Access

Metasploit ships a module for this exact CVE
(`exploit/windows/smb/ms09_050_smb2_negotiate_func_index`). Setting `RHOSTS`
and `LHOST` and running the exploit landed a session directly as
`NT AUTHORITY\SYSTEM` — no intermediate low-privilege shell at all. Root
flag retrieved on first exploitation.

---

## Lessons Learned

- **Version fingerprinting from `nmap -sV` alone is often enough to
  identify a CVE** with a known, weaponized Metasploit module — always
  cross-reference detected versions against `searchsploit`/Metasploit
  before assuming deeper enumeration is needed.
- Not every machine has a distinct privilege-escalation phase: some
  vulnerabilities (particularly kernel/SMB-level ones on legacy Windows)
  land at SYSTEM directly.

---

## Remediation

- Patch or decommission legacy SMB implementations; Windows Server 2008 R2
  is long past end-of-life and should not be internet- or even
  internally-facing without compensating controls.
- Segment legacy systems that cannot be immediately patched, restricting
  SMB access to only the hosts that require it.

---

**Machine:** [Proving Grounds — Internal](https://portal.offsec.com/labs/play)
