# 01 - Reconnaissance & Network Service Enumeration

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Host Discovery Methodologies

Before port scanning, rapidly discover live hosts within the designated target subnet.

### Direct Subnet Sweeps (From Kali / Direct Access)
```bash
# Nmap ICMP and ARP Ping Sweep (Fastest on direct Layer 2/3)
nmap -sn 192.168.100.0/24 -oN live_hosts.txt

# ARP Scan (Best for direct local ethernet segment)
sudo arp-scan -I eth0 --localnet

# FPing (Rapid sweep using ICMP echo requests)
fping -a -g 192.168.100.0/24 2>/dev/null | tee fping_hosts.txt
```

### Subnet Discovery Through a Compromised Pivot Host
When positioned on a pivot host without scanning tools installed:

#### Linux Pivot Host (Pure Bash Ping Sweep):
```bash
for i in $(seq 1 254); do
    (ping -c 1 -W 1 192.168.200.$i >/dev/null 2>&1 && echo "192.168.200.$i is UP") &
done; wait
```

#### Windows Pivot Host (PowerShell Ping Sweep):
```powershell
1..254 | ForEach-Object {
    $ip = "192.168.200.$_"
    if (Test-Connection -ComputerName $ip -Count 1 -Quiet -TimeoutSeconds 1) {
        Write-Output "$ip is UP"
    }
}
```

---

## 🔍 2. Port Scanning Strategies

Adopt a **two-phase scanning workflow** to prevent missing closed/filtered high ports while minimizing timeout delays.

### Phase 1: Rapid All-Port Discovery
```bash
# Scan all 65,535 TCP ports at high rate without pinging
nmap -p- --min-rate 1000 -T4 -Pn $TARGET -oN nmap_all_ports.txt

# Alternative: Rustscan (Ultralight & high-throughput)
rustscan -a $TARGET --range 1-65535 -- -sC -sV -Pn
```

### Phase 2: In-Depth Service & Script Audit
Extract the discovered open ports from Phase 1 and run intensive service detection:
```bash
# Extract comma-separated open port list from nmap output
PORTS=$(grep ^[0-9] nmap_all_ports.txt | cut -d '/' -f 1 | tr '\n' ',' | sed 's/,$//')

# Run targeted NSE vulnerability and version scripts
nmap -p$PORTS -sC -sV -O -Pn $TARGET -oA nmap_deep_audit
```

### Essential UDP Scanning
Do not skip UDP for mission-critical network services (SNMP, DNS, TFTP):
```bash
# High-frequency UDP scan against the top 20 ports
sudo nmap -sU --top-ports 20 -Pn $TARGET -oN nmap_udp_top20.txt

# Specific scan for SNMP, DNS, NTP, TFTP
sudo nmap -sU -p 53,69,123,161 -Pn $TARGET -oN nmap_udp_targeted.txt
```

---

## 🛠️ 3. Service-Specific Deep Enumeration

### 1. SMB & NetBIOS (Ports 139, 445)
SMB is one of the richest sources for initial access, credential leaks, and domain enumeration.

```bash
# Comprehensive SMB audit with NetExec (checks SMB signing, domain, OS version)
netexec smb $TARGET

# Check for Null Sessions and Guest Access
netexec smb $TARGET -u '' -p '' --shares
netexec smb $TARGET -u 'guest' -p '' --shares

# Enum4linux-ng (Modern, clean SMB/RPC enumerator)
enum4linux-ng -A $TARGET -oA enum4linux_report

# Manual share inspection with smbclient
smbclient -L //$TARGET/ -N
smbclient //$TARGET/SharedDocs -N
# Inside smbclient: recurse ON; prompt OFF; mget * (download all files)

# RPCClient Null Session probing
rpcclient -U "" -N $TARGET
# Essential RPCClient commands:
# > enumdomusers           (Enumerate domain users)
# > enumdomgroups          (Enumerate domain groups)
# > queryuser <RID/Name>   (Inspect user details)
# > querygroupmem <RID>    (Inspect members of a group)
```

