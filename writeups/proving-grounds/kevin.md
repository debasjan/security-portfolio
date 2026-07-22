# Kevin — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Default credentials, public Metasploit module (CVE-2009-3999) |

---

## TL;DR

Kevin is a single-vector box: **HP Power Manager** running on port 80
accepts default `admin:admin` credentials, and the disclosed version is
directly vulnerable to a known CVE with a ready Metasploit module — no
privilege escalation phase required at all.

---

## Recon & Enumeration

```bash
sudo nmap -p- -sCV <TARGET_IP>
```

Port 80 served an **HP Power Manager** web login. Logging in with
`admin:admin` succeeded immediately, and the post-login page disclosed the
exact version: 4.2.

---

## Foothold / Initial Access

HP Power Manager 4.2 is vulnerable to **CVE-2009-3999**, with a public
Metasploit module (`exploit/windows/http/hp_power_manager_filename`).
Setting `RHOSTS`/`LHOST` and running the module returned a session directly
as `NT AUTHORITY\SYSTEM`. Root flag retrieved on first exploitation.

---

## Lessons Learned

- **Default admin credentials on niche management/monitoring software
  (printer managers, UPS/power management consoles, etc.) are still common**
  in the wild, and worth trying before anything more involved.
- A disclosed version number post-login is enough to go straight to
  `searchsploit`/Metasploit before any manual vulnerability hunting.

---

## Remediation

- Change all default credentials immediately at deployment, and enforce
  this via configuration management rather than manual admin discipline.
- Patch HP Power Manager past the CVE-2009-3999 vulnerable range, or retire
  it if no longer supported.
- Keep infrastructure management consoles off networks reachable by
  untrusted hosts.

---

**Machine:** [Proving Grounds — Kevin](https://portal.offsec.com/labs/play)
