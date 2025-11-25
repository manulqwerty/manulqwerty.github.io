---
layout: post
title: "OSINT Capture-The-Flag Cheatsheet"
subtitle: "Fast techniques, tools and workflows for social, metadata, geolocation, infrastructure and digital footprint CTF challenges"
tags: [osint, ctfs, htb, recon, enumeration, investigation]
---

A practical cheat sheet for fast OSINT investigations during HTB, CTFs, TryHackMe and digital footprint challenges.

---

# 0. ESSENTIAL OSINT TOOLS

## Web & Infrastructure
- **Amass**: https://github.com/owasp-amass  
- **Subfinder**: https://github.com/projectdiscovery/subfinder  
- **httpx**  
- **theHarvester**  
- **CRT.sh** (certificate transparency)  

## Social Media / People Search
- **Spiderfoot**  
- **Sherlock / Maigret**  
- **Holehe**  
- **WhatsMyName**: https://whatsmyname.app  
- **NameCheckup**  

## File & Image Metadata
- **exiftool**  
- **strings**  
- **binwalk**  

## Geolocation / Image Analysis
- **Google Lens / Yandex Image Search**  
- **EXIF Viewer**  
- **Geocreepy**  
- **PeakVisor**, **Wikimapia**, **OpenStreetMap**  

## Archived Data / Web History
- **Wayback Machine**  
- **Archive.today**  
- **CommonCrawl**  

## IP/Domain Intel
- **Shodan**  
- **Censys**  
- **URLscan.io**  
- **VirusTotal**  

---

# 1. BASICS: IDENTIFIERS TO COLLECT

Collect:
- Usernames  
- Emails  
- Phone numbers  
- Domains / subdomains  
- IPs  
- Hashes  
- Image metadata  
- Landmarks  
- Timezone hints  
- Vehicle plates / signage  

---

# 2. USERNAME ENUMERATION

## Sherlock
```

sherlock username

```

## Maigret
```

maigret username

```

## WhatsMyName
Cross-platform account detection.

---

# 3. EMAIL INTELLIGENCE

## Social binding
```

holehe [email@example.com](mailto:email@example.com)

```

## Breach checks
- haveibeenpwned  
- dehashed  
- intelx.io  

## Domain associations
```

theHarvester -d domain.com -b all

```

---

# 4. DOMAIN & SUBDOMAIN ENUMERATION

## Subfinder
```

subfinder -d target.com -silent

```

## Amass
```

amass enum -d target.com

```

## CRT.sh
https://crt.sh/?q=target.com

## Alive hosts
```

subfinder -d target.com | httpx -silent

```

---

# 5. IP & HOST INTEL

## Shodan
```

[https://www.shodan.io/host/IP](https://www.shodan.io/host/IP)

```

## Censys
```

[https://search.censys.io/hosts/IP](https://search.censys.io/hosts/IP)

```

## VirusTotal
Check:
- Relations  
- URLs  
- Communicating files  

---

# 6. IMAGE / GEOLOCATION OSINT

## Metadata extraction
```

exiftool image.jpg

```

## Reverse image search
- Yandex  
- Google Lens  
- TinEye  

## GEOINT techniques
- Sun direction  
- Mountain profiles  
- Road marks, vegetation  
- Map comparisons (OSM, Wikimapia, PeakVisor)

---

# 7. SOCIAL MEDIA OSINT

### Twitter/X
```

from:user since:2023

```

### Instagram
Check tagged accounts and EXIF of older posts.

### Reddit
Pivot on post history + username reuse.

---

# 8. OSINT VIA PUBLIC FILES

## Strings extraction
```

strings file.jpg

```

## Extract embedded data
```

binwalk -e file.png

```

## Stego hints
- Color anomalies  
- LSB  
- Extra metadata  

---

# 9. ARCHIVE & HISTORICAL DATA

## Wayback Machine
```

web.archive.org/web/*/target.com

```

## Archive.today
Use for blocked pages.

## CommonCrawl
Check past snapshots.

---

# 10. PHONE NUMBER / COUNTRY INTEL

## Carrier & country lookup
```

[https://numverify.com/](https://numverify.com/)

```

## OSINT pivot
- Add number to WhatsApp  
- Telegram profile picture trick  

---

# 11. WIFI / MAC / BLE OSINT

## MAC vendor lookup
```

[https://maclookup.app/](https://maclookup.app/)

```

## WiGLE
Locate BSSID / SSID on WiGLE.net.

---

# 12. DOCUMENT METADATA

## PDF
```

exiftool file.pdf
pdfinfo file.pdf

```

## Office docs
```

unzip file.docx -d out/
strings out/docProps/*

```

---

# 13. HASH / FILE OSINT

## Hash file
```

sha256sum file
md5sum file

```

## VirusTotal lookup
Get behavior, relations and samples.

---

# 14. GITHUB & PASTE OSINT

## GitHub code search
```

[https://github.com/search?q=username&type=code](https://github.com/search?q=username&type=code)

```

## Paste sites
- pastebin  
- ghostbin  
- grep.app  
- publicwww  

---

# 15. ADVANCED SHODAN FILTERS (INTEGRATED)

Shodan becomes extremely effective with combined filters.

## By IP/range
```

net:1.1.1.0/24
ip:8.8.8.8

```

## By port
```

port:22
port:443
port:8080
port:2375   # Docker
port:9200   # Elasticsearch

```

## By service
```

product:Apache
product:"OpenSSH"
product:"Microsoft IIS"

```

## By version
```

product:Apache version:2.4.49
product:"OpenSSH" version:7.2

```

## By HTML content
```

html:"Index of /"
http.html:"password"
http.title:"login"

```

## By country / city
```

country:ES
city:"Madrid"

```

## By organization or ASN
```

org:"Google"
asn:13335

```

## Vulnerability hunting examples
### Log4Shell
```

product:Java "log4j"

```

### Jenkins unauth
```

http.title:"Dashboard [Jenkins]"

```

### MongoDB exposed
```

port:27017 "MongoDB Server"

```

## SSL certificate filters
```

ssl:"domain.com"
ssl.cert.subject.cn:"*.example.com"
ssl.cert.issuer.cn:"Let's Encrypt"

```

## Combined filters (very powerful)
Example: Hikvision cameras in Spain:
```

product:Hikvision port:80 country:ES

```

Example: Apache 2.4.49 worldwide:
```

product:Apache version:2.4.49

```

Example: Login panels in OVH France:
```

http.title:"admin" org:"OVH" country:FR

```

## Shodan CLI
```

pip install shodan
shodan init <API_KEY>
shodan search "port:22 country:DE"
shodan host 8.8.8.8

```

---

# Final Notes

This cheat sheet provides a complete OSINT workflow:

* Usernames, emails & social accounts  
* Domain, DNS & network enumeration  
* Shodan/Censys advanced intelligence  
* Image/GEOINT analysis  
* Metadata extraction & steganography  
* Archive research & historical web OSINT  
* File, hash, MAC, Wi-Fi and document intel  
* GitHub & paste leak discovery  

Useful for HTB OSINT challenges, CTFs, digital investigations and reconnaissance.

---