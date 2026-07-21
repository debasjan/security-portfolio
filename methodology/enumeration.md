# Initial Enumeration & Foothold — Methodology

> How I go from *nothing* to a shell. Built from notes across dozens of HTB / THM /
> Proving Grounds machines.

**The one rule that matters:** enumerate everything before exploiting anything.
Most missed footholds are missed *enumeration*, not missed exploits — the way in
was usually visible, I just hadn't looked hard enough.

Related: [Linux PrivEsc](./linux-privesc.md) · [Windows PrivEsc](./windows-privesc.md) · [Active Directory](./active-directory.md)

---

## Phase 0 — Port scan

```bash
nmap -Pn -p- --min-rate 5000 -oN nmap-all.txt <TARGET_IP>     # fast, all TCP ports
nmap -Pn -sCV -p<open_ports> -oN nmap-svc.txt <TARGET_IP>     # versions + default scripts
nmap -Pn -sU --top-ports 50 -oN nmap-udp.txt <TARGET_IP>      # UDP: 161/69/53 hide here
```

Every version number goes straight into `searchsploit <service> <version>`, and I
note **every** open port before touching any of them — the interesting service is
often not the obvious one.

---

## Phase 1 — Per-service first moves

I keep a mental (and written) map of what to try first on each port. This is the
lookup table I work from:

| Port | Service | First moves |
|---|---|---|
| 21 | FTP | anonymous login, version exploit, can I upload to a webroot? |
| 22 | SSH | version (CVE?), credential reuse, key reuse |
| 25 | SMTP | `VRFY` / `RCPT` user enumeration |
| 53 | DNS | zone transfer `dig axfr @<TARGET_IP> <domain>` |
| 80/443 | HTTP | full web enumeration (below) |
| 88 | Kerberos | it's an AD box → [Active Directory](./active-directory.md) |
| 111 | RPC/NFS | `showmount -e <TARGET_IP>` |
| 135/139/445 | SMB | `nxc smb`, null session, `smbclient -L`, EternalBlue? |
| 161 | SNMP | `snmpwalk -v2c -c public <TARGET_IP>` → users/processes/ports |
| 389/636 | LDAP | `ldapsearch -x`, anonymous bind |
| 1433 | MSSQL | `sa` / default creds, `impacket-mssqlclient`, `xp_cmdshell` |
| 2049 | NFS | mount shares, hunt for creds/keys |
| 3306 | MySQL | default creds, version |
| 3389 | RDP | credential reuse, BlueKeep? |
| 5432 | Postgres | default creds → RCE |
| 5985/5986 | WinRM | `evil-winrm` with any creds I've found |
| 6379 | Redis | unauth → write SSH key / webshell |

---

## Phase 2 — Web enumeration (where most footholds live)

```bash
whatweb http://<TARGET_IP> ; curl -s http://<TARGET_IP> -I
feroxbuster -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html
gobuster vhost -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
nikto -h http://<TARGET_IP>
```

For each web app I work through:
- **View source** + HTML comments + JS files — endpoints and credentials leak here.
- **Default credentials** on every login form.
- **Known CMS?** → `wpscan --url <> --enumerate u,vp` / Joomla / Drupal → searchsploit.
- **Redirected to a hostname?** → add the vhost to `/etc/hosts`. A hidden vhost is a hidden box.
- **Injection:** SQLi (by hand — I don't reach for sqlmap first), LFI/RFI, command injection, SSTI, file upload → webshell.

---

## Phase 3 — Turn a finding into a shell

- Upload / webshell → reverse shell (`msfvenom`, php-reverse-shell, `nc`).
- Known-CVE RCE — I read the PoC before I run it, every time.
- Credentials found → spray them across SSH / SMB / WinRM / RDP / web logins.
- LFI → log poisoning or a PHP wrapper → RCE.
- MSSQL / Redis / Postgres → built-in code execution.

```bash
rlwrap nc -lvnp 443
# stabilise (Linux): python3 -c 'import pty; pty.spawn("/bin/bash")' ; Ctrl-Z ; stty raw -echo; fg
```

---

## My six golden rules

1. The full port scan finishes before I commit to any rabbit hole.
2. Version number → searchsploit, **every** time.
3. Default creds and anonymous access before any exploit development.
4. Web: enumerate directories **and** vhosts.
5. Every credential gets tried on every service and every host.
6. Read a public exploit before running it.

---

## References

- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [HackTricks — Pentesting Methodology](https://book.hacktricks.xyz/generic-methodologies-and-resources/pentesting-methodology)
- [SecLists](https://github.com/danielmiessler/SecLists)
