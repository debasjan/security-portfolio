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
| [Timelapse](./hackthebox/timelapse.md) | Easy | Archive/PFX cracking, cert-based auth, LAPS abuse |
| [Administrator](./hackthebox/administrator.md) | Medium | ACL chaining, password-manager cracking, targeted Kerberoast, DCSync |
| [Monteverde](./hackthebox/monteverde.md) | Medium | Anonymous LDAP, password spray, Azure AD Connect extraction |
| [Resolute](./hackthebox/resolute.md) | Medium | Anonymous enum, PS transcript leak, DnsAdmins abuse |

### Linux

| Machine | Difficulty | Techniques |
|---|---|---|
| [Cap](./hackthebox/cap.md) | Easy | IDOR, pcap credential extraction, Linux capabilities |
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
| [Jerry](./hackthebox/jerry.md) | Easy | Default Tomcat manager creds, WAR upload RCE |
| [Legacy](./hackthebox/legacy.md) | Easy | MS08-067 |

### Proving Grounds (Practice)

| Machine | Difficulty | Techniques |
|---|---|---|
| [Access](./proving-grounds/access.md) | Easy | Upload filter bypass via `.htaccess`, SPN enum, Kerberoasting |
| [Algernon](./proving-grounds/algernon.md) | Easy | Public pre-auth RCE (SmarterMail) |
| [AuthBy](./proving-grounds/authby.md) | Easy | Anonymous FTP creds, offline hash cracking, Juicy Potato |
| [Boolean](./proving-grounds/boolean.md) | Easy | Client-side validation bypass, path traversal, SSH key deploy |
| [Craft](./proving-grounds/craft.md) | Medium | LibreOffice macro RCE, writable webroot, PrintSpoofer |
| [Extplorer](./proving-grounds/extplorer.md) | Easy | Default creds, config credential leak, `disk` group + `debugfs` |
| [Heist](./proving-grounds/heist.md) | Medium | SSRF-triggered NTLM capture, BloodHound, GMSA read, `SeRestorePrivilege` |
| [Hutch](./proving-grounds/hutch.md) | Medium | Anonymous LDAP password leak, WebDAV upload, PrintSpoofer |
| [Internal](./proving-grounds/internal.md) | Easy | Public SMB RCE (CVE-2009-3103) |
| [Jacko](./proving-grounds/jacko.md) | Easy | Unauthenticated H2 console RCE, GodPotato |
| [Kevin](./proving-grounds/kevin.md) | Easy | Default creds, public Metasploit module (CVE-2009-3999) |
| [Nickel](./proving-grounds/nickel.md) | Medium | API info disclosure, PDF cracking, localhost-only SYSTEM endpoint |
| [Pelican](./proving-grounds/pelican.md) | Easy | Unauthenticated command injection, `sudo gcore` memory dump |
| [Resourced](./proving-grounds/resourced.md) | Medium | NTDS leak via SMB share, Resource-Based Constrained Delegation |
| [Shenzi](./proving-grounds/shenzi.md) | Easy | SMB null session, WordPress Theme Editor RCE, `AlwaysInstallElevated` |
| [Slort](./proving-grounds/slort.md) | Easy | LFI-to-RFI, scheduled-task binary replacement |
| [Squid](./proving-grounds/squid.md) | Medium | Port discovery through a proxy, phpMyAdmin default creds, FullPowers |
| [Twiggy](./proving-grounds/twiggy.md) | Easy | Unauthenticated pre-auth RCE (SaltStack) |

### TryHackMe

| Room | Difficulty | Techniques |
|---|---|---|
| [Anonymous](./tryhackme/anonymous.md) | Easy | Anonymous/writable FTP, SUID `env` |
| [Attacktive Directory](./tryhackme/attacktive-directory.md) | Medium | Kerbrute enum, AS-REP Roasting, DCSync via backup account |
| [Brooklyn Nine Nine](./tryhackme/brooklyn-nine-nine.md) | Easy | FTP credential leak, SUID `less` |
| [Easy Peasy](./tryhackme/easy-peasy.md) | Easy | Layered encoding, hash cracking, steganography |
| [Ice](./tryhackme/ice.md) | Easy | Icecast RCE, UAC bypass, Mimikatz/Kiwi |
| [Ignite](./tryhackme/ignite.md) | Easy | Fuel CMS authenticated RCE |
| [LazyAdmin](./tryhackme/lazyadmin.md) | Easy | CMS credential reuse, writable `sudo` script |
| [PrintNightmare](./tryhackme/printnightmare.md) | Medium | CVE-2021-1675/34527, Event Log/Sysmon threat hunting |
| [Probe](./tryhackme/probe.md) | Easy | Multi-service fingerprinting (enumeration only) |
| [RootMe](./tryhackme/rootme.md) | Easy | Upload filter bypass, SUID Python |
| [ToolsRus](./tryhackme/toolsrus.md) | Easy | Basic-auth brute-force, Tomcat manager RCE |

---

## ⚠️ Publishing rules per platform (READ before adding anything)

**Hack The Box** — **retired machines only**. Never active machines — HTB's
terms explicitly prohibit it and reserve the right to pursue legal action.
Policy is checked at publish time since it can change; see
[HTB's write-up policy](https://help.hackthebox.com/en/articles/5188925-can-i-create-write-ups-about-hackthebox-content).

**TryHackMe** — generally the most permissive; many rooms explicitly
encourage write-ups. Check the specific room's description regardless.

**Proving Grounds** — Practice machines are usually fine; **never** anything
that overlaps with the OSCP exam machine pool, and nothing from the exam
itself.

**Golden rule:** when in doubt, skip it. No writeup is worth risking a
platform account or a certification.
