# ✍️ Writeups

Machine solutions focused on **the reasoning behind each step**, not just the
commands. **Retired / permitted machines only.**

## Index

### Active Directory

| Machine | Difficulty | Techniques |
|---|---|---|
| [Active](./hackthebox/active.md) | Easy | GPP `cpassword` decryption, Kerberoasting |
| [Cicada](./hackthebox/cicada.md) | Easy | Guest SMB enum, password spray, SeBackupPrivilege |
| [Fluffy](./hackthebox/fluffy.md) | Easy | CVE-2025-24071, ACL chaining, ADCS ESC16 |
| [Forest](./hackthebox/forest.md) | Easy | AS-REP Roasting, BloodHound, ACL abuse, DCSync |
| [Sauna](./hackthebox/sauna.md) | Easy | Username OSINT, AS-REP Roasting, AutoLogon creds, DCSync |
| [Support](./hackthebox/support.md) | Easy | Anonymous SMB, .NET decompilation, LDAP `info` leak, RBCD |
| [Timelapse](./hackthebox/timelapse.md) | Easy | Archive/PFX cracking, cert-based auth, LAPS abuse |
| [Administrator](./hackthebox/administrator.md) | Medium | ACL chaining, password-manager cracking, targeted Kerberoast, DCSync |
| [Escape](./hackthebox/escape.md) | Medium | Guest SMB PDF, MSSQL hash capture, AD CS ESC1 |
| [Monteverde](./hackthebox/monteverde.md) | Medium | Anonymous LDAP, password spray, Azure AD Connect extraction |
| [Puppy](./hackthebox/puppy.md) | Medium | `GenericWrite` group abuse, KeePass, account re-enable, DPAPI |
| [Resolute](./hackthebox/resolute.md) | Medium | Anonymous enum, PS transcript leak, DnsAdmins abuse |
| [Signed](./hackthebox/signed.md) | Medium | MSSQL-only enum, Responder crack, Kerberos Silver Ticket |
| [StreamIO](./hackthebox/streamio.md) | Medium | UNION SQLi, LFI→source, `firepwd` on `key4.db`, `WriteOwner`→LAPS |
| [TombWatcher](./hackthebox/tombwatcher.md) | Medium | WriteSPN → targeted Kerberoast, AddSelf, gMSA read, ForceChangePassword, WriteOwner, dacledit on OU, AD CS ESC3 |
| [Voleur](./hackthebox/voleur.md) | Medium | office2john, AD Recycle Bin restore, DPAPI, targeted Kerberoast (Kerberos-only) |
| [Blackfield](./hackthebox/blackfield.md) | Hard | Anonymous SMB user enum, AS-REP Roasting, `ForceChangePassword`, LSASS dump w/ pypykatz, NetExec `backup_operator` |

### Linux

| Machine | Difficulty | Techniques |
|---|---|---|
| [Cap](./hackthebox/cap.md) | Easy | IDOR, pcap credential extraction, Linux capabilities |
| [CozyHosting](./hackthebox/cozyhosting.md) | Easy | Spring Actuator session hijack, cmd injection (`${IFS}`), sudo `ssh` |
| [Nibbles](./hackthebox/nibbles.md) | Easy | Source recon, CMS RCE, sudo misconfiguration |
| [Academy](./hackthebox/academy.md) | Easy | Anonymous FTP leak, upload RCE, cron hijack |
| [Armageddon](./hackthebox/armageddon.md) | Easy | Drupalgeddon2, config leak, GTFOBins `snap` |
| [Bashed](./hackthebox/bashed.md) | Easy | Exposed webshell, sudo to secondary user |
| [Beep](./hackthebox/beep.md) | Easy | TLS downgrade, Elastix LFI, credential reuse |
| [Broker](./hackthebox/broker.md) | Easy | ActiveMQ RCE (CVE-2023-46604), sudo nginx abuse |
| [Editor](./hackthebox/editor.md) | Easy | XWiki RCE, config leak, Netdata SUID abuse |
| [Expressway](./hackthebox/expressway.md) | Easy | UDP/IKE discovery, PSK cracking, sudo chroot CVE |
| [Keeper](./hackthebox/keeper.md) | Easy | Default creds, KeePass memory-dump CVE, key conversion |
| [Lame](./hackthebox/lame.md) | Easy | Samba `usermap_script` RCE (CVE-2007-2447) |
| [Outbound](./hackthebox/outbound.md) | Easy | Roundcube RCE, 3DES session decrypt, sudo CVE |
| [Sau](./hackthebox/sau.md) | Easy | SSRF pivot, unauth command injection, pager escape |
| [Shocker](./hackthebox/shocker.md) | Easy | Shellshock (CVE-2014-6271), GTFOBins Perl |
| [Soulmate](./hackthebox/soulmate.md) | Easy | Vhost discovery, CrushFTP bypass, Erlang shell abuse |
| [UpDown](./hackthebox/updown.md) | Medium | `.git` leak, header bypass, `phar://` bypass, SUID Python2 |

### Windows

| Machine | Difficulty | Techniques |
|---|---|---|
| [Optimum](./hackthebox/optimum.md) | Easy | Version-based RCE, kernel exploit |
| [Blue](./hackthebox/blue.md) | Easy | EternalBlue / MS17-010 |
| [Devel](./hackthebox/devel.md) | Easy | Anonymous FTP → ASPX RCE, kernel privesc |
| [Heist](./hackthebox/heist.md) | Easy | Cisco config crack, RID brute + spray, Procdump Firefox, Pass-the-Password |
| [Jerry](./hackthebox/jerry.md) | Easy | Default Tomcat manager creds, WAR upload RCE |
| [Legacy](./hackthebox/legacy.md) | Easy | MS08-067 |
| [Aero](./hackthebox/aero.md) | Medium | CVE-2023-38146 (ThemeBleed) — foothold |
| [Jeeves](./hackthebox/jeeves.md) | Medium | Unauth Jenkins Groovy RCE, KeePass, Pass-the-Hash, NTFS ADS |

