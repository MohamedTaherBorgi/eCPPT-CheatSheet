## Table of Contents
1. [Methodology & Mental Model](#1-methodology--mental-model)
2. [Passive Recon](#2-passive-recon)
3. [Host Discovery](#3-host-discovery)
4. [Nmap — Port Scanning](#4-nmap--port-scanning)
5. [Service Enumeration](#5-service-enumeration)
6. [SMB Enumeration](#6-smb-enumeration)
7. [FTP Enumeration](#7-ftp-enumeration)
8. [SSH Enumeration](#8-ssh-enumeration)
9. [SMTP Enumeration](#9-smtp-enumeration)
10. [SNMP Enumeration](#10-snmp-enumeration)
11. [RPC / NFS Enumeration](#11-rpc--nfs-enumeration)
12. [Web Enumeration](#12-web-enumeration)
13. [DNS Enumeration](#13-dns-enumeration)
14. [LDAP Enumeration](#14-ldap-enumeration)
15. [Quick-Win Checklist](#15-quick-win-checklist)

---

## 1. Methodology & Mental Model

```
Phase Order:
  Passive Recon → Host Discovery → Port Scan → Service ID → Deep Enum per Service → Vuln ID
```

> [!warning]
>**`export`** <u>only applies to current session</u>
>
>See with **`echo $DOMAIN`**

```bash
# Always set target variables at the start of every engagement
export TARGET="10.10.10.5"
export DOMAIN="corp.local"
export LHOST="10.10.14.2"

# Create a structured output directory — keep all results organised
mkdir -p ~/engagements/$TARGET/{scans,loot,exploits,screenshots}
```

```bash
# Attacker mindset — for each service ask:
#   1. Can I authenticate anonymously / with defaults?
#   2. What version is running — is it known-vulnerable?
#   3. Does it leak usernames, shares, configs?
#   4. Can I brute-force or spray it?
#   5. Does it interact with AD?
```

---

## 2. Passive Recon

### 2.1 WHOIS & DNS Lookups

```bash
# Get domain registration info — registrar, org, contacts, name servers
whois $DOMAIN
```

```bash
# Resolve a domain to its IP(s)
host $DOMAIN          #Ask the internet via dns server in /etc/resolv.conf

host $DOMAIN $TARGET  #Ask the target machine
```

```bash
# Reverse lookup — what domain name maps to an IP
host $TARGET
```

```bash
# Query specific DNS record types
nslookup -type=MX $DOMAIN       # mail servers
nslookup -type=NS $DOMAIN       # name servers
nslookup -type=TXT $DOMAIN      # SPF records, internal hints
nslookup -type=CNAME $DOMAIN    # aliases
```

### 2.2 Google Dorking

```bash
# Find exposed login portals on target domain
site:corp.local inurl:login

# Find files with sensitive extensions hosted on the domain
site:corp.local ext:pdf OR ext:xlsx OR ext:docx

# Find exposed config or backup files
site:corp.local ext:conf OR ext:bak OR ext:env

# Find subdomains indexed by Google
site:*.corp.local -www

# Find directory listings
site:corp.local intitle:"index of"
```

### 2.3 TheHarvester — Email & Subdomain Gathering

```bash
# Enumerate emails, subdomains, and hosts from public sources
theHarvester -d $DOMAIN -b all -l 500

# Target specific data sources
theHarvester -d $DOMAIN -b google,bing,linkedin,shodan
```

### 2.4 Shodan

```bash
# Search for the target IP on Shodan — shows open ports, banners, vulns
# CLI: pip install shodan && shodan init <API_KEY>
shodan host $TARGET

# Search for specific service/product on a domain
shodan search "hostname:$DOMAIN apache"

# Find all IPs in an org's ASN range
shodan search "org:TargetCorp"
```

### 2.5 Recon-ng

```bash
# Launch Recon-ng framework for automated OSINT
recon-ng

# Install modules and run a domain recon workflow
[recon-ng] > marketplace install all
[recon-ng] > workspaces create $DOMAIN
[recon-ng] > modules load recon/domains-hosts/hackertarget
[recon-ng] > options set SOURCE $DOMAIN
[recon-ng] > run
```

---

## 3. Host Discovery

### 3.1 Nmap Ping Sweep

```bash
# Discover alive hosts with ICMP echo + TCP SYN — fast subnet sweep
nmap -sn 10.10.10.0/24 -oG - | grep "Up" | awk '{print $2}'
```

```bash
# Ping sweep saving results to file
nmap -sn 10.10.10.0/24 --oA ~/engagements/$TARGET/scans/host_discovery
```

```bash
# Disable DNS resolution to speed up host discovery
nmap -sn -n 10.10.10.0/24
```

### 3.2 ARP Scan (LAN — Most Reliable)

```bash
# ARP scan — finds hosts even if ICMP is blocked, works on local subnet only
sudo arp-scan -l
sudo arp-scan 10.10.10.0/24
```

```bash
# Netdiscover — passive and active ARP discovery
sudo netdiscover -r 10.10.10.0/24
sudo netdiscover -i eth0 -p   # passive mode — just listen
```

### 3.3 Fping

```bash
# Fast parallel ping sweep — quicker than nmap for live host discovery
fping -a -g 10.10.10.0/24 2>/dev/null
```

```bash
# Output only alive hosts to a file
fping -a -g 10.10.10.0/24 2>/dev/null > alive_hosts.txt
```

---

## 4. Nmap — Port Scanning

### 4.1 Standard Scanning Workflow

```bash
# Phase 1: Fast top-1000 port scan to quickly identify open ports
nmap -sV -sC -Pn -T4 $TARGET -oA ~/engagements/$TARGET/scans/quick
```

```bash
# Phase 2: Full port scan — don't miss services on high ports
nmap -p- -Pn -T4 $TARGET -oA ~/engagements/$TARGET/scans/full
```

```bash
# Phase 3: Targeted deep scan on discovered open ports
nmap -sV -sC -Pn -p 22,80,443,445 $TARGET -oA ~/engagements/$TARGET/scans/targeted
```

### 4.2 Nmap Flag Reference

```bash
# -sS  TCP SYN scan (default, requires root) — stealthy, fast
# -sT  TCP connect scan — works without root, louder
# -sU  UDP scan — slow but finds DNS, SNMP, TFTP etc.
# -sV  Service/version detection
# -sC  Default script scan (equivalent to --script=default)
# -A   Aggressive: OS detect + version + scripts + traceroute
# -Pn  Skip host discovery (treat host as up) — use when ICMP blocked
# -p-  Scan all 65535 ports
# -T4  Timing template 4 (aggressive) — use on stable networks
# -n   Skip DNS resolution (faster)
# -oA  Output in all formats (normal, XML, grepable)
# -oN  Normal output
# -oG  Grepable output (easy to parse)
```

```bash
# OS detection (requires root)
sudo nmap -O -Pn $TARGET
```

```bash
# UDP scan on common ports — finds SNMP (161), DNS (53), TFTP (69), NTP (123)
sudo nmap -sU -Pn --top-ports 20 $TARGET
```

### 4.3 NSE Scripts

```bash
# Run all scripts in a category
nmap --script=vuln $TARGET           # known vuln checks
nmap --script=auth $TARGET           # auth bypass checks
nmap --script=discovery $TARGET      # service enumeration
nmap --script=brute $TARGET          # brute-force (use carefully)
```

```bash
# Run specific scripts
nmap --script=smb-vuln-ms17-010 $TARGET          # EternalBlue check
nmap --script=smb-enum-shares,smb-enum-users $TARGET
nmap --script=http-title,http-headers $TARGET
nmap --script=ftp-anon,ftp-bounce $TARGET
nmap --script=ssh-auth-methods $TARGET
```

```bash
# Find all scripts for a service
ls /usr/share/nmap/scripts/ | grep smb
ls /usr/share/nmap/scripts/ | grep http
```

### 4.4 Output Parsing

```bash
# Extract open ports from grepable output — use as input for targeted scan
grep "open" ~/engagements/$TARGET/scans/full.gnmap | grep -oP '\d+/open' | cut -d/ -f1 | tr '\n' ','
```

```bash
# View XML output in browser-friendly format
xsltproc ~/engagements/$TARGET/scans/full.xml -o ~/engagements/$TARGET/scans/full.html
```

---

## 5. Service Enumeration

### 5.1 Banner Grabbing

```bash
# Grab service banner with netcat — fast manual fingerprinting
nc -nv $TARGET 21     # FTP
nc -nv $TARGET 22     # SSH
nc -nv $TARGET 25     # SMTP
nc -nv $TARGET 80     # HTTP (send: GET / HTTP/1.0)
nc -nv $TARGET 110    # POP3
```

```bash
# Banner grab with telnet
telnet $TARGET 21
telnet $TARGET 25
```

```bash
# Banner grab with curl (HTTP/HTTPS)
curl -I http://$TARGET           # HEAD request — shows headers
curl -I https://$TARGET -k       # -k ignores SSL cert errors
```

```bash
# Grab SSL certificate info — find domain names and internal hostnames
openssl s_client -connect $TARGET:443 2>/dev/null | openssl x509 -noout -text
```

### 5.2 Searchsploit — Vulnerability Lookup

```bash
# Search for exploits matching a service name/version from nmap output
searchsploit apache 2.4.49
searchsploit openssh 7.4
searchsploit "windows server 2008"
searchsploit smb ms17-010
```

```bash
# Show full path to exploit file
searchsploit -p 42315
```

```bash
# Copy exploit to current directory
searchsploit -m 42315
```

```bash
# Update exploit-db database
searchsploit -u
```

---

## 6. SMB Enumeration

> Key ports: 139 (NetBIOS), 445 (SMB direct)

### 6.1 Nmap SMB Scripts

```bash
# Enumerate SMB version, shares, users, sessions, OS info
nmap -p 139,445 --script=smb-enum-shares,smb-enum-users,smb-os-discovery $TARGET
```

```bash
# Check for EternalBlue (MS17-010) — critical RCE
nmap -p 445 --script=smb-vuln-ms17-010 $TARGET
```

```bash
# Check for other critical SMB vulns
nmap -p 445 --script=smb-vuln-ms08-067,smb-vuln-cve2009-3103 $TARGET
```

```bash
# Full SMB vuln scan
nmap -p 445 --script=smb-vuln* $TARGET
```

### 6.2 SMBClient

```bash
# List shares anonymously (null session)
smbclient -L //$TARGET -N
```

```bash
# Connect to a specific share anonymously
smbclient //$TARGET/SHARENAME -N
```

```bash
# Connect with credentials
smbclient //$TARGET/SHARENAME -U "DOMAIN\username%password"
```

```bash
# Download all files from a share recursively
smbclient //$TARGET/SHARENAME -N -c "prompt OFF; recurse ON; mget *"
```

### 6.3 CrackMapExec

```bash
# Enumerate SMB: OS version, signing, hostname
crackmapexec smb $TARGET
```

```bash
# Enumerate shares with credentials
crackmapexec smb $TARGET -u username -p password --shares
```

```Bash
crackmapexec smb $TARGET -u 'guest' -p '' --rid-brute

crackmapexec smb $TARGET -u 'anonymous' -p '' --rid-brute
```

```bash
# Enumerate logged-on users
crackmapexec smb $TARGET -u username -p password --loggedon-users
```

```bash
# Check if account is local admin (look for "Pwn3d!" in output)
crackmapexec smb 10.10.10.0/24 -u username -p password
```

```bash
# Enumerate with NTLM hash (Pass-the-Hash)
crackmapexec smb $TARGET -u username -H NTHASH --shares
```

> [!warning] <u>With **crackmapexec** and **nxc**</u> <u>**BUT SMBCLIENT CAN WORK**</u>
>**If nmap shows smb2/smb3 : Message signing enabled and required and NOT using smb1, the NULL and Anon/Guest sessions will fail.**

To understand why, you have to distinguish between the **Protocol** (the language) and the **Authentication Policy** (the rules).

#### 1. SMBv1 vs. SMBv3

- **SMBv1:** This protocol was "insecure by design." It allowed a lot of information leakage (like listing users and shares) before you even proved who you were.
    
- **SMBv3:** This is the modern standard. It is designed to be "secure by default." While SMBv3 _can_ technically support null sessions if an administrator explicitly enables them, it is almost never the case on a modern Windows Server (2016/2019/2022).

#### 2. Null vs. Anonymous: The Subtle Difference

In the world of Windows security, these are often used interchangeably, but there's a technical nuance:

- **Null Session:** Connecting with **Username: `""`** and **Password: `""`**.
    
- **Anonymous Session:** Connecting with **Username: `"Anonymous"`** or **`"Guest"`** and **Password: `""`**.

>**If you see SMBv3 and "Signing Required" in Nmap, the "Null Session" (completely empty) is almost certainly dead. However, the Guest account is sometimes left enabled on specific shares (like a "Public" or "Transfer" share) even on SMBv3.**

#### If you have creds :
##### Dumping Local Hashes (SAM)

If **t-skid** has **local administrative privileges** on the machine, you can dump the **SAM** database. This contains the NTLM hashes of local users (like the local `Administrator`).

```Bash
crackmapexec smb 10.114.175.102 -u 't-skid' -p 'tj072889*' --sam
```

##### Dumping Domain Hashes (NTDS.dit) (**DCSYNC**)

If **t-skid** is a **Domain Admin** (or has `DS-Replication-Get-Changes` rights), you can perform a **DRSUTAPI** dump. This pulls the NTLM hashes for **every user in the domain** from the Active Directory database (`NTDS.dit`).

```Bash
crackmapexec smb 10.114.175.102 -u 't-skid' -p 'tj072889*' --ntds
```

> **Note:** This is the "Nuclear Option." If it works, you have compromised every account in the network.

### 6.4 Enum4linux / Enum4linux-ng

```bash
# Users RID brute
enum4linux -r -u 'guest' $TARGET
enum4linux -r -u 'anonymous' $TARGET
```

```bash
# Full SMB/RPC enumeration — users, groups, shares, password policy, OS
enum4linux -a $TARGET
```

```bash
# Faster and cleaner output (preferred)
enum4linux-ng -A $TARGET
```

```bash
# Target specific enumeration types
enum4linux -U $TARGET    # users only
enum4linux -S $TARGET    # shares only
enum4linux -G $TARGET    # groups only
enum4linux -P $TARGET    # password policy
```

### 6.5 SMBMap

```bash
# Map shares with access level — shows READ/WRITE per share
smbmap -H $TARGET
```

```bash
# Authenticate and map shares
smbmap -H $TARGET -u username -p password -d DOMAIN
```

```bash
# Recursively list files in a share
smbmap -H $TARGET -u username -p password -r SHARENAME
```

```bash
# Search for specific file names across shares
smbmap -H $TARGET -u username -p password -R --include-folders -A "*.xml|*.config|*.txt"
```

### 6.6 SYSVOL / NETLOGON — GPP Credential Hunt

```bash
# Mount SYSVOL and search for Group Policy Preference files (may contain creds)
smbclient //$TARGET/SYSVOL -N -c "prompt OFF; recurse ON; mget *"

# Search downloaded SYSVOL for cpassword fields (MS14-025)
grep -r "cpassword" .
```

```bash
# Decrypt GPP cpassword (use gpp-decrypt tool)
gpp-decrypt "ENCRYPTED_CPASSWORD_STRING"
```

---

## 7. FTP Enumeration

> Key port: 21

```bash
# Check for anonymous FTP login — extremely common finding
nmap -p 21 --script=ftp-anon $TARGET
```

```bash
# Connect to FTP and try anonymous login manually
ftp $TARGET
# Username: anonymous
# Password: anonymous@anonymous.com (or blank)
```

```bash
# Download all files from FTP once connected
ftp> binary      # switch to binary mode before downloading
ftp> prompt OFF
ftp> mget *
```

```bash
# Use wget to mirror an entire anonymous FTP site
wget -r ftp://$TARGET/ --ftp-user=anonymous --ftp-password=anonymous
```

```bash
# Check FTP version for vulnerabilities
nmap -p 21 -sV --script=ftp-syst $TARGET
```

```bash
# Brute-force FTP login with Hydra
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://$TARGET -t 4
```

---

## 8. SSH Enumeration

> Key port: 22

```bash
# Identify SSH version and supported auth methods
nmap -p 22 -sV --script=ssh-auth-methods,ssh2-enum-algos $TARGET
```

```bash
# Banner grab — sometimes reveals OS and SSH version
nc -nv $TARGET 22
```

```bash
# Enumerate supported authentication methods for a specific user
nmap -p 22 --script=ssh-auth-methods --script-args="ssh.user=root" $TARGET
```

```bash
# Try default / well-known SSH keys (id_rsa, etc.)
ssh -i /path/to/id_rsa user@$TARGET
```

```bash
# Brute-force SSH with Hydra (use small wordlists, respect lockout)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://$TARGET -t 4
```

```bash
# Try username enumeration (timing-based, version dependent)
# CVE-2018-15473 — OpenSSH < 7.7 leaks valid usernames
python3 ssh_user_enum.py --userList users.txt --host $TARGET
```

```bash
# Check for weak SSH host keys (Debian PRNG vuln — CVE-2008-0166)
searchsploit "debian openssh"
```

---

## 9. SMTP Enumeration

> Key ports: 25, 465, 587

```bash
# Connect to SMTP and check supported commands
nc -nv $TARGET 25
telnet $TARGET 25
```

```bash
# Send EHLO to see available SMTP extensions and server version
EHLO attacker.com
```

```bash
# VRFY command — verify if a user exists on the mail server
VRFY root
VRFY admin
VRFY postmaster
```

```bash
# EXPN command — expand a mailing list alias to see members
EXPN sales
EXPN admins
```

```bash
# Enumerate valid users with SMTP VRFY via Nmap script
nmap -p 25 --script=smtp-enum-users --script-args="smtp-enum-users.methods=VRFY" $TARGET
```

```bash
# Enumerate with smtp-user-enum tool
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/Names/names.txt -t $TARGET
smtp-user-enum -M RCPT -U users.txt -t $TARGET -D $DOMAIN
```

```bash
# Check for open relay (server accepts and forwards email for external domains)
nmap -p 25 --script=smtp-open-relay $TARGET
```

---

## 10. SNMP Enumeration

> Key port: 161 (UDP)

```bash
# Scan for SNMP on common UDP port — often missed in TCP-only scans
sudo nmap -sU -p 161 $TARGET
```

```bash
# Brute-force SNMP community strings (default: public, private, manager)
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $TARGET
```

```bash
# Walk the SNMP MIB tree — dumps system info, running processes, users, software
snmpwalk -c public -v1 $TARGET
snmpwalk -c public -v2c $TARGET
```

```bash
# Query specific OIDs — extract targeted info
snmpwalk -c public -v1 $TARGET 1.3.6.1.4.1.77.1.2.25    # Windows user accounts
snmpwalk -c public -v1 $TARGET 1.3.6.1.2.1.25.4.2.1.2   # running processes
snmpwalk -c public -v1 $TARGET 1.3.6.1.2.1.25.6.3.1.2   # installed software
snmpwalk -c public -v1 $TARGET 1.3.6.1.2.1.6.13.1.3     # open TCP ports
snmpwalk -c public -v1 $TARGET 1.3.6.1.2.1.1.5.0         # hostname
```

```bash
# Use snmp-check for formatted, human-readable SNMP dump
snmp-check $TARGET -c public -v 1
```

```bash
# Nmap SNMP scripts
nmap -sU -p 161 --script=snmp-info,snmp-processes,snmp-win32-users $TARGET
```

---

## 11. RPC / NFS Enumeration

> Key ports: 111 (rpcbind/portmapper), 2049 (NFS)

### 11.1 RPC

```bash
# List all registered RPC services on the target
rpcinfo -p $TARGET
```

> [!warning]
>- `rpcinfo` sends a request to the <u>Portmapper</u> (typically <u>port 111</u>).
>    
>- If port 111 is filtered, `rpcinfo` will wait for a long timeout period before giving up.
>    
>- Looking at your Nmap results, **port 111 is not in your list of open ports.**

```bash
# Enumerate RPC via Nmap
nmap -p 111 --script=rpcinfo $TARGET
```

### 11.2 RPCCLIENT
#### Use `rpcclient`

Since port 135 is confirmed open in your scan, use `rpcclient` to start an interactive session. This is much more powerful for Active Directory enumeration.

```Bash
# Connect with a null session (no username/password)
rpcclient -U "" -N 10.114.180.108
```

Once you get the `rpcclient $>` prompt, you can run commands like:

- `srvinfo` (Server information)
    
- `enumdomusers` (List all users)
    
- `querydominfo` (Domain information)
    
- `netshareenumall` (List all shares)

### 11.2 NFS

```bash
# List NFS exports (mounts shared over the network)
showmount -e $TARGET
```

```bash
# Mount an NFS export to your local machine and browse it
sudo mkdir /mnt/nfs
sudo mount -t nfs $TARGET:/exported/path /mnt/nfs -o nolock
ls -la /mnt/nfs
```

```bash
# Check file permissions — look for world-readable or writable dirs
ls -laR /mnt/nfs/
```

```bash
# If no_root_squash is set, create a SUID binary on the NFS share
# Then execute it from the target to get root
cp /bin/bash /mnt/nfs/bash
chmod +s /mnt/nfs/bash
# On target: /path/to/bash -p → root shell
```

```bash
# Nmap NFS scripts
nmap -p 111,2049 --script=nfs-ls,nfs-showmount,nfs-statfs $TARGET
```

---

## 12. Web Enumeration

> Key ports: 80 (HTTP), 443 (HTTPS), 8080, 8443, 8888

### 12.1 Initial Fingerprinting

```bash
# Get web server headers — reveals server version, framework, cookies
curl -I http://$TARGET
curl -I https://$TARGET -k
```

```bash
# Identify tech stack: server, CMS, frameworks, JS libraries
whatweb http://$TARGET
```

```bash
# Full tech detection with Wappalyzer CLI
wappalyzer http://$TARGET
```

```bash
# Spider the site to find all pages, forms, and links
wget --spider -r http://$TARGET 2>&1 | grep '^--' | awk '{print $3}'
```

### 12.2 Directory & File Brute-Force

```bash
# Gobuster — fast directory brute-force
gobuster dir -u http://$TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak,conf -t 50
```

```bash
# Gobuster with auth headers
gobuster dir -u http://$TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -H "Authorization: Bearer TOKEN"
```

```bash
# Feroxbuster — recursive directory brute-force (finds nested paths)
feroxbuster -u http://$TARGET -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt -r
```

```bash
# Dirsearch — lightweight alternative with smart filtering
dirsearch -u http://$TARGET -e php,html,js,txt,bak
```

```bash
# Nikto — web vulnerability scanner (noisy, good for quick wins)
nikto -h http://$TARGET
```

### 12.3 Virtual Host & Subdomain Discovery

```bash
# Gobuster vhost mode — find virtual hosts on a web server
gobuster vhost -u http://$DOMAIN -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

```bash
# Ffuf — fuzz for virtual hosts via Host header
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://$TARGET -H "Host: FUZZ.$DOMAIN" -fs 0
```

```bash
# Subdomain enumeration with amass
amass enum -d $DOMAIN -passive
amass enum -d $DOMAIN -brute -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
```

### 12.4 CMS Fingerprinting

```bash
# WordPress — enumerate version, users, plugins, themes, vulns
wpscan --url http://$TARGET --enumerate u,p,t,vp --api-token YOUR_API_TOKEN
```

```bash
# WordPress user enumeration only (no API key needed)
wpscan --url http://$TARGET --enumerate u
```

```bash
# Joomla scanner
joomscan --url http://$TARGET
```

```bash
# Droopescan — Drupal/Joomla/SilverStripe/WordPress
droopescan scan drupal -u http://$TARGET
```

---

## 13. DNS Enumeration

> Key port: 53 (TCP/UDP)

```bash
# Forward lookup — resolve hostname to IP
host www.$DOMAIN $DC_IP
```

```bash
# Reverse lookup — resolve IP to hostname
host $TARGET $DC_IP
```

```bash
# Zone transfer attempt — dumps entire DNS zone if misconfigured (huge find)
dig axfr $DOMAIN @$DC_IP
host -t axfr $DOMAIN $DC_IP
```

```bash
# Enumerate all record types
dig ANY $DOMAIN @$DC_IP
```

```bash
# Subdomain brute-force with dnsrecon
dnsrecon -d $DOMAIN -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

```bash
# Full DNS recon: zone transfer, standard records, reverse sweep
dnsrecon -d $DOMAIN -t std,axfr,rvl,brt
```

```bash
# Dnsenum — automated DNS enumeration including zone transfers
dnsenum $DOMAIN --dnsserver $DC_IP
```

```bash
# Find mail servers (MX records) — useful for SMTP testing
dig MX $DOMAIN
```

```bash
# Find name servers (NS records)
dig NS $DOMAIN
```

---

## 14. LDAP Enumeration

> Key ports: 389 (LDAP), 636 (LDAPS), 3268/3269 (Global Catalog)

```bash
# Pull naming contexts — reveals domain structure without creds
ldapsearch -x -H ldap://$TARGET -s base namingcontexts
```

```bash
# Dump all objects with anonymous bind (if allowed)
ldapsearch -x -H ldap://$TARGET -b "DC=corp,DC=local" "(objectClass=*)"
```

```Bash
# Extract the Domain Password Policy
ldapsearch -x -H ldap://$DC_IP -b "DC=vulnnet-rst,DC=local" "(objectClass=domain)" pwdProperties pwdHistoryLength minPwdLength lockoutThreshold
```

```Bash
# Enumerate ALL Domain Users
ldapsearch -x -H ldap://$DC_IP:3268 -b "DC=vulnnet-rst,DC=local" "(objectClass=user)" sAMAccountName | grep sAMAccountName
```

```bash
# Find "Description" Field Leaks
ldapsearch -x -H ldap://$DC_IP -b "DC=vulnnet-rst,DC=local" "(objectClass=user)" description
```

```bash
# Map the "Service Principal Names" (SPNs)
ldapsearch -x -H ldap://$DC_IP -b "DC=vulnnet-rst,DC=local" "(servicePrincipalName=*)" sAMAccountName servicePrincipalName
```
#### Automated Tooling Recommendation

Instead of running manual `ldapsearch` commands, use a tool that parses this into a readable format:

##### **`windapsearch`**: A specialized tool for this exact scenario :

```bash
# Enumerate Domain Users (The -U flag)
python3 windapsearch.py --dc-ip $DC_IP -d vulnnet-rst.local -U
  
# Enumerate Kerberoastable Domain Users
python3 windapsearch.py --dc-ip $DC_IP -d vulnnet-rst.local --user-spns
```

##### `nxc` (NetExec) :

```bash
nxc ldap $DC_IP -u '' -p '' --users
nxc ldap $DC_IP -u '' -p '' --pass-pol
```

##### Other commands :
```bash
# Enumerate users with credentials
ldapsearch -x -H ldap://$TARGET -D "CN=user,DC=corp,DC=local" -w password \
  -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName mail
```

```bash
# Enumerate groups
ldapsearch -x -H ldap://$TARGET -D "CN=user,DC=corp,DC=local" -w password \
  -b "DC=corp,DC=local" "(objectClass=group)" cn member
```

```bash
# Find accounts with no Kerberos pre-auth (AS-REP roastable)
ldapsearch -x -H ldap://$TARGET -D "CN=user,DC=corp,DC=local" -w password \
  -b "DC=corp,DC=local" "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" sAMAccountName
```

```bash
# Find accounts with SPNs set (Kerberoastable)
ldapsearch -x -H ldap://$TARGET -D "CN=user,DC=corp,DC=local" -w password \
  -b "DC=corp,DC=local" "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName servicePrincipalName
```

---

## 15. Quick-Win Checklist

```
HOST DISCOVERY
[ ] Ping sweep / ARP scan entire subnet
[ ] Output live hosts to file for all subsequent scans

PORT SCANNING
[ ] Quick nmap scan (top 1000 ports) → note open ports immediately
[ ] Full port scan -p- running in background
[ ] Targeted service/version scan on discovered ports
[ ] UDP scan (top 20) — don't miss SNMP, DNS, NFS

EASY WINS — CHECK FIRST
[ ] FTP anonymous login (port 21)
[ ] SMB null session (port 445) — shares, users, password policy
[ ] SNMP public community string (UDP 161) — huge info dump
[ ] NFS exports with no_root_squash (port 111/2049)
[ ] LDAP anonymous bind (port 389)
[ ] HTTP: robots.txt, sitemap.xml, /.git/, /backup/, /admin/
[ ] DNS zone transfer (port 53)
[ ] Default credentials on all services

SERVICE ENUMERATION
[ ] SMB: enum4linux-ng, smbmap, crackmapexec
[ ] Web: whatweb, gobuster/feroxbuster, nikto, wpscan (if WordPress)
[ ] SMTP: smtp-user-enum, open relay check
[ ] SSH: version + auth methods
[ ] RPC: rpcinfo -p, showmount -e

VERSION → VULN
[ ] Run searchsploit on every identified version
[ ] Check nmap vuln scripts on key services
[ ] Look up CVEs for exact versions found
[ ] Check HackTricks for service-specific abuse: https://book.hacktricks.xyz
```

---

## Port Quick Reference

| Port | Protocol | Service | Key Check |
|------|----------|---------|-----------|
| 21 | TCP | FTP | Anonymous login, version |
| 22 | TCP | SSH | Version, auth methods, key reuse |
| 25/587 | TCP | SMTP | User enum (VRFY), open relay |
| 53 | TCP/UDP | DNS | Zone transfer, subdomain brute |
| 80/443 | TCP | HTTP/S | Dir brute, CMS, headers |
| 111 | TCP/UDP | RPCBind | NFS exports |
| 139/445 | TCP | SMB | Null session, shares, EternalBlue |
| 161 | UDP | SNMP | Public community string |
| 389/636 | TCP | LDAP | Anonymous bind, user/group enum |
| 2049 | TCP | NFS | Mount exports, no_root_squash |
| 3268/3269 | TCP | Global Catalog | Cross-domain LDAP queries |
| 3306 | TCP | MySQL | Default creds, anonymous login |
| 3389 | TCP | RDP | NLA check, BlueKeep |
| 5985/5986 | TCP | WinRM | Evil-WinRM if creds exist |
| 8080/8443 | TCP | HTTP-alt | Admin panels, dev services |

---

*eCPPTv3 Cheatsheet — 01_recon_enumeration.md*
