# Twiggy — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Unauthenticated pre-auth RCE (SaltStack master) |

---

## TL;DR

Twiggy is a short, single-vulnerability box: an exposed **SaltStack master**
service is vulnerable to a well-known pre-authentication remote code
execution flaw, and a public exploit script gets a root shell in one step —
no privilege escalation phase needed.

---

## Recon & Enumeration

```bash
sudo nmap -p- -sCV <TARGET_IP>
```

SaltStack's master ports were identified as open and running a version
affected by a public pre-auth RCE.

---

## Foothold / Initial Access

The relevant exploit uses SaltStack's own Python client library
(`salt`) to reach the master's `cmd.exec_code` runner function directly —
an administrative function that was reachable without authentication in
the vulnerable version. Setting up a matching Python environment and running
the published exploit script against the target returned a shell as
**root** immediately:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install salt
python exploit.py --master <TARGET_IP>
```

Root shell obtained directly; no separate privilege escalation stage was
needed. Root flag retrieved.

---

## Lessons Learned

- **Some vulnerabilities land at the highest privilege directly** —
  SaltStack's master process runs privileged operations by design, so a
  pre-auth RCE against it skips the usual foothold-then-privesc structure
  entirely.
- Matching the exploit's expected library version (installing the same
  `salt` client package used by the exploit author) is often necessary for
  public PoC scripts that depend on a specific library's internal protocol
  handling.

---

## Remediation

- Patch SaltStack to a version past the disclosed pre-auth RCE
  (CVE-2020-11651/CVE-2020-11652 class of issues), and restrict master API
  access to trusted management networks only.
- Never expose configuration-management infrastructure (SaltStack, Ansible
  Tower, Puppet masters, etc.) to untrusted or internet-facing networks.

---

**Machine:** [Proving Grounds — Twiggy](https://portal.offsec.com/labs/play)
