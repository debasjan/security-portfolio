# Expressway — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Retired |
| **Key techniques** | UDP service discovery, IKE/IPsec PSK cracking, `sudo` chroot privilege escalation (CVE-2025-32463) |

---

## TL;DR

Expressway is one of the few boxes in this set where the entire foothold lives
in UDP rather than TCP — a default `nmap` scan shows almost nothing until a
UDP sweep reveals an IKE/IPsec VPN endpoint. Forcing IKEv1 Aggressive Mode
leaks enough material to crack the pre-shared key offline, which doubles as
an SSH password. Root comes from a very recent `sudo` chroot vulnerability.

---

## Recon & Enumeration

```bash
nmap -A -T4 10.10.11.87
```

A default TCP scan showed almost nothing beyond SSH — a strong hint that the
real attack surface wasn't TCP at all. A UDP sweep confirmed that:

```bash
nmap -sU -T4 10.10.11.87
```

Port **500/udp** was open, identified as `isakmp` — the negotiation protocol
for IKE/IPsec VPNs. This is easy to miss entirely if UDP scanning isn't a
standard part of the recon pass, since TCP-only enumeration would report this
box as effectively a single-service SSH box.

---

## Foothold / Initial Access

`ike-scan` is the purpose-built tool for talking to an IKE endpoint. Forcing
**Aggressive Mode** (an older, faster IKEv1 mode that trades security for
speed) causes the server to leak identity and cryptographic material that
Main Mode would keep hidden — including data that enables offline cracking of
the pre-shared key:

```bash
ike-scan --id=1 -A 10.10.11.87
ike-scan --id=1 -A -P psk.txt 10.10.11.87
```

The response confirmed Aggressive Mode support, an older cipher suite
(3DES/SHA1/DH group 2), and an identity of `ike@expressway.htb` — a username
in disguise. With the handshake material saved, `psk-crack` ran an offline
dictionary attack:

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt psk.txt
```

The recovered PSK doubled as the `ike` user's SSH password — VPN
pre-shared keys and account passwords being the same value is a realistic
(if poor) practice this box is clearly modeling. User flag retrieved.

---

## Privilege Escalation

`linpeas` (already present on the box under `/tmp`, left by presumably an
earlier session or the box's own setup) flagged the installed `sudo`
version, **1.9.17**, as vulnerable to **CVE-2025-32463** — a very recently
disclosed "chroot to root" bug in `sudo`'s `-R`/`--chroot` option. Older
vulnerable versions would switch into an attacker-specified chroot directory
*before* fully evaluating privileges, meaning a writable directory (say,
under `/tmp`) containing a fake `/etc/nsswitch.conf` and a malicious
`libnss_*.so` gets loaded as root once `sudo` chroots into it — turning any
`sudo` invocation, regardless of the actual command allowed, into arbitrary
code execution.

A public PoC for CVE-2025-32463 automated building that fake chroot structure
and triggering the load. Running it delivered a root shell and the root flag.

---

## Lessons Learned

- **A near-empty TCP scan is a reason to run UDP, not a reason to stop
  looking** — this entire box lives behind port 500/udp.
- **IKE Aggressive Mode is a known weak point** — it exists for legacy
  compatibility, but it leaks exactly the material needed for offline PSK
  cracking, unlike Main Mode.
- **`sudo -V` is a two-second check with an outsized payoff** — a brand-new
  CVE at the time of this box's release was already fully weaponized.

---

## Remediation

- Disable IKEv1 Aggressive Mode; require Main Mode or migrate to IKEv2, which
  doesn't have this specific information leak.
- Use strong, high-entropy pre-shared keys, and don't reuse VPN PSKs as
  account passwords anywhere.
- Patch `sudo` promptly — chroot-related privilege escalation bugs recur
  periodically and are typically fixed fast once disclosed; staying current
  is the actual mitigation.

---

**Machine:** [Hack The Box — Expressway](https://www.hackthebox.com/machines/expressway)
