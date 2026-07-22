# Boolean — Proving Grounds

| | |
|---|---|
| **Platform** | Proving Grounds (Practice) |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Status** | ✅ Practice machine, no overlap with the OSCP exam pool |
| **Key techniques** | Client-side validation bypass via Burp, path traversal to SSH key deployment, key-based root access |

---

## TL;DR

Boolean's registration flow requires email confirmation before an account is
usable — but that confirmation check is enforced client-side only. A
tampered registration request flips the account straight to confirmed,
unlocking a file manager with a path-traversal download/browse feature. That
traversal is used to plant an SSH public key directly into a user's
`.ssh/authorized_keys`, and a leftover root SSH key sitting in that same
user's home directory hands over root with one more SSH login.

---

## Recon & Enumeration

```bash
sudo nmap -sCV -p- <TARGET_IP>
```

An interesting non-standard port stood out alongside the web login page on
port 80.

---

## Foothold / Initial Access

Registering a test account on the web app returned "must be confirmed" —
account creation succeeds, but the account isn't usable until confirmed
(normally via an emailed link). Intercepting the registration request in
Burp showed the confirmation state was just a request parameter, sent as
`false`. Appending `&user[confirmed]=True` to the same request flipped the
account to confirmed without any email step at all — the check was never
enforced server-side.

Logging in with the now-confirmed account exposed a file manager that
allowed uploads, including a PHP web shell. Clicking the uploaded file
revealed the download link took a raw file path as a parameter
(`?cwd=&file=shell.php&download=true`) — testing path traversal on that
parameter confirmed arbitrary file read (`/etc/passwd` came back cleanly).

Since the traversal allowed *reading* arbitrary paths, the next question was
whether the same file manager's upload could *write* into an arbitrary path
too. Generating an SSH keypair locally and uploading the public key,
renamed to `authorized_keys`, into a target user's `.ssh/` directory via the
traversal path worked — planting a working SSH key without ever touching a
password:

```bash
ssh-keygen -f remi
mv remi.pub authorized_keys
```

```
http://<TARGET_IP>/?cwd=../../../../../home/remi/.ssh
```

```bash
ssh -i remi remi@<TARGET_IP>
```

User flag retrieved.

---

## Privilege Escalation

A `.bash_aliases` file in the user's home directory hinted at root SSH key
authentication being expected, and a root private key was found sitting in
the user's own `~/.ssh/keys` directory — a leftover/misplaced credential
rather than anything that needed exploiting.

Authenticating with that key directly failed with "too many authentication
failures" — a common issue when the SSH client offers multiple identity
files before the intended one, hitting the server's `MaxAuthTries` limit.
Forcing the client to offer only the specified key fixed it:

```bash
ssh -i root_key -o IdentitiesOnly=yes root@<TARGET_IP>
```

Root shell obtained, root flag retrieved.

---

## Lessons Learned

- **Never trust a client-side "confirmed" flag** — always intercept and
  tamper with state-changing parameters in Burp before assuming a workflow
  gate is real.
- A path-traversal **read** primitive is worth testing for a **write**
  primitive too, especially through the same feature (a file browser/upload
  combo) — here it was the difference between reading `/etc/passwd` and
  planting a working SSH key.
- `Error: too many authentication failures` from `ssh` is a client-side
  offering-order problem, not a credential problem — `-o
  IdentitiesOnly=yes` resolves it when multiple keys are available.

---

## Remediation

- Enforce account-confirmation state server-side; never trust a
  client-supplied "confirmed" parameter.
- Sanitize file manager path parameters against traversal
  (`../`) on both read and write operations.
- Remove leftover credential material (private keys, password files) from
  user home directories once no longer needed, and restrict `.ssh` write
  access to the account owner only.

---

**Machine:** [Proving Grounds — Boolean](https://portal.offsec.com/labs/play)
