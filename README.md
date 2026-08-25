# TryHackMe Writeups

I started keeping these writeups mostly for myself, as a way to force a clear explanation out of what I'd just done on a room. Publishing them turned out to be a nice side effect.

Profile: [tryhackme.com/p/us3rgnv](https://tryhackme.com/p/us3rgnv) — 144 rooms completed, rank 30,752

Flags are redacted in every writeup (`THM{REDACTED}`), in line with TryHackMe's rules.

## Index

### Linux

| Room | Difficulty | Focus | Writeup |
|---|---|---|---|
| HA: Joker CTF | Medium | Joomla, content discovery | [link](./linux/HA%20jokerCTF-writeup.md) |
| Blog | Medium | WordPress CVE exploitation | [link](./linux/blog.thm-writeup.md) |
| hc0n Christmas CTF | Hard | Crypto, reversing, mobile, ROP chain | [link](./linux/hc0n-writeup.md) |
| Mountaineer Linux | Hard | WordPress, path traversal | [link](./linux/mountaineer.thm-writeup.md) |
| Mr. Robot CTF | Medium | WordPress, privilege escalation | [link](./linux/mrrobot.thm-writeup.md) |
| Year of the Fox | Hard | Samba masquerading as Windows | [link](./linux/yearofthefox.thm-writeup.md) |

### Windows

| Room | Difficulty | Focus | Writeup |
|---|---|---|---|
| Attacktive Directory | Easy (beginner-friendly) | Kerberos enum, AS-REP Roasting, DCSync | [link](./windows/AttacktiveDirectory-writeup.md) |
| Fusion Corp | Medium | AS-REP Roasting, NTDS.DIT extraction | [link](./windows/FusionCorp-writeup.md) |
| Ledger | Hard | AD CS misconfiguration | [link](./windows/Ledger.thm-writeup.md) |

## How each writeup is organized

Recon first, then enumeration, then whatever got me the foothold, then privilege escalation, then a short takeaway at the end. The takeaway is the part I care about most — it's one or two lines on what actually mattered in the room, not a recap of every command I ran.

## Elsewhere

[HackTheBox writeups](https://github.com/us3rgnv/HackTheBox-writeups) · [Medium](https://medium.com/@us3rgnv) · [LinkedIn](https://www.linkedin.com/in/fatima-gurbanova-650a11350)

---

*For educational purposes only. Don't run any of this against systems you don't have explicit permission to test.*
