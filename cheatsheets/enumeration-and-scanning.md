# Enumeration & Scanning Cheat Sheet

Networking basics, host discovery, and per-service enumeration commands I
reach for constantly.

## Passive recon

```bash
whois <DOMAIN> -h <WHOIS_SERVER>
host <DOMAIN> ; host -t mx <DOMAIN> ; host -t txt <DOMAIN>

# Google dorking
site:<domain> filetype:txt intitle:"index of"

# GitHub recon — check for leaked .git dirs or secrets in public repos
# https://book.hacktricks.xyz/generic-methodologies-and-resources/external-recon-methodology/github-leaked-secrets

exiftool -a -u <FILE>          # metadata in documents/images
# shodan.io, searchdns.netcraft.com, securityheaders.com, ssllabs.com/ssltest — passive OSINT
```

## Networking basics

```bash
# Routing / IP / ARP / open ports — Linux | Windows | macOS
ip route            | route print        | netstat -r
ip a / ip -br -c a  | ipconfig /all      | ifconfig
ip neighbour         | arp -a             | arp
netstat -tulpn / ss -tnl | netstat -ano  | lsof -n -i4TCP -i4UDP

nc -v example.com 80
nc -zv <TARGET_IP> <PORT>
openssl s_client -connect <HOST>:<PORT>
```

## Host discovery & port scanning

```bash
sudo nmap -sn <TARGET_IP/NETWORK>              # ping sweep
netdiscover -i eth1 -r <TARGET_IP/NETWORK>     # ARP scan
sudo arp-scan -I eth1 <TARGET_IP/NETWORK>

nmap -Pn -p- <TARGET_IP>                       # all ports, skip ping
nmap -T4 -A -p- <TARGET_IP> -v                 # complete scan
nmap -sU <TARGET_IP>                           # UDP
sudo nmap -sV -p <PORT> --script "vuln" <TARGET_IP>   # vuln category scripts
nmap -Pn -sV -sC -O -oA outputfile <TARGET_IP> # save all formats

# NSE script lookup
locate .nse | grep <keyword>
nmap --script="<name>" <TARGET_IP>
```

## Service enumeration

### FTP (21)

```bash
ftp <TARGET_IP>           # try anonymous:anonymous
nmap -p21 --script ftp-anon,ftp-brute --script-args userdb=<USERS_LIST> <TARGET_IP>
hydra -L <USERS_LIST> -P <PASS_LIST> <TARGET_IP> ftp
```

### SSH (22)

```bash
ssh <USER>@<TARGET_IP>
# id_rsa/id_ecdsa protected by passphrase
ssh2john id_rsa > hash ; john --wordlist=rockyou.txt hash
hydra -l <USER> -P <PASS_LIST> <TARGET_IP> ssh
```

### SMB (139/445)

```bash
nmap -p445 --script smb-protocols,smb-security-mode,smb-os-discovery,smb-enum-shares,smb-enum-users <TARGET_IP>
sudo nbtscan -r <TARGET_IP/NETWORK>

smbclient -L //<TARGET_IP> -N              # anonymous
smbclient //<TARGET_IP>/<SHARE> -U <USER>
rpcclient -U "" -N <TARGET_IP>              # then: enumdomusers / enumdomgroups / getdompwinfo

smbmap -H <TARGET_IP> -u <USER> -p '<PW>' -r '<SHARE>'
enum4linux -a -u "<USER>" -p "<PW>" <TARGET_IP>

# crackmapexec / netexec
netexec smb <TARGET_IP> -u '' -p '' --shares --users --rid-brute
netexec smb <TARGET_IP> -u <USER> -p <PW> --shares --users --groups --sessions --pass-pol
```

### HTTP(S) (80/443)

```bash
whatweb <TARGET_IP>
gobuster dir -u http://<TARGET_IP> -w <wordlist> -x php,txt,html
gobuster vhost -u http://<TARGET_IP> -w <subdomains-wordlist>
nikto -h http://<TARGET_IP>
python3 dirsearch.py -u http://<TARGET_IP> -w <wordlist>
feroxbuster -u http://<TARGET_IP> -w <wordlist>
```

Checklist: view source/JS, check `/robots.txt` and `/sitemap.xml`, identify
CMS/version (Wappalyzer/whatweb) and check known exploits, inspect the TLS
cert for extra hostnames, try default credentials, check for a `cgi-bin/`.

```bash
# WordPress
wpscan --url <TARGET> --enumerate vp,u,vt,tt --follow-redirection --verbose

# Drupal / Joomla
droopescan scan drupal -u http://<TARGET>
droopescan scan joomla --url http://<TARGET>
```

### SQL (3306 MySQL / 1433 MSSQL)

```bash
mysql -h <TARGET_IP> -u root
nmap -p3306 --script=mysql-empty-password,mysql-info,mysql-databases <TARGET_IP>

# MSSQL — correct port is 1433, not MySQL's 3306
nmap -p1433 --script ms-sql-info,ms-sql-empty-password <TARGET_IP>
impacket-mssqlclient <USER>:<PW>@<TARGET_IP> -windows-auth
```

### SMTP (25)

```bash
nc <TARGET_IP> 25              # HELO/EHLO to check capabilities
smtp-user-enum -M VRFY -U <USERS_LIST> -t <TARGET_IP>
```

### LDAP (389/636)

```bash
ldapsearch -x -H ldap://<TARGET_IP>                                    # unauthenticated, always try first
ldapsearch -x -H ldap://<TARGET_IP> -D '' -w '' -b "DC=<domain>,DC=<tld>"
ldapsearch -x -H ldap://<TARGET_IP> -D '<DOMAIN>\<USER>' -w '<PW>' -b "CN=Users,DC=<domain>,DC=<tld>"

# password-policy / lockout check before spraying
ldapsearch -x -H ldap://<TARGET_IP> -b "dc=<domain>,dc=<tld>" -s sub "*" | grep -i lock

windapsearch.py --dc-ip <IP> -u <USER> -p <PW> --da         # domain admins
```

### NFS (2049)

```bash
showmount -e <TARGET_IP>
# on target: cat /etc/exports — check for "no_root_squash"
mount -o rw <TARGET_IP>:<share> <local_mount_dir>
```

### SNMP (161)

```bash
sudo nmap <TARGET_IP> -sU -p161 -A
snmpcheck -t <TARGET_IP> -c public
snmpwalk -c public -v1 <TARGET_IP>                                  # full MIB tree
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.4.1.77.1.2.25            # Windows users
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.6.3.1.2           # installed software
```

### RPC (135)

```bash
rpcclient -U "" -N <TARGET_IP>
# enumdomusers / enumpriv / queryuser <user> / netshareenum / lsaenumsid
```

## Known-CVE quick checks

```bash
nmap -sV --script ssl-heartbleed -p443 <TARGET_IP>          # Heartbleed
nmap --script smb-vuln-ms17-010 -p445 <TARGET_IP>            # EternalBlue
nmap --script log4shell.nse --script-args log4shell.callback-server=<CALLBACK_IP>:1389 -p8080 <TARGET_IP>
searchsploit <product> <version>
searchsploit -m <EXPLOIT_ID>     # copy exploit to CWD
```
