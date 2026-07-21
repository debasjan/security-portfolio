# Web Application Attacks — Methodology

> My working playbook for attacking web applications specifically — the layer
> above generic service enumeration. Built from PortSwigger Web Security
> Academy material and from real footholds across this portfolio (Cap's IDOR,
> Nibbles/Academy/Bashed's upload-to-RCE pattern, Sau's SSRF pivot).

**The one rule that matters:** authentication, access control, and file
handling are the three places web apps most often get the *logic* wrong, as
opposed to a single missing patch. A missing patch is luck; a logic flaw is
almost always there if you look for it.

Related: [Initial Enumeration](./enumeration.md) · [Linux PrivEsc](./linux-privesc.md) · [Windows PrivEsc](./windows-privesc.md)

---

## Phase 0 — Recon & fingerprinting

```bash
whatweb http://<TARGET>
curl -s http://<TARGET> -I
feroxbuster -u http://<TARGET> -w <wordlist> -x php,txt,html,bak,old
gobuster vhost -u http://<TARGET> -w <subdomains-wordlist>
```

I always view page source and JS files by hand at this stage — comments,
hidden endpoints, and CMS fingerprints routinely name the exact software and
version in use (Nibbles' entire foothold started with a single HTML comment).
A `robots.txt` disallow list is also worth reading manually: it's effectively
a map of paths someone considered sensitive enough to hide from search
engines, which often makes it worth checking directly (Editor).

---

## Phase 1 — Authentication attacks

Authentication is where a website decides whether you are who you claim to
be — and it's a smaller, more auditable surface than most of an application,
which makes logic flaws in it disproportionately valuable.

**Username enumeration** — before brute-forcing anything, check whether the
app tells you a username is valid:
- Different **status codes** or **error messages** between "user doesn't
  exist" and "wrong password."
- Different **response times** — a valid username that triggers an extra
  password-hashing step will measurably respond slower than an invalid one.

**Brute-force protection flaws** — the two common defenses (IP block,
account lock) both fail predictably:
- An IP block that resets on a *successful* login from that IP means
  sprinkling your own valid login into the attempt list keeps the block from
  ever triggering.
- Account locking after N attempts can be sidestepped by spraying a *small*
  password list (≤ N guesses) across *many* usernames instead of many
  passwords against one account — no single account ever hits its limit.
- IP-based rate limiting is often bypassable by spoofing `X-Forwarded-For`
  (or similar headers) if the app trusts it for client identification.

**2FA implementation flaws:**
- After step one (password) succeeds, check whether "logged-in-only" pages
  are reachable *before* completing step two — some apps only gate the UI
  flow, not the actual session.
- If the second step identifies the account via a cookie/parameter set after
  step one, changing that value can let you submit a victim's verification
  code request without ever knowing their password.
- 2FA codes are usually short (4–6 digits) — if there's no rate limit on the
  *second* step specifically, it's a small, fast brute-force target on its
  own even with strong step-one credentials.

**Password reset flaws:**
- A reset link identified by a predictable parameter (`?user=victim`) instead
  of a high-entropy token can be walked directly.
- Even with a proper token, check whether the app *actually re-validates* it
  when the new password is submitted — some implementations only check the
  token to show the form, not to accept the submission.
- `X-Forwarded-Host` (or similar) trusted in building the reset link's domain
  can redirect a victim's reset token to an attacker-controlled server —
  "password reset poisoning."

**"Remember me" cookies** — if a persistent-login cookie is built from
predictable values (username + a weak hash of the password, especially with
no salt), it's brute-forceable offline once you understand its construction.
Building your own account and studying your own cookie is usually the
fastest way to reverse-engineer the format before targeting anyone else's.

---

## Phase 2 — Access control & IDOR

Access control bugs are where authentication succeeded correctly but the app
never checked whether *this* authenticated user should reach *that*
specific resource.

The clearest signal is a **numbered or predictable resource identifier** in a
URL or request body — `/data/20`, `/invoice?id=1042`, a UUID that's actually
sequential under the hood. On Cap, a "download your own network capture"
feature used exactly this shape, and simply requesting a different ID
returned another user's data with no ownership check at all.

```bash
# Walk the ID space directly, or diff behavior between your own session
# and an unauthenticated/different-user request to the same endpoint.
```

Also worth checking: whether a *lower-privileged* role can reach an
*admin-only* function just by knowing the URL — vertical access control gaps
are common when the UI hides a link but the server never re-checks the role.

---

## Phase 3 — File upload → RCE

A recurring pattern across this portfolio (Nibbles, Academy, Bashed, UpDown):
an upload feature that trusts the client, the filename, or a blacklist rather
than the actual file content.

**What to check, in order:**
1. **Is any upload accepted at all**, and where does it land relative to the
   web root? An uploaded file that isn't web-reachable isn't RCE yet.
2. **What does the filter actually check?** Extension blacklists routinely
   miss less-common executable formats — `.phar` behaves like `.php` through
   PHP's `phar://` wrapper but is rarely blacklisted alongside it (UpDown).
   A double extension (`shell.php.jpg`) sometimes survives naive filtering
   too.
3. **If dangerous functions are disabled** (`system`, `exec`, `shell_exec`),
   don't assume code execution is closed off — enumerate *which* functions
   are actually still enabled (tools like `dfunc-bypasser` automate this)
   rather than giving up after the obvious ones fail. `proc_open`/`popen`
   are commonly missed.
4. Once a shell lands, it's almost always as a low-privileged web service
   account (`www-data`, `apache`) — treat this as a foothold, not a finish
   line, and move straight into local enumeration.

---

## Phase 4 — Injection (brief — see also platform-specific notes)

- **SQLi**: test manually first (`'`, `"`, boolean/time-based payloads)
  before reaching for automation — understanding *why* a payload works
  matters more for OSCP-style exams than a tool's output.
- **SSTI**: template syntax reflected unescaped (`{{7*7}}` style probes)
  in a rendered response is worth testing on any app using a templating
  engine for user-influenced content.
- **Command injection**: any feature that visibly shells out to a system
  utility (ping tools, file converters, "check if a site is up" utilities)
  is worth probing with classic separators (`;`, `|`, `` ` ``,
  `$()`) — the "is this URL up" pattern reappeared across
  multiple boxes in this set and is a recurring real-world design mistake.

---

## Phase 5 — SSRF as a network pivot

Server-Side Request Forgery isn't just an information leak — it can be the
only path to a service that was never otherwise reachable. On Sau, a public
tool's "forward requests to this URL" feature, pointed at `127.0.0.1`,
revealed and ultimately let me exploit a second application that had no
external exposure at all.

- Any feature that fetches a URL on the server's behalf (webhooks, "import
  from URL," PDF/screenshot generators, forward/proxy tools) is worth
  pointing at `127.0.0.1` and common internal ports.
- If external URLs are blocked but internal ones aren't, that's the
  vulnerability — allow-list validation should apply to *resolved* addresses,
  not just the literal string submitted.

---

## My golden rules

1. Read page source, JS, and `robots.txt` by hand before running any
   automated tool — the fastest wins are often already visible.
2. A numbered ID in a request is a hypothesis to test, not a detail to
   ignore.
3. An extension blacklist is not an allow-list — check what the *runtime*
   can execute, not just what the filter author thought of.
4. A disabled dangerous function is not proof code execution is impossible —
   enumerate what's still enabled.
5. Any "fetch this URL for me" feature is a potential SSRF pivot into
   whatever's actually running on localhost.
6. Treat a web shell as `www-data` as a foothold, not a result — the local
   privilege escalation phase starts immediately.

---

## References

- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — the primary source for the authentication material above
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [HackTricks — Web Application Pentesting](https://book.hacktricks.xyz/pentesting-web)
- [dfunc-bypasser](https://github.com/teambi0s/dfunc-bypasser)
