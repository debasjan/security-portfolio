# ToolsRus — TryHackMe

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Room explicitly aimed at write-ups/beginners |
| **Key techniques** | Directory brute-forcing, HTTP basic-auth brute-forcing (Hydra), Apache Tomcat/Coyote fingerprinting, public Metasploit RCE |

---

## TL;DR

A tool-survey style room: directory brute-forcing finds a
basic-auth-protected path, Hydra brute-forces the password, and that
credential unlocks a second web service (Apache Tomcat) on a non-standard
port. A version-matched Metasploit module against Tomcat's manager
interface delivers a root shell directly.

---

## Recon & Enumeration

```bash
nmap -sC -sV -p- <TARGET_IP>
gobuster dir -u http://<TARGET_IP> -w <wordlist>
```

Brute-forcing the default web root found a directory revealing a name
(hinting at a valid username), and a separate path protected by HTTP basic
authentication. A second web service was found running on a non-standard
port, identified as **Apache Tomcat 7.0.88 / Apache-Coyote 1.1**.

---

## Foothold / Initial Access

Basic-auth credentials for the protected directory were brute-forced with
Hydra against the username found earlier:

```bash
hydra -l <user> -P /usr/share/wordlists/rockyou.txt -f <TARGET_IP> http-get /protected -V
```

The recovered password, combined with Tomcat's well-known default manager
path, was used with **Nikto** to scan `/manager/html` and confirm valid
Tomcat manager access. With valid manager credentials and the exact
Tomcat/Coyote version identified, Metasploit's Tomcat manager exploitation
module deployed a malicious WAR through the manager interface and executed
it directly:

```
msfconsole
# select and configure a Tomcat manager deployment/exploit module
```

Shell returned as **root** directly — no separate privilege escalation
phase was needed on this box.

---

## Lessons Learned

- **Basic-auth-protected directories are a Hydra target, not a dead end** —
  once a candidate username is known (from an earlier enumeration step),
  brute-forcing the password against `http-get` is a quick, standard
  attempt.
- **Tomcat manager access (whether via default, brute-forced, or leaked
  credentials) is close to direct code execution** — WAR file deployment
  through the manager interface is a well-worn, reliable path, and
  Metasploit automates it end-to-end once the version and credentials are
  known.

---

## Remediation

- Never protect sensitive paths with HTTP basic auth alone if the password
  is weak enough for a wordlist attack; pair with account lockout or
  stronger authentication.
- Restrict or disable the Tomcat manager application in production, and if
  required, place it behind network-level access control in addition to
  credentials.

---

**Room:** [TryHackMe — ToolsRus](https://tryhackme.com/room/toolsrus)