### TryHackMe

| Room | Difficulty | Techniques |
|---|---|---|
| [Alfred](./tryhackme/alfred.md) | Easy | Jenkins default creds, Nishang reverse shell, Windows token impersonation |
| [Anonymous](./tryhackme/anonymous.md) | Easy | Anonymous/writable FTP, SUID `env` |
| [Brooklyn Nine Nine](./tryhackme/brooklyn-nine-nine.md) | Easy | FTP credential leak, SUID `less` |
| [Easy Peasy](./tryhackme/easy-peasy.md) | Easy | Layered encoding, hash cracking, steganography |
| [Game Zone](./tryhackme/game-zone.md) | Easy | SQL injection, John password crack, SSH port-forward |
| [Ice](./tryhackme/ice.md) | Easy | Icecast RCE, UAC bypass, Mimikatz/Kiwi |
| [Ignite](./tryhackme/ignite.md) | Easy | Fuel CMS authenticated RCE |
| [Kenobi](./tryhackme/kenobi.md) | Easy | SMB/NFS enum, ProFTPD `mod_copy`, SUID path hijack |
| [LazyAdmin](./tryhackme/lazyadmin.md) | Easy | CMS credential reuse, writable `sudo` script |
| [Overpass 2 - Hacked](./tryhackme/overpass-2-hacked.md) | Easy | PCAP forensics, hash cracking, cron-hijack timeline |
| [Probe](./tryhackme/probe.md) | Easy | Multi-service fingerprinting (enumeration only) |
| [RootMe](./tryhackme/rootme.md) | Easy | Upload filter bypass, SUID Python |
| [Steel Mountain](./tryhackme/steel-mountain.md) | Easy | HttpFileServer RCE (CVE-2014-6287), unquoted service path |
| [ToolsRus](./tryhackme/toolsrus.md) | Easy | Basic-auth brute-force, Tomcat manager RCE |
| [Vulnversity](./tryhackme/vulnversity.md) | Easy | Upload extension bypass, SUID `systemctl` |
| [Attacktive Directory](./tryhackme/attacktive-directory.md) | Medium | Kerbrute enum, AS-REP Roasting, DCSync via backup account |
| [Brainpan 1](./tryhackme/brainpan-1.md) | Medium | Stack buffer overflow, EIP control, custom shellcode |
| [Brainstorm](./tryhackme/brainstorm.md) | Medium | Windows buffer overflow, SEH restricted, `msfvenom` |
| [Daily Bugle](./tryhackme/daily-bugle.md) | Medium | Joomla SQLi, `yara` cracked, Yum sudo abuse |
| [Gatekeeper](./tryhackme/gatekeeper.md) | Medium | Windows buffer overflow with badchars, Firefox creds |
| [HackPark](./tryhackme/hackpark.md) | Medium | BlogEngine plugin RCE, WinPEAS, SYSTEM privesc |
| [Mr Robot CTF](./tryhackme/mr-robot-ctf.md) | Medium | WordPress theme RCE, SUID `nmap` interactive |
| [PrintNightmare](./tryhackme/printnightmare.md) | Medium | CVE-2021-1675/34527, Event Log/Sysmon threat hunting |
| [Relevant](./tryhackme/relevant.md) | Medium | Anonymous SMB creds, SeImpersonatePrivilege, PrintSpoofer |
| [Skynet](./tryhackme/skynet.md) | Medium | Anonymous SMB, Squirrelmail, Cuppa CMS LFI/RFI, `tar` wildcard cron |
| [Internal](./tryhackme/internal.md) | Hard | wpscan brute, WP theme editor RCE, SSH pivot to Jenkins, Docker root |
| [Retro](./tryhackme/retro.md) | Hard | WordPress + Hydra, RDP, CVE-2019-1388 UAC bypass |

### Others (TCM Security / VulnHub practice)

| Machine | Difficulty | Techniques |
|---|---|---|
| [Kioptrix Level 1](./others/kioptrix-1.md) | Easy | Apache mod_ssl OpenFuck (CVE-2002-0082), Samba RCE |
| [TCM Dev](./others/tcm-dev.md) | Easy | Directory enum, credential reuse, sudo `find` |
| [TCM Butler](./others/tcm-butler.md) | Easy | Jenkins default creds, Groovy shell, Windows privesc |
| [TCM Black Pearl](./others/tcm-black-pearl.md) | Medium | Chained web enum, DB creds, sudo abuse |

---

## ⚠️ Publishing rules per platform (READ before adding anything)

**Hack The Box** — **retired machines only**. Never active machines — HTB's
terms explicitly prohibit it and reserve the right to pursue legal action.
Policy is checked at publish time since it can change; see
[HTB's write-up policy](https://help.hackthebox.com/en/articles/5188925-can-i-create-write-ups-about-hackthebox-content).

**TryHackMe** — generally the most permissive; many rooms explicitly
encourage write-ups. Check the specific room's description regardless.

**Proving Grounds** — **no write-ups published in this repo.** OffSec's own
["Rules of the Game"](https://help.offsec.com/hc/en-us/articles/360048114312-Rules-of-the-Game)
asks users to refrain from sharing information about PG Practice machines,
and the Practice pool includes retired OSCP exam machines — not worth the
risk to the certification. Lab progress/stats are still fine to share (see
the main README); the write-ups themselves are the part that stays private.

**Golden rule:** when in doubt, skip it. No writeup is worth risking a
platform account or a certification.
