# Nmap Complete Notes & Cheatsheet

> **Beginner to Advanced | Penetration Testing Reference**  
> By [SigintRecon](https://github.com/sigintrecon)

---

> **LEGAL WARNING:** Only use Nmap on systems you **own** or have **written permission** to test.  
> Unauthorized scanning is illegal under **Pakistan's PECA 2016** and cybercrime laws worldwide.

---

## Table of Contents

| # | Topic |
|---|-------|
| 1 | [What is Nmap?](#1-what-is-nmap) |
| 2 | [Professional Recon Workflow](#2-professional-recon-workflow) |
| 3 | [Host Discovery](#3-host-discovery) |
| 4 | [Port Scan Types](#4-port-scan-types) |
| 5 | [UDP Scanning](#5-udp-scanning) |
| 6 | [Service & Version Detection](#6-service--version-detection) |
| 7 | [NSE Scripts](#7-nse-scripts-nmap-scripting-engine) |
| 8 | [Firewall Evasion](#8-firewall-evasion-techniques) |
| 9 | [Port Specification](#9-port-specification) |
| 10 | [Output Formats](#10-output-formats) |
| 11 | [ICMP & TTL Concepts](#11-icmp--ttl-concepts) |
| 12 | [Subnetting Quick Reference](#12-subnetting-quick-reference) |
| 13 | [Quick Reference Cheatsheet](#13-quick-reference-cheatsheet) |
| 14 | [Real-World Pentest Examples](#14-real-world-pentest-examples) |
| 15 | [Legal & Ethical Reminder](#15-legal--ethical-reminder) |

---

## 1. What is Nmap?

**Nmap (Network Mapper)** is a free, open-source tool used for network discovery and security auditing.  
It sends specially crafted packets to target hosts and analyzes responses to discover:

| Feature | Description |
|---------|-------------|
| Host Discovery | Find which devices are alive on a network |
| Port Scanning | Detect open, closed, and filtered ports |
| Service Detection | Identify software and version on each port |
| OS Detection | Guess the operating system of the target |
| NSE Scripts | Run vulnerability checks and automation |
| Output Formats | Save results in text, XML, grepable formats |

---

## 2. Professional Recon Workflow

Always follow this order in real pentests and CTFs. Each step builds on the previous one.

```
Step 1:  nmap -sn 192.168.1.0/24          → Who is alive on the network?
Step 2:  nmap -sA 192.168.1.10             → Does the target have a firewall?
Step 3:  nmap -sS 192.168.1.10             → Which ports are open?
Step 4:  nmap -sV 192.168.1.10             → What services/versions are running?
Step 5:  nmap -O 192.168.1.10              → What OS is the target running?
Step 6:  nmap --script=vuln 192.168.1.10   → Any known vulnerabilities?
```

> **PRO TIP:** In CTFs and labs, you can combine steps 3–6 into one command:
> ```bash
> nmap -A -sV --script=vuln <target>
> ```
> In real pentests, go step-by-step to avoid detection.

---

## 3. Host Discovery

### 3.1 Ping Sweep

Find which hosts are alive on a network without scanning ports.  
Nmap sends **ICMP Echo Requests** to each IP in the range.

```bash
nmap -sn 192.168.1.0/24           # Scan entire /24 network
nmap -sn 192.168.1.1-50           # Scan IP range
nmap -sn 192.168.1.1,5,10         # Scan specific IPs
nmap -sn 192.168.1.0/24 -oG alive_hosts.txt   # Save results
```

**How it works:**
- On **local network**: Nmap uses ARP requests (more reliable, never blocked)
- On **remote network**: Nmap uses ICMP Echo + TCP SYN to port 443 + TCP ACK to port 80
- If ICMP is blocked: Use `-Pn` to skip ping and scan anyway

> **PRO TIP:** Use `fping` for faster ping sweeps on large networks:
> ```bash
> fping -a -g 192.168.1.0/24 2>/dev/null
> ```

---

### 3.2 ARP Scan

ARP resolves IP addresses to MAC addresses. ARP scanning is the **most reliable** host discovery method on local networks because firewalls almost never block ARP.

```bash
nmap -sn -PR 192.168.1.0/24                     # ARP only scan via nmap
sudo arp-scan 192.168.1.0/24                    # Dedicated ARP scanner (shows MAC vendor)
sudo netdiscover -r 192.168.1.0/24              # Interactive ARP discovery
sudo netdiscover -p                             # Passive mode (completely silent!)
```

| Tool | Best Use | Notes |
|------|----------|-------|
| `nmap -sn` | General purpose | Combines ICMP + ARP automatically |
| `arp-scan` | Local network | Shows MAC vendor info |
| `netdiscover` | Local network | Passive mode available |

> **PRO TIP:** ARP scan can reveal hidden devices that block ICMP ping. Always run both ping sweep and ARP scan for complete results.

---

## 4. Port Scan Types

### Understanding TCP Flags

Before learning scan types, understand the 6 TCP flags:

| Flag | Full Name | Meaning |
|------|-----------|---------|
| SYN | Synchronize | Start a new connection |
| ACK | Acknowledge | Confirm packet received |
| FIN | Finish | End connection gracefully |
| RST | Reset | Forcefully terminate connection |
| PSH | Push | Send data immediately, skip buffer |
| URG | Urgent | This data is high priority |

---

### 4.1 SYN Scan (Half-Open Scan) — `-sS`

The **most popular and default** scan. Sends a SYN packet and waits for a response.  
Never completes the full TCP handshake, so it leaves **no log entries** on most systems.

```
How it works:
  We send:     SYN  ──────────────► Target
  Port OPEN:   SYN+ACK ◄────────── Target  →  We send RST (abort)
  Port CLOSED: RST ◄─────────────  Target
  Filtered:    No response (firewall dropped it)
```

```bash
nmap -sS 192.168.1.10              # Basic SYN scan (requires root/sudo)
nmap -sS -p 80,443 192.168.1.10    # Specific ports
nmap -sS -p 1-1000 192.168.1.10    # Port range
nmap -sS -p- 192.168.1.10          # All 65535 ports
```

- Requires `root`/`sudo` privileges
- **Stealth Level: HIGH** — does not complete TCP handshake
- Most reliable scan — detects open, closed, and filtered ports

> **PRO TIP:** Always use `-sS` as your primary scan. Add `-p-` to scan all 65535 ports on important targets — the default scan only covers the top 1000.

---

### 4.2 TCP Connect Scan — `-sT`

Completes the full **TCP three-way handshake**. Less stealthy but works without root privileges.

```bash
nmap -sT 192.168.1.10
nmap -sT -p 80,443,8080 192.168.1.10
```

- Does **NOT** require root privileges
- **Stealth Level: LOW** — leaves connection logs on target
- Use when you cannot run nmap as root

---

### 4.3 ACK Scan — `-sA`

Does **NOT** find open ports. Its only purpose is to **detect firewalls** and determine if ports are filtered or unfiltered.

```
How it works:
  We send:         ACK  ─────────────► Target
  No firewall:     RST ◄─────────────  Target  →  UNFILTERED
  Firewall active: No response          →  FILTERED (firewall blocking)
```

```bash
nmap -sA 192.168.1.10              # Firewall detection
nmap -sA -p 80,443 192.168.1.10    # Check specific ports for firewall
```

- RST received = No firewall — direct communication possible
- No response = Firewall present — packets are being dropped
- **Stealth Level: MEDIUM**

> **PRO TIP:** Run ACK scan before your main scan to understand firewall behavior. If heavily filtered, switch to evasion techniques.

---

### 4.4 XMAS Scan — `-sX`

Sets **FIN, PSH, and URG** flags simultaneously. Named "XMAS" because all flags light up like a Christmas tree. Used to bypass stateless firewalls.

```
How it works:
  We send:            FIN+PSH+URG ──────► Target
  Port OPEN:          No response         →  open|filtered
  Port CLOSED:        RST ◄────────────── Target
  Stateful firewall:  No response         →  All blocked
```

```bash
nmap -sX 192.168.1.10
nmap -sX -p 1-1000 192.168.1.10
```

- Bypasses stateless firewalls (rules only check SYN packets)
- Does **NOT work on Windows** (Windows sends RST for all ports)
- Works correctly on **Linux targets**
- **Stealth Level: STEALTHY**

> **PRO TIP:** If `-sS` shows everything as filtered, try `-sX` and `-sN`. If those work, the target has a stateless firewall.

---

### 4.5 NULL Scan — `-sN`

Sends a TCP packet with **NO flags** set. The most stealthy scan because IDS/firewall rules rarely account for packets with zero flags.

```
How it works:
  We send:    [empty packet, no flags] ──► Target
  Port OPEN:  No response               →  open|filtered
  Port CLOSED: RST ◄──────────────────── Target
```

```bash
nmap -sN 192.168.1.10
nmap -sN -p 1-1000 192.168.1.10
```

- **Stealth Level: VERY HIGH** — hardest to detect
- Does **NOT work on Windows**
- Best for Linux/Unix targets
- Cannot differentiate between open and filtered ports

> **PRO TIP:** NULL scan is your "ghost mode." Use it when maximum stealth is required against Linux targets.

---

### 4.6 FIN Scan — `-sF`

Sends only the **FIN** flag. Similar behavior to XMAS and NULL scans but with a single flag.

```bash
nmap -sF 192.168.1.10
nmap -sF -p 22,80,443 192.168.1.10
```

- Same limitations as XMAS and NULL — does not work on Windows
- **Stealth Level: STEALTHY**

---

### 4.7 Scan Types Comparison Table

| Scan | Flag(s) | Finds Open Ports? | Stealth | Works on Windows? |
|------|---------|------------------|---------|------------------|
| `-sS` SYN | SYN | ✅ Yes | High | ✅ Yes |
| `-sT` Connect | SYN+ACK+RST | ✅ Yes | Low | ✅ Yes |
| `-sA` ACK | ACK | ❌ No (firewall only) | Medium | ✅ Yes |
| `-sX` XMAS | FIN+PSH+URG | ⚠️ Sometimes | Stealthy | ❌ No |
| `-sN` NULL | None | ⚠️ Sometimes | Very High | ❌ No |
| `-sF` FIN | FIN | ⚠️ Sometimes | Stealthy | ❌ No |

---

## 5. UDP Scanning

Most pentesters ignore UDP — which is exactly why it is often the **most interesting attack surface**.  
Services like DNS, SNMP, DHCP, and TFTP run on UDP and are frequently misconfigured.

```
How it works:
  We send:         UDP packet ──────────────► Target
  Port OPEN:       No response (UDP has no handshake)
  Port CLOSED:     ICMP Type 3 "Port Unreachable" ◄── Target
  Port FILTERED:   No response (firewall dropped it)

  NOTE: open and filtered look the same → nmap marks them open|filtered
```

```bash
sudo nmap -sU 192.168.1.10                      # Default UDP scan (top 1000)
sudo nmap -sU -p 53,161,67,69 192.168.1.10      # Common UDP ports
sudo nmap -sU --top-ports 200 192.168.1.10       # Top 200 UDP ports
sudo nmap -sU -sS 192.168.1.10                   # UDP + SYN scan combined
sudo nmap -sU -sV 192.168.1.10                   # UDP + version detection
sudo nmap -sU --open 192.168.1.10                # Show only open ports
```

### Important UDP Ports to Always Check

| Port | Service | Why It Matters |
|------|---------|----------------|
| 53/UDP | DNS | Zone transfer, cache poisoning, subdomain enum |
| 67/UDP | DHCP | DHCP starvation, rogue DHCP server attacks |
| 69/UDP | TFTP | Often no authentication — file read/write |
| 111/UDP | RPC | Remote procedure call — can expose services |
| 123/UDP | NTP | Amplification attacks, time manipulation |
| 161/UDP | SNMP | Community strings leak network info (v1/v2c) |
| 162/UDP | SNMP Trap | Network device alerts |
| 500/UDP | IKE/VPN | VPN fingerprinting |
| 1434/UDP | MSSQL | SQL Server browser service |

> **WARNING:** UDP scanning is SLOW. Scanning all 65535 UDP ports can take hours. Always target specific high-value ports first.

> **PRO TIP:** SNMP port 161 is gold. Run:
> ```bash
> nmap -sU -p 161 --script=snmp-info,snmp-sysdescr <target>
> ```
> Misconfigured SNMP reveals OS, running processes, network interfaces, and more.

---

## 6. Service & Version Detection

Knowing port 80 is open is not enough. You need to know **WHAT** is running on port 80.  
Is it Apache 2.4.49? That has a critical CVE. Version detection turns a port scan into a vulnerability assessment.

```bash
nmap -sV 192.168.1.10                       # Service/version detection
nmap -sV --version-intensity 9 192.168.1.10  # Maximum probing (thorough)
nmap -sV --version-intensity 0 192.168.1.10  # Light probing (fast)
nmap -sV --version-light 192.168.1.10        # Intensity 2 (quick)
nmap -sV --version-all 192.168.1.10          # Try all probes
```

### 6.1 OS Detection

Nmap analyzes TCP/IP stack behavior to guess the operating system.

```bash
nmap -O 192.168.1.10                     # OS detection
nmap -O --osscan-guess 192.168.1.10      # Aggressively guess OS
nmap -O --osscan-limit 192.168.1.10      # Skip if no open+closed ports
nmap -A 192.168.1.10                     # All: OS + version + scripts + traceroute
```

### 6.2 Aggressive Scan — `-A`

The most complete single scan. Combines OS detection, version detection, default scripts, and traceroute.

```bash
nmap -A 192.168.1.10        # Full aggressive scan
nmap -A -p- 192.168.1.10    # All ports + aggressive
nmap -A -T4 192.168.1.10    # Fast + aggressive
```

> **WARNING:** Aggressive scan is NOISY. Use freely in labs and CTFs. In real authorized pentests, use it only when noise is acceptable.

---

## 7. NSE Scripts (Nmap Scripting Engine)

NSE is where Nmap transforms from a port scanner into a **vulnerability scanner**.  
Scripts are written in Lua and can perform everything from banner grabbing to full exploitation.

### Script Categories

```bash
--script=default     # Safe, informational scripts (same as -sC)
--script=vuln        # Check for known CVEs and vulnerabilities
--script=auth        # Test for default/empty credentials
--script=discovery   # Extra information gathering
--script=brute       # Password brute force attacks
--script=exploit     # Active exploitation scripts (use carefully!)
--script=safe        # Only completely safe scripts
--script=intrusive   # May crash or affect target services
```

---

### 7.1 Essential Scripts by Service

#### SMB Scripts (Windows Targets)

```bash
nmap --script=smb-vuln* -p 445 192.168.1.10          # All SMB vulns
nmap --script=smb-vuln-ms17-010 -p 445 192.168.1.10  # EternalBlue check
nmap --script=smb-vuln-ms08-067 -p 445 192.168.1.10  # MS08-067
nmap --script=smb-enum-shares -p 445 192.168.1.10    # List shares
nmap --script=smb-enum-users -p 445 192.168.1.10     # List users
nmap --script=smb-os-discovery -p 445 192.168.1.10   # OS via SMB
nmap --script=smb-security-mode -p 445 192.168.1.10  # SMB signing
```

#### HTTP Scripts

```bash
nmap --script=http-title -p 80,443 192.168.1.10      # Page title
nmap --script=http-enum -p 80,443 192.168.1.10       # Dir/file enumeration
nmap --script=http-headers -p 80 192.168.1.10        # Response headers
nmap --script=http-methods -p 80 192.168.1.10        # Allowed HTTP methods
nmap --script=http-shellshock -p 80 192.168.1.10     # Shellshock CVE
nmap --script=http-sql-injection -p 80 192.168.1.10  # Basic SQLi check
nmap --script=http-auth-finder -p 80 192.168.1.10    # Auth type
```

#### FTP, SSH, and Database Scripts

```bash
# FTP
nmap --script=ftp-anon -p 21 192.168.1.10            # Anonymous login?
nmap --script=ftp-brute -p 21 192.168.1.10           # Brute force
nmap --script=ftp-bounce -p 21 192.168.1.10          # FTP bounce attack

# SSH
nmap --script=ssh-auth-methods -p 22 192.168.1.10    # Auth methods allowed
nmap --script=ssh-brute -p 22 192.168.1.10           # Brute force SSH
nmap --script=ssh-hostkey -p 22 192.168.1.10         # Get host key

# Databases
nmap --script=mysql-empty-password -p 3306 192.168.1.10
nmap --script=mysql-info -p 3306 192.168.1.10
nmap --script=ms-sql-info -p 1433 192.168.1.10

# SNMP
nmap -sU --script=snmp-info -p 161 192.168.1.10
nmap -sU --script=snmp-brute -p 161 192.168.1.10    # Brute community string
```

---

### 7.2 Banner Grabbing

Banner grabbing extracts version/identity information directly from a service.  
This tells you exactly what software is running so you can search for CVEs.

```bash
# Method 1: Nmap script
nmap --script=banner -p 21,22,80,8080 192.168.1.10

# Method 2: Netcat (manual)
nc 192.168.1.10 21                               # Connect to FTP
nc 192.168.1.10 22                               # Connect to SSH
echo 'HEAD / HTTP/1.0' | nc 192.168.1.10 80     # HTTP banner

# Method 3: Telnet
telnet 192.168.1.10 80
telnet 192.168.1.10 21

# Method 4: curl
curl -I http://192.168.1.10                      # HTTP headers
curl -sv http://192.168.1.10 2>&1 | head -20
```

> **PRO TIP:** After grabbing a banner, search:  
> `<service> <version> exploit` or `<service> <version> CVE`  
> on [exploit-db.com](https://exploit-db.com), [nvd.nist.gov](https://nvd.nist.gov), and Google.

---

## 8. Firewall Evasion Techniques

When standard scans are blocked, these techniques help bypass firewalls and IDS/IPS systems.

### 8.1 `-f` — Packet Fragmentation

Breaks packets into tiny fragments (8 bytes each). Stateless firewalls inspect complete packets — fragments confuse them.

```bash
nmap -f 192.168.1.10            # Fragment packets (8 bytes)
nmap -ff 192.168.1.10           # 16-byte fragments
nmap --mtu 24 192.168.1.10      # Custom MTU (must be multiple of 8)
```

- Stateless firewalls fail to reassemble fragments for inspection
- Modern **stateful firewalls DO reassemble** — so this may not always work

---

### 8.2 `-D` — Decoy Scan

Makes scan appear to come from **multiple IP addresses** simultaneously.  
Target logs show many attackers — impossible to identify the real one.

```bash
nmap -D RND:10 192.168.1.10                       # 10 random decoy IPs
nmap -D RND:5 192.168.1.10                        # 5 random decoys
nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.10        # Specific decoys (ME = your real IP)
nmap -D 10.0.0.1,10.0.0.2 192.168.1.10           # Decoys only
```

- `RND:10` generates 10 random fake source IPs
- `ME` places your real IP among the decoys at that position
- Target sees traffic from 10+ IPs — logs are useless for attribution

> **PRO TIP:** Use `-D` with real IPs from the same subnet as the target for maximum realism. Random IPs may trigger reverse-path filtering.

---

### 8.3 `--source-port` — Source Port Spoofing

Many firewalls allow traffic from trusted ports (DNS:53, HTTPS:443) by default.  
Spoofing the source port makes your scan look like legitimate traffic.

```bash
nmap --source-port 53 192.168.1.10      # Appear as DNS traffic
nmap --source-port 443 192.168.1.10     # Appear as HTTPS traffic
nmap --source-port 80 192.168.1.10      # Appear as HTTP traffic
nmap -g 53 192.168.1.10                 # -g is shorthand for --source-port
```

---

### 8.4 `--data-length` — Packet Padding

Adds random data to packets to change their size. IDS systems detect Nmap by its signature (packet size, timing). Padding breaks the signature.

```bash
nmap --data-length 25 192.168.1.10      # Add 25 random bytes
nmap --data-length 50 192.168.1.10      # Add 50 random bytes
```

---

### 8.5 `-T` — Timing Control

Controls how fast Nmap sends packets. Slower scans evade IDS rate-based detection.

| Flag | Name | Speed | Best Use |
|------|------|-------|----------|
| `-T0` | Paranoid | 1 packet / 5 minutes | Maximum stealth, very slow |
| `-T1` | Sneaky | 1 packet / 15 seconds | IDS evasion, slow |
| `-T2` | Polite | 0.4s between packets | Reduce network load |
| `-T3` | Normal | Default | Balanced (default) |
| `-T4` | Aggressive | Fast | CTFs and labs |
| `-T5` | Insane | Very fast | Local network only, very noisy |

> **PRO TIP:** For real authorized pentests: use `-T2` or `-T1`. For CTFs and labs: use `-T4`. Never use `-T5` on networks you do not fully own.

---

### 8.6 `--randomize-hosts`

Scans hosts in random order instead of sequential. Sequential scanning (.1, .2, .3...) is a clear IDS trigger.

```bash
nmap --randomize-hosts 192.168.1.0/24
nmap --randomize-hosts -sS 192.168.1.0/24
```

---

### 8.7 `-Pn` — Skip Host Discovery

Treats all hosts as alive and skips the ping phase. Use when the target blocks ICMP but you know it is up.

```bash
nmap -Pn 192.168.1.10           # Skip ping, assume host is up
nmap -Pn -sS 192.168.1.10       # No ping + SYN scan
nmap -Pn -A 192.168.1.10        # No ping + aggressive
```

---

### 8.8 Combined Evasion — Real Pentest Command

```bash
# Maximum stealth scan
nmap -f -D RND:10 --source-port 53 -T1 --data-length 25 --randomize-hosts 192.168.1.10

# Stealth + version detection
nmap -sS -f --source-port 443 -T2 -sV 192.168.1.10

# Bypass ICMP block + stealth scan
nmap -Pn -f -D RND:5 -sS 192.168.1.10
```

---

## 9. Port Specification

```bash
nmap -p 80 192.168.1.10                  # Single port
nmap -p 80,443,8080 192.168.1.10         # Multiple ports
nmap -p 1-1000 192.168.1.10              # Port range
nmap -p- 192.168.1.10                    # ALL 65535 ports
nmap -p U:53,T:80,443 192.168.1.10       # UDP port 53, TCP 80 and 443
nmap --top-ports 100 192.168.1.10        # Top 100 most common ports
nmap --top-ports 1000 192.168.1.10       # Top 1000 ports (default)
nmap -F 192.168.1.10                     # Fast: top 100 ports only
nmap -p http,https 192.168.1.10          # Use service names
nmap --open 192.168.1.10                 # Show only open ports
```

> **PRO TIP:** Always run `-p-` on your primary target in CTFs. Services on unusual ports (like SSH on 2222 or HTTP on 8888) are very common in challenges.

---

## 10. Output Formats

Always save your scan results. You will need them for reports, later analysis, and to avoid rescanning.

```bash
nmap -oN output.txt 192.168.1.10         # Normal (human readable)
nmap -oX output.xml 192.168.1.10         # XML (tool-parseable)
nmap -oG output.gnmap 192.168.1.10       # Grepable (bash scripting)
nmap -oA output 192.168.1.10             # All three formats at once
```

**Working with grepable output:**
```bash
grep 'open' output.gnmap                           # Find open ports
grep '80/open' output.gnmap | awk '{print $2}'     # IPs with port 80 open
```

> **PRO TIP:** Always use `-oA` in real pentests. It saves all three formats simultaneously.

---

## 11. ICMP & TTL Concepts

### 11.1 ICMP Message Types

| Type | Name | Pentesting Use |
|------|------|----------------|
| Type 0 | Echo Reply | Host is alive (response to ping) |
| Type 3 | Destination Unreachable | UDP closed ports — ICMP 3 code 3 |
| Type 8 | Echo Request | Ping — host discovery |
| Type 11 | Time Exceeded | Traceroute — TTL expired |

---

### 11.2 TTL — Time To Live

TTL is a counter in every packet. Each router decrements it by 1. When TTL reaches 0, the router drops the packet and sends ICMP Type 11 back. **Traceroute exploits this to map network paths.**

```bash
ping -c 3 192.168.1.10               # Basic ICMP test
traceroute 192.168.1.10              # Map network path (Linux)
tracert 192.168.1.10                 # Map network path (Windows)
nmap --traceroute 192.168.1.10       # Traceroute via nmap
```

**Default TTL Values by OS:**

| Default TTL | Operating System |
|-------------|-----------------|
| 64 | Linux / macOS / Android |
| 128 | Windows |
| 255 | Cisco / Network devices |

> **PRO TIP:** Check TTL value in ping response to guess the OS without a full scan. TTL near 64 = likely Linux, near 128 = likely Windows.

---

## 12. Subnetting Quick Reference

CIDR notation (/24, /16 etc.) tells you how many IPs are in a network range.

| CIDR | Subnet Mask | Total IPs | Usable Hosts | Example |
|------|-------------|-----------|--------------|---------|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 | 10.0.0.0/8 |
| /16 | 255.255.0.0 | 65,536 | 65,534 | 192.168.0.0/16 |
| /24 | 255.255.255.0 | 256 | 254 | 192.168.1.0/24 |
| /25 | 255.255.255.128 | 128 | 126 | 192.168.1.0/25 |
| /27 | 255.255.255.224 | 32 | 30 | 192.168.1.0/27 |
| /28 | 255.255.255.240 | 16 | 14 | 192.168.1.0/28 |
| /30 | 255.255.255.252 | 4 | 2 | 192.168.1.0/30 |
| /32 | 255.255.255.255 | 1 | 1 (host only) | 192.168.1.10/32 |

> **PRO TIP:** Use `ipcalc 192.168.1.0/27` in Kali for instant subnet calculations.  
> Formula: `Usable hosts = 2^(32-prefix) - 2`

---

## 13. Quick Reference Cheatsheet

### Host Discovery
```bash
nmap -sn 192.168.1.0/24                     # Ping sweep
nmap -sn -PR 192.168.1.0/24                 # ARP scan only
nmap -Pn 192.168.1.10                       # Skip ping, assume up
sudo arp-scan 192.168.1.0/24                # ARP scan (arp-scan tool)
sudo netdiscover -r 192.168.1.0/24          # Netdiscover
sudo netdiscover -p                         # Passive ARP
```

### Port Scanning
```bash
nmap -sS 192.168.1.10                       # SYN scan (stealth)
nmap -sT 192.168.1.10                       # TCP connect
nmap -sA 192.168.1.10                       # ACK (firewall detect)
nmap -sX 192.168.1.10                       # XMAS scan
nmap -sN 192.168.1.10                       # NULL scan
nmap -sF 192.168.1.10                       # FIN scan
sudo nmap -sU 192.168.1.10                  # UDP scan
nmap -sS -sU 192.168.1.10                   # TCP + UDP combined
nmap -p- 192.168.1.10                       # All 65535 ports
nmap -F 192.168.1.10                        # Fast (top 100)
nmap --open 192.168.1.10                    # Open ports only
```

### Version & OS Detection
```bash
nmap -sV 192.168.1.10                       # Service versions
nmap -O 192.168.1.10                        # OS detection
nmap -A 192.168.1.10                        # All (OS+version+scripts)
nmap -sV --version-intensity 9 192.168.1.10 # Max version probe
```

### NSE Scripts
```bash
nmap -sC 192.168.1.10                       # Default scripts
nmap --script=vuln 192.168.1.10             # Vulnerability scan
nmap --script=banner 192.168.1.10           # Banner grabbing
nmap --script=smb-vuln* -p 445 192.168.1.10         # All SMB vulns
nmap --script=smb-vuln-ms17-010 -p 445 192.168.1.10 # EternalBlue
nmap --script=http-enum -p 80 192.168.1.10           # HTTP enumeration
nmap --script=ftp-anon -p 21 192.168.1.10            # FTP anonymous login
nmap -sU --script=snmp-info -p 161 192.168.1.10      # SNMP info
```

### Firewall Evasion
```bash
nmap -f 192.168.1.10                        # Fragment packets
nmap -D RND:10 192.168.1.10                 # Decoy scan
nmap --source-port 53 192.168.1.10          # Source port spoof
nmap --data-length 25 192.168.1.10          # Pad packets
nmap -T0 192.168.1.10                       # Paranoid (slowest)
nmap -T1 192.168.1.10                       # Sneaky
nmap --randomize-hosts 192.168.1.0/24       # Random order
nmap -f -D RND:10 --source-port 53 -T1 192.168.1.10  # Combined
```

### Output
```bash
nmap -oN out.txt 192.168.1.10               # Normal text
nmap -oX out.xml 192.168.1.10               # XML
nmap -oG out.gnmap 192.168.1.10             # Grepable
nmap -oA out 192.168.1.10                   # All formats
nmap -v 192.168.1.10                        # Verbose output
nmap -vv 192.168.1.10                       # Very verbose
```

---

## 14. Real-World Pentest Examples

### CTF / Lab — Full Target Scan

```bash
# Step 1: Discover network
nmap -sn 192.168.1.0/24

# Step 2: Full port scan on target
nmap -sS -p- -T4 192.168.1.10 -oN ports.txt

# Step 3: Version + OS on open ports
nmap -sV -O -p 22,80,445 192.168.1.10

# Step 4: Vulnerability scan
nmap --script=vuln -p 22,80,445 192.168.1.10

# Or all-in-one:
nmap -A -p- --script=vuln 192.168.1.10 -oA full_scan
```

---

### Windows Target (SMB Focus)

```bash
nmap -sS -p 135,139,445,3389 192.168.1.10
nmap --script=smb-vuln*,smb-enum* -p 445 192.168.1.10
nmap --script=smb-os-discovery -p 445 192.168.1.10
```

---

### Web Server Target

```bash
nmap -sV -p 80,443,8080,8443 192.168.1.10
nmap --script=http-title,http-enum,http-headers -p 80,443 192.168.1.10
nmap --script=http-shellshock,http-sql-injection -p 80 192.168.1.10
```

---

### Stealth Recon (Real Authorized Pentest)

```bash
# Phase 1: Silent host discovery
sudo netdiscover -p   # passive — no packets sent

# Phase 2: Slow stealthy scan
nmap -sS -T1 -f --source-port 443 -D RND:5 192.168.1.10 -oG stealth.gnmap

# Phase 3: Targeted version detection on discovered ports
nmap -sV --version-intensity 5 -p <open_ports> 192.168.1.10 -oN versions.txt
```

---

## 15. Legal & Ethical Reminder

> **NEVER** scan systems you do not own or do not have explicit **written permission** to test.  
> In Pakistan, unauthorized network scanning violates the **Prevention of Electronic Crimes Act (PECA) 2016** and can result in serious legal consequences.

**Authorized use cases:**
- Your own home lab (VirtualBox, VMware)
- [TryHackMe](https://tryhackme.com), [HackTheBox](https://hackthebox.com), [VulnHub](https://vulnhub.com) — these provide explicit authorization
- Bug bounty programs — only within defined scope
- Penetration testing with a **signed contract** from the client

> **PRO TIP:** Always get written authorization before any real-world scan. A verbal "go ahead" is not enough in professional pentesting.

---

## Recommended Learning Resources

| Resource | Type | Link |
|----------|------|------|
| TryHackMe | Interactive Labs | [tryhackme.com](https://tryhackme.com) |
| HackTheBox | CTF / Labs | [hackthebox.com](https://hackthebox.com) |
| Nmap Official Docs | Documentation | [nmap.org/book](https://nmap.org/book/man.html) |
| Exploit-DB | CVE Search | [exploit-db.com](https://exploit-db.com) |
| NVD NIST | Vulnerability DB | [nvd.nist.gov](https://nvd.nist.gov) |

---

## 🔗 Connect

> Created by **SigintRecon** — Aspiring Red Team Operator  
> GitHub: [github.com/sigintrecon](https://github.com/sigintrecon)  
> TryHackMe: [tryhackme.com/p/sigintrecon](https://tryhackme.com/p/sigintrecon)

---

*For authorized and educational use only.*
