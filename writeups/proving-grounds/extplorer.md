# Extplorer — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Default creds on a WordPress file-manager plugin, config-file credential leak, `disk` group abuse via `debugfs` |

---

## TL;DR

A WordPress install exposes the **eXtplorer** file manager plugin at a
guessable path, still on its default `admin:admin` credentials. Uploading a
PHP web shell through it into `wp-includes` gets code execution, and the
plugin's own config file leaks a password hash for a second local user.
Privilege escalation abuses that user's membership in the `disk` group —
raw access to the block device lets `debugfs` read `/etc/shadow` directly
off the filesystem image, bypassing normal file permissions entirely.

---

## Recon & Enumeration

```bash
sudo nmap -sCV -p- <TARGET_IP>
```

Port 80 served a WordPress site. Directory brute-forcing found a
`filemanager` path:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

That resolved to an **eXtplorer** login page.

---

## Foothold / Initial Access

`admin:admin` — the plugin's documented default credential pair — logged in
without issue. eXtplorer's file manager can create and edit files directly
on the server, so a new `shell.php` was created in the editor and filled
with a standard PHP reverse shell (pentestmonkey's), then placed inside
`/wp-includes` — a path unlikely to be blocked by any WordPress-specific
protection rules aimed at the uploads directory:

```bash
nc -lvnp 7777
```

Browsing to the planted shell caught a connection as `www-data`.

---

## Privilege Escalation

eXtplorer's own configuration file, `filemanager/config/.htusers.php`,
contained a password hash for a local user (`dora`). Cracking it with John
against `rockyou.txt` recovered the plaintext, and `su dora` with that
password succeeded — user flag retrieved.

Checking group membership (`id`) showed `dora` belongs to the **`disk`**
group — direct read access to raw block devices (`/dev/sd*`). That's
equivalent to bypassing the filesystem's own permission model entirely:
tools that read a device node directly don't go through normal file-level
access checks. Using `debugfs` (a low-level ext filesystem debugger) against
the root partition's device node made it possible to read `/etc/shadow`
straight off the raw filesystem image, hash and all — no elevated privilege
needed beyond `disk` group membership:

```bash
fdisk -l
debugfs /dev/sdXN
```

Cracking the extracted root hash offline recovered the root password
directly.

---

## Lessons Learned

- **Plugin/application defaults are still worth checking first**, especially
  on lesser-known file-manager style plugins — `admin:admin` is a
  disproportionately common finding.
- **A file manager's own configuration files are a credential source in
  their own right**, separate from the application's primary database.
- **Group membership in `disk` (or similarly privileged groups like
  `docker`, `lxd`, `adm`) is functionally a privilege escalation path** —
  raw device access bypasses normal file permission checks, letting tools
  like `debugfs` reach `/etc/shadow` regardless of its file-level
  permissions.

---

## Remediation

- Change all default application/plugin credentials immediately after
  installation, and enforce this at deployment time rather than relying on
  administrators to remember.
- Restrict `disk` (and other block-device-capable) group membership to
  accounts that genuinely require it — it is equivalent to root-level file
  access on that disk.
- Store plugin credentials hashed with a modern algorithm (bcrypt/Argon2)
  rather than a format crackable at speed with commodity wordlists.

---

**Machine:** [Proving Grounds — Extplorer](https://portal.offsec.com/labs/play)
