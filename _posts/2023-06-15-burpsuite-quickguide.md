---
layout: post
title: "BurpSuite Repeater/Intruder Power Tips"
subtitle: "Advanced workflows, payload tricks, bypass techniques and hidden features for HTB, CTFs and real-world web exploitation"
tags: [burp, intruder, repeater, web, exploitation, hacking, htb, ctf]
---

A practical and advanced cheat sheet for mastering BurpSuite Repeater and Intruder during HTB, CTFs, bug bounties and real-world web exploitation.

---

# 0. ESSENTIAL SETTINGS & MUST-ENABLE FEATURES

## Use “Beautify” / “Pretty” View
- JSON ✓  
- XML ✓  
- HTML ✓  
- URL-encoded ✓  

Improves visibility of injection points.

## Disable “Update Content-Length”
Helps in:
- HTTP Request Smuggling  
- Manual boundary tampering  
- Chunked/CL desync tests  

## Turn on “Invisible mode” when needed
Useful for:
- Non-proxy aware clients  
- Raw socket fuzzing through Burp  

---

# 1. REPEATER POWER TIPS (ADVANCED)

# 1.1 Quickly clone tabs
Right click → **Duplicate tab**.  
Useful for parallel payload testing.

---

# 1.2 Send requests to Repeater with decoded/encoded bodies
Repeater → right side → “Inspector” → toggle:
- URL encoding  
- Base64  
- JSON keys/values  
- Hex  

You can modify nested payloads that raw mode hides.

---

# 1.3 Auto-discovery of insertion points (JSON/XML/Multipart)
Repeater → Inspector → Click on any JSON/XML node  
→ “Insert partnered request”  
→ auto-maps payload positions.

Perfect for:
- SSTI  
- XSS  
- SQLi inside nested JSON  

---

# 1.4 Burp’s CRLF injection helper
Type:
```

%0d%0a

```
Raw mode automatically shows line breaks.

Used for:
- Cache poisoning  
- Header injection  
- Host override attacks  

---

# 1.5 HTTP Request Smuggling via Repeater
Modify:
```

Content-Length:
Transfer-Encoding:

```

Tricks:
- Insert invalid chunk sizes  
- Extra Content-Length headers  
- Keep-alive pipe requests  

Example:
```

POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

0

G

```

---

# 1.6 Manual Param Mining
Use Repeater to fuzz interesting params manually:
```

?debug=true
?admin=1
?role=superuser
?preview=1

```

Often results in:
- Debug pages  
- Admin panels  
- Verbose errors → exploit chain  

---

# 1.7 Unicode/Encoding Bypass Techniques
Try replacing characters with:
- `%u0131`  
- full-width unicode  
- homoglyphs (Greek/Arabic lookalikes)  

Useful for WAF bypass.

---

# 1.8 Swap HTTP methods
Check:
```

OPTIONS
HEAD
PUT
DELETE
PROPFIND

```

Sometimes bypasses auth:
```

PUT /admin/ HTTP/1.1

```

---

# 2. INTRUDER POWER TIPS (ADVANCED)

# 2.1 Choose correct attack types

### Sniper (default)
Single position → test single input.

### Battering Ram
Multiple positions share same payload.  
Useful for:
- Bypassing multi-param auth  
- Sending same token in all fields  

### Pitchfork
Multiple payload lists, synced line-by-line.

### Clusterbomb
Full combinatorial brute-force.  
Useful for:
- Login bruteforce  
- Parameter brute-force  
- WAF evasion combos  

---

# 2.2 Grep-Extract (underrated)
Extract dynamic values from body/header.  
Examples:
- Anti-CSRF tokens  
- OAuth state values  
- One-time nonces  
- Hidden fields  

Great for staged exploits.

---

# 2.3 Grep-Match for automation
Automatically highlight responses containing:
```

admin
success
flag
{"
"role":

```

Blazing fast for CTFs.

---

# 2.4 Turbo Intruder (REPLACES Classic Intruder for speed)
From PortSwigger labs:
```

Extensions → BApp Store → Turbo Intruder

```

Advantages:
- Millions of requests  
- Custom Python logic  
- Parallel threading  
- Race condition exploitation  
- Param-mining at scale  

---

# 2.5 Payload Obfuscation Tricks

## Random case transformation
```

uSeR=admin

```

## Character padding
```

admin/**/
admin%20
admin\t

```

## Encode payloads multiple times
```

%2527
%255c%255c

```

---

# 2.6 Useful Intruder Payload Lists
Use from **SecLists**:

### LFI paths
```

/etc/passwd
../../../../windows/win.ini

```

### SQLi
```

' OR 1=1--
" OR ""="

```

### SSTI
```

{{7*7}}
${{7*7}}

```

### Admin parameter discovery
```

admin
debug
test
beta
super
flag

```

---

# 2.7 Fuzz non-obvious fields

## Cookies  
```

Cookie: role=admin
Cookie: debug=true

```

## Hidden HTML fields  
Use:
```

grep -R "<input type="hidden""

```

## Headers  
Try fuzzing:
```

X-Forwarded-For:
X-Originating-IP:
X-Host:
X-Forwarded-Host:
X-HTTP-Method-Override:

```

---

# 2.8 Intruder Race Condition Attacks (fast)
Turbo Intruder example script:
```python
engine=Engine.BURP
requests=Range(1,50)
```

Useful for:

* Bypassing purchase limits
* Double-spending
* Logic breaks

---

# 3. ADVANCED BYPASS TECHNIQUES

# 3.1 Bypass JSON-based WAFs

Try:

```
"username":"admin\t"
"username":["admin"]
"username": {"$ne":""}
```

# 3.2 Parameter Pollution

```
user=admin&user=guest
```

# 3.3 Header Override

```
X-Original-URL: /admin
X-Rewrite-URL: /admin
```

# 3.4 Cache Poisoning via Intruder

Fuzz:

```
Host:
X-Forwarded-Host:
```

Look for:

```
X-Cache: hit
```

---

# 4. BURP PRO TIPS (HIDDEN FEATURES)

## 4.1 Match & Replace

```
Replace User-Agent automatically  
Replace Referer  
Replace Host  
```

## 4.2 Custom MIME types

Enable detection of content not recognized by Burp.

## 4.3 Upstream Proxy Rules

Chain requests internally for pivoting.

## 4.4 Logger++ (must install)

* Full traffic logging
* Regex tagging
* Huge for CTFs

---

# Final Notes

This cheat sheet covers:

* Advanced BurpSuite Repeater workflows
* Payload encoding & bypass techniques
* Manual & automated fuzzing with Intruder
* Turbo Intruder for speed & race attacks
* Parameter mining, WAF bypass, JSON/XML fuzzing
* HTTP smuggling and cache poisoning using Repeater
* Header manipulation tricks
* Productivity shortcuts

Perfect for HTB web machines, OSCP/OSWE, bug bounties and real-world assessments.

---