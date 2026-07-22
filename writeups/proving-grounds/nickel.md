# Nickel — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Medium |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Process command-line info disclosure via an internal API, base64 credential leak, encrypted-PDF cracking, localhost-only command endpoint abuse |

---

## TL;DR

Nickel exposes an internal "DevOps" API whose method-guessing eventually
turns up an endpoint that dumps the full command line of every running
process — including one that leaks deployment credentials in its
arguments. Those credentials give SSH access, and from there a
password-protected PDF found on the box (cracked offline) reveals the
existence of a command-execution endpoint bound only to `127.0.0.1` and
running as SYSTEM — invisible to any external scan, but directly reachable
once inside.

---

## Recon & Enumeration

```bash
sudo nmap -sCV -p- <TARGET_IP>
```

Key ports: FTP (anonymous disabled), SSH, RDP, an HTTP "DevOps Dashboard"
on 8089, and a second HTTP service on 33333 acting as its backing API. The
dashboard's page source referenced the API by a link-local/APIPA address —
a hint that the real API address needed to be substituted with the actual
target IP.

---

## Foothold / Initial Access

Probing the API on 33333 with a plain `GET` returned Express.js's (Node.js)
signature 404 — meaning the route exists but isn't registered for that
method. Switching to `POST` (letting `curl` build a well-formed request
rather than hand-crafting one in Burp, which kept tripping malformed-header
errors) got further:

```bash
curl -i -X POST http://<TARGET_IP>:33333/list-running-procs -d 'test=1'
```

This endpoint returned full command lines for every running process — one
of which was a deployment tool invocation containing a username and a
base64-encoded password:

```
DevTasks.exe --deploy C:\work\dev.yaml --user ariah -p "<base64>" --server nickel-dev --protocol ssh
```

Decoding the password and connecting over SSH with the recovered
credentials succeeded. User flag retrieved.

---

## Privilege Escalation

`whoami /priv` showed nothing exploitable, and the account had no
`SeImpersonatePrivilege` — ruling out the Potato/PrintSpoofer family
entirely. With no obvious token-based path, enumeration shifted to
credential hunting on disk: a password-protected PDF (`Infrastructure.pdf`)
was found and pulled back to the attacking machine over `scp`.

Extracting the PDF's encryption hash and cracking it offline recovered the
password, and the decrypted contents referenced an internal
**"Temporary Command endpoint"** at a bare URL with a trailing `?` — a
strong hint that it takes a command via query string. That endpoint hadn't
appeared in the external nmap scan at all; checking active listeners
*from inside* the box explained why:

```
netstat -ano | findstr LISTENING
```

Port 80 was listening on `127.0.0.1` only — bound to loopback, invisible to
any external scan, and running as PID 4 (`System`). A test request
confirmed blind command execution as SYSTEM:

```
curl.exe "http://127.0.0.1/?whoami"
```

Uploading `nc.exe` and triggering it through the same endpoint (with every
space URL-encoded as `%20`) returned a reverse shell running as
`NT AUTHORITY\SYSTEM`. Root flag retrieved.

---

## Lessons Learned

- **An internal API's process-listing endpoint is a credential-leak vector
  in its own right** — command-line arguments routinely contain secrets that
  were never meant to be "logged" this way.
- **Always enumerate local listeners (`netstat`) after landing a shell** —
  external `nmap` cannot see loopback-bound services, and this box's real
  privilege escalation path was invisible from outside entirely.
- Don't assume the privesc vector matches the previous box's theme; ruling
  out token-based paths quickly (no `SeImpersonatePrivilege`) saved time
  before pivoting to credential/file hunting.

---

## Remediation

- Never expose process command-line details through an API, authenticated
  or not — secrets passed as CLI arguments should be treated as
  effectively public on any multi-user system.
- Encrypt-at-rest is not a substitute for access control: a
  password-protected PDF sitting in a reachable directory is still a
  crackable offline target.
- Bind sensitive command-execution endpoints to loopback *and* require
  authentication — loopback binding alone only stops external scanning, not
  a local low-privilege user.

---

**Machine:** [Proving Grounds — Nickel](https://portal.offsec.com/labs/play)
