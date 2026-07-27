# Computer Networks — Full Explanation with Linux Exercises

## 1. What Is a Computer Network?

A computer network is two or more devices connected so they can share data and resources (files, printers, internet access). Networks are classified by scale:

- **PAN** (Personal Area Network) — Bluetooth, a few meters
- **LAN** (Local Area Network) — your home/office, switches + Wi-Fi
- **MAN** (Metropolitan Area Network) — a city
- **WAN** (Wide Area Network) — the Internet is the biggest WAN
- **VPN** (Virtual Private Network) — encrypted tunnel over a public network

---

## 2. The Two Reference Models

### OSI Model (7 layers — conceptual)

| # | Layer | What it does | Examples |
|---|-------|-------------|----------|
| 7 | Application | User-facing protocols | HTTP, DNS, SSH, FTP |
| 6 | Presentation | Encoding, encryption | TLS/SSL, JPEG, ASCII |
| 5 | Session | Open/manage connections | Sessions, sockets |
| 4 | Transport | Reliable delivery, ports | TCP, UDP |
| 3 | Network | Routing between networks | IP, ICMP, routers |
| 2 | Data Link | Frames, MAC addresses | Ethernet, Wi-Fi, switches |
| 1 | Physical | Bits on the wire | Cables, radio, hubs |

### TCP/IP Model (4 layers — what actually runs the Internet)

Application → Transport → Internet → Link.

**Mnemonic for OSI (bottom-up):** *"Please Do Not Throw Sausage Pizza Away"* (Physical, Data Link, Network, Transport, Session, Presentation, Application).

---

## 3. Key Concepts

### IP Addresses
- **IPv4**: 32 bits, e.g. `192.168.1.10`. Divided into network + host parts via a **subnet mask** (`255.255.255.0` = `/24`).
- **Private ranges** (not routable on the Internet): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
- **IPv6**: 128 bits, e.g. `2001:db8::1`.
- **Loopback**: `127.0.0.1` — your machine talking to itself.

### MAC Addresses
48-bit hardware address of a network card (`aa:bb:cc:11:22:33`). Used on Layer 2 within a LAN.

### Ports
Numbers 0–65535 that identify *which service* on a machine:
- Well-known: 22 (SSH), 53 (DNS), 80 (HTTP), 443 (HTTPS)
- Registered: 1024–49151 · Dynamic/ephemeral: 49152+

### TCP vs UDP

| TCP | UDP |
|-----|-----|
| Connection-oriented | Connectionless |
| Reliable, ordered, retransmits | Fast, no guarantees |
| Web, SSH, email | DNS, streaming, VoIP, games |

**TCP 3-way handshake:** SYN → SYN-ACK → ACK.

### DNS
Translates names (`google.com`) to IPs. Record types: `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias), `MX` (mail), `NS`, `TXT`.

### DHCP
Automatically assigns IP, gateway, and DNS servers when you join a network.

### NAT
Your router translates many private IPs into one public IP — why your whole house shares one Internet address.

### Routing & the Default Gateway
Packets destined outside your subnet go to the **default gateway** (your router), which forwards them hop by hop.

---

## 4. How Devices Fit Together

- **Hub** (dead): repeats to everyone — Layer 1
- **Switch**: forwards frames by MAC address — Layer 2
- **Router**: forwards packets between networks by IP — Layer 3
- **Access Point**: bridges Wi-Fi clients into the LAN
- **Firewall**: filters traffic by rules (port, IP, state)

---

## 5. Linux Exercises (Hands-On)

### Exercise 1 — Discover your own network identity
```bash
ip addr show              # your interfaces and IPs (modern replacement for ifconfig)
ip link show              # Layer 2 info: MAC addresses, interface state
ip route show             # routing table — find your default gateway
cat /etc/resolv.conf      # which DNS servers you use
```
**Questions to answer:** What's your IP? Subnet? Gateway? MAC?

### Exercise 2 — Connectivity and path tracing
```bash
ping -c 4 8.8.8.8                  # is the Internet reachable by IP?
ping -c 4 google.com               # is DNS working too?
traceroute google.com              # each hop to the destination
                                   # (sudo apt install traceroute)
tracepath google.com               # no install needed on most systems
```
**Insight:** if `8.8.8.8` works but `google.com` doesn't → DNS problem, not connectivity.

### Exercise 3 — DNS deep dive
```bash
dig google.com                     # full DNS answer (apt install dnsutils)
dig @8.8.8.8 google.com            # query a specific DNS server
dig MX gmail.com                   # mail records
nslookup google.com
host google.com
```

### Exercise 4 — See who's listening and who's connected
```bash
ss -tulpn                          # all listening TCP/UDP ports + processes
ss -tn state established           # active connections
lsof -i :22                        # what process uses port 22
```

### Exercise 5 — Packet capture (the "aha" moment)
```bash
sudo tcpdump -i any -n icmp        # in one terminal
ping -c 3 8.8.8.8                  # in another — watch the packets appear
sudo tcpdump -i any -n port 53     # watch your DNS queries live
```
Install Wireshark for a GUI version: `sudo apt install wireshark`. Capture traffic and find a **TCP 3-way handshake** yourself.

### Exercise 6 — Scan a network
```bash
sudo apt install nmap
nmap -sn 192.168.1.0/24            # discover live hosts on your LAN
nmap localhost                     # which ports are open on your machine
nmap -sV localhost                 # with service version detection
```
⚠️ Only scan networks you own or have permission to test.

### Exercise 7 — Run a real client/server (no internet needed)
```bash
# Terminal 1 — start a web server on port 8000:
python3 -m http.server 8000

# Terminal 2 — talk to it:
curl -v localhost:8000             # see the full HTTP exchange
ss -tlnp | grep 8000               # confirm it's listening
```
Also try netcat chat:
```bash
nc -l 1234                         # terminal 1: listen
nc 127.0.0.1 1234                  # terminal 2: connect — type and chat
```

### Exercise 8 — Firewall basics (ufw)
```bash
sudo ufw status
sudo ufw allow 22/tcp              # allow SSH
sudo ufw enable
sudo ufw deny 8000                 # block your test server — verify with curl
```

### Exercise 9 — ARP (Layer 2 in action)
```bash
ip neigh show                      # ARP cache: IP → MAC mappings on your LAN
ping -c 1 <gateway-ip>             # then check the cache again
```

### Exercise 10 — Bring an interface down/up & set an IP manually
```bash
sudo ip link set eth0 down         # careful: kills connectivity on that NIC
sudo ip link set eth0 up
sudo ip addr add 192.168.1.50/24 dev eth0     # static IP (lab only!)
```

---

## 6. Useful Files to Know

| File | Purpose |
|------|---------|
| `/etc/hosts` | Local name resolution (checked before DNS) |
| `/etc/resolv.conf` | DNS servers |
| `/etc/network/interfaces` or Netplan (`/etc/netplan/*.yaml`) | Interface config (Debian/Ubuntu) |
| `/etc/services` | Port number ↔ service name mapping |

---

## 7. Suggested Learning Path

1. Weeks 1–2: models, IP/subnetting, exercises 1–3
2. Week 3: TCP/UDP, ports, exercises 4–7
3. Week 4: DNS, DHCP, NAT, firewalls, exercises 8–10
4. Week 5+: subnetting drills (e.g., split `192.168.1.0/24` into 4 subnets), Wireshark analysis, and set up two VMs talking to each other

Want me to generate a **subnetting practice worksheet**, a **cheat-sheet PDF of these commands**, or set up a **guided two-VM lab** (client + server with Apache/SSH) next?