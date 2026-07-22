# [Machine name] — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds |
| **Difficulty** | Easy / Medium / Hard |
| **OS** | Linux / Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | e.g. SQLi, GTFOBins, Kerberoasting |

> ⚠️ Publish Practice machines only, and never anything overlapping with the
> OSCP exam machine pool or the exam itself.

---

## TL;DR

2-3 sentences: what the machine is and the path from nothing to root.
(Recruiters often read only this — make it good.)

---

## Recon & Enumeration

What you scanned, what you found. **Most important: WHY** you looked where you looked.

```bash
# example commands (scrubbed of your real IPs/tokens)
nmap -sC -sV -oA nmap/initial <TARGET_IP>
```

Observation → conclusion. "Saw port X running service Y, so..."

---

## Foothold / Initial Access

How you got the first shell. Step by step, with **your reasoning**.

- What you tried first and why
- What didn't work (that's valuable too!)
- What ultimately worked

---

## Privilege Escalation

The path from user to root/SYSTEM.

- Local enumeration: what you looked for
- The vector you found and why it was vulnerable
- Exploitation

---

## Lessons Learned

- What you learned
- What you'd do faster next time
- Real-world relevance (why this matters outside a lab)

---

## Remediation

How a defender should fix this. (Shows you think defensively too — a plus for recruiters.)