### 2. SNMP (Port 161 UDP)
Default or predictable SNMP community strings (`public`, `private`) reveal running processes, local users, and internal IP addresses.

```bash
# Brute-force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $TARGET

# Automated SNMP audit with snmp-check
snmp-check -t $TARGET -c public

# Full SNMP MIB Walk (Linux)
snmpwalk -v 2c -c public $TARGET 1.3.6.1.2.1.25.4.2.1.2   # Running processes
snmpwalk -v 2c -c public $TARGET 1.3.6.1.4.1.77.1.2.25   # User accounts
snmpwalk -v 2c -c public $TARGET 1.3.6.1.2.1.6.13.1.3    # Open network TCP ports
```

### 3. LDAP & Active Directory Global Catalog (Ports 389, 636, 3268)
```bash
# Anonymous Bind / Unauthenticated LDAP search
ldapsearch -x -H ldap://$TARGET -b "dc=corp,dc=local" "(objectClass=*)"

# Extract user list using ldapsearch
ldapsearch -x -H ldap://$TARGET -b "dc=corp,dc=local" "(sAMAccountName=*)" sAMAccountName | grep sAMAccountName: | awk '{print $2}' > domain_users.txt

# Extract SPNs (Service Principal Names) via LDAP
ldapsearch -x -H ldap://$TARGET -b "dc=corp,dc=local" "(servicePrincipalName=*)" servicePrincipalName
```

### 4. MSSQL (Port 1433)
```bash
# Check MSSQL info with NetExec
netexec mssql $TARGET -u 'sa' -p 'sa'

# Interactive MSSQL access with Impacket
mssqlclient.py corp.local/jdoe:Password123@$TARGET -windows-auth

# Essential MSSQL commands:
# SQL> SELECT @@version;
# SQL> EXEC sp_databases;
# SQL> EXEC sp_helplogins;
# SQL> enable_xp_cmdshell
# SQL> xp_cmdshell whoami
```

### 5. NFS (Network File System - Port 2049)
```bash
# Discover exported NFS shares
showmount -e $TARGET

# Mount exported share to local system
sudo mkdir -p /mnt/target_nfs
sudo mount -t nfs $TARGET:/exported_folder /mnt/target_nfs -o nolock

# Inspect permissions and Root Squashing status
ls -la /mnt/target_nfs
```

### 6. FTP (Port 21)
```bash
# Test anonymous login
ftp $TARGET
# Username: anonymous | Password: anonymous

# Nmap FTP vulnerability & anonymous check scripts
nmap -p 21 --script ftp-anon,ftp-syst,ftp-bounce $TARGET -Pn
```

---

## 🐍 4. Pivot-Side Lightweight Port Scanners

When you have a low-privilege shell on a pivot box and need to scan hosts on an internal subnet without uploading heavy binaries:

### Lightweight Python Socket Scanner (Runs on Linux/Windows with Python installed):
```python
import socket
import sys

target = sys.argv[1]
ports = [21, 22, 23, 25, 53, 80, 88, 110, 135, 139, 143, 389, 443, 445, 1433, 3306, 3389, 5985, 8080]

print(f"Scanning {target}...")
for port in ports:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(0.5)
    result = s.connect_ex((target, port))
    if result == 0:
        print(f"[+] Port {port} is OPEN")
    s.close()
```

### Native PowerShell TCP Port Scanner:
```powershell
$target = "192.168.200.15"
$ports = 21,22,80,88,135,139,389,443,445,1433,3389,5985,8080
foreach ($p in $ports) {
    $t = New-Object System.Net.Sockets.TcpClient
    $c = $t.BeginConnect($target, $p, $null, $null)
    $w = $c.AsyncWaitHandle.WaitOne(400, $false)
    if ($t.Connected) {
        Write-Host "[+] Port $p is OPEN" -ForegroundColor Green
        $t.EndConnect($c)
    }
    $t.Close()
}
```
