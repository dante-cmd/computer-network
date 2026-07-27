# NAT — Full Explanation

## 1. What NAT Is and Why It Exists

**NAT (Network Address Translation)** rewrites IP addresses (and usually ports) in packets as they pass through a router. It exists for one main reason:

> **There aren't enough IPv4 addresses for everyone.**

Only ~4.3 billion IPv4 addresses exist, but there are far more devices than people. The solution, standardized in the 1990s:

1. Reserve **private ranges** (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) that anyone can reuse internally — these are *never routed on the Internet*
2. Let an entire household/company share **one public IP**, with the router translating between the two worlds

**Analogy:** an office building. Outside, the building has one street address. Inside, each employee has a desk number. The **receptionist (NAT)** receives all incoming mail, figures out which desk it goes to, and stamps outgoing mail with the building's address instead of desk numbers. People outside never see desk numbers; people inside never worry about street addresses.

```
   INSIDE (private)              ROUTER               OUTSIDE (public)
┌───────────────────┐         ┌─────────────┐
│ 192.168.1.10  💻  │ ──────► │     NAT     │ ──────►  Internet
│ 192.168.1.11  📱  │ ──────► │ 190.42.7.25 │         sees ONE address
│ 192.168.1.12  📺  │ ──────► │  (public)   │
└───────────────────┘         └─────────────┘
```

---

## 2. How It Actually Works: The Translation Table

The magic is a **table in the router's memory** that remembers every connection. This is what lets replies find their way back.

### Outbound packet (your laptop → Google)

Your laptop wants to reach `142.250.72.14:443` (Google HTTPS). The original packet:

```
SRC: 192.168.1.10:52311  →  DST: 142.250.72.14:443
```

That source IP is private — useless on the Internet. So the router **rewrites it**:

```
SRC: 190.42.7.25:40001  →  DST: 142.250.72.14:443
   └── public IP ──┘ └─ new port
```

And records the mapping in its **NAT table**:

| Internal (private) | Translated (public) | Destination | Protocol |
|--------------------|--------------------|-------------|----------|
| 192.168.1.10:52311 | 190.42.7.25:40001 | 142.250.72.14:443 | TCP |

### Inbound reply (Google → your laptop)

Google answers to `190.42.7.25:40001`. The router looks in its table:

> "Port 40001... that belongs to 192.168.1.10's connection on port 52311."

It rewrites the destination back and forwards it to your laptop. Google never knew your real address existed; your laptop never knew translation happened. **Both sides think they're talking directly.**

### Why the port gets rewritten too

What if your phone *and* laptop both connect to Google, both from local port 52311? After translation both would claim to be `190.42.7.25:52311` — collision! So the router assigns a unique **public-side port** per connection (40001, 40002...). The port becomes part of the mapping key.

---

## 3. The Types of NAT

### SNAT — Source NAT
Changes the **source** address of outgoing packets. This is the everyday "share one Internet connection" scenario. When the public address is dynamic or shared, the specific variant is called **MASQUERADE** (Linux term) — "whatever my current public IP is, use it."

### DNAT — Destination NAT
Changes the **destination** of incoming packets. This is **port forwarding**: "connections arriving at my public IP on port 8080 → send them to internal server 192.168.1.20:80." Used to expose a home server, game server, or camera to the Internet.

### PAT / NAT Overload (what your home router actually does)
"Port Address Translation" = many private IPs sharing one public IP by also translating **ports**. Technically this is what people mean 99% of the time when they say "NAT." Strictly, "one-to-one NAT" (one private ↔ one public IP, no port changes) also exists but is rare because it doesn't save addresses.

### Static vs Dynamic
- **Static NAT**: fixed 1:1 mapping (e.g., a server always maps to the same public IP)
- **Dynamic NAT**: pool of public IPs, assigned as needed
- **PAT/Overload**: many:1 via ports — the default everywhere

---

## 4. What NAT Breaks (The Downside)

NAT was a patch, and patches have side effects:

### Incoming connections fail by default
If someone on the Internet initiates a connection to `190.42.7.25:3000`, the router checks its table... finds nothing... **drops the packet**. NAT only knows how to route *replies* to connections *you started*.

This is:
- ✅ A **free security benefit** — your devices are hidden, unsolicited traffic dies at the router (why people say NAT acts like a basic firewall)
- ❌ A problem when you *want* incoming connections — the fix is **port forwarding** (manually adding a permanent table entry: "public port 3000 always → 192.168.1.10:3000")

### Protocols that embed IPs in their data
Old protocols (FTP active mode, SIP/VoIP, some games) write IP addresses *inside* the packet payload. NAT rewrites the header but not the payload → protocol confusion. Routers carry special **helper modules** (ALGs) to peek inside and fix those too.

### NAT traversal for P2P
Video calls, gaming, and BitTorrent need two NATed devices to connect *to each other* — both behind NAT, neither can receive incoming. Techniques:
- **UPnP/NAT-PMP**: app asks the router to open a port automatically
- **STUN**: discover what your public IP:port mapping looks like
- **TURN**: relay traffic through a third-party server when direct connection fails
- **Hole punching**: both sides send outbound simultaneously so each NAT opens an entry the other can use

### CGNAT — NAT at the ISP level
ISPs also ran out of addresses, so many now put *customers* behind a second layer of NAT (Carrier-Grade NAT), using the `100.64.0.0/10` range. You're behind "double NAT" — port forwarding becomes impossible without asking the ISP. Check: if your router's "WAN IP" is in `100.64.x.x` or a private range, you have CGNAT (common on mobile data and some Peruvian ISPs on cheap plans).

### NAT and IPv6
IPv6 has enough addresses for everything, so **NAT becomes unnecessary** — every device can have a real public address. (Firewalls still filter; NAT was never truly a security feature, just a side effect.)

---

## 5. NAT vs Firewall — Don't Confuse Them

| | NAT | Firewall |
|--|-----|----------|
| Job | **Rewrite** addresses/ports | **Allow or block** traffic by rules |
| Question it answers | "How do I translate this?" | "Should this pass at all?" |
| Security | Incidental (hides internals) | Intentional (policy) |

They usually live in the same box (your router does both), but they're separate functions. Linux demonstrates this beautifully — same kernel machinery (`netfilter`), different tables.

---

## 6. Linux Hands-On Exercises

### Exercise 1 — Prove you're behind NAT
```bash
ip addr show                    # private IP, e.g. 192.168.1.10
curl -s ifconfig.me; echo       # public IP, e.g. 190.42.7.25
```
Two different addresses = NAT happened somewhere between you and the Internet. That "somewhere" is your gateway:
```bash
ip route | grep default         # the translator's address
```

### Exercise 2 — Watch your router's translation table (if you can)
If your router runs Linux (most do — OpenWrt, etc.) and you have SSH access:
```bash
cat /proc/net/nf_conntrack      # or: conntrack -L
```
You'll see every active connection with both sides of the translation — the table from section 2, live.

### Exercise 3 — Turn your Linux machine into a NAT router (the classic lab)
Best in **two VMs** on VirtualBox: VM-A (the "router") with two interfaces — one bridged/NAT to the Internet, one internal network; VM-B with only the internal network.

On VM-A:
```bash
# 1. Enable IP forwarding (let the kernel route packets):
sudo sysctl -w net.ipv4.ip_forward=1
# (permanent: uncomment net.ipv4.ip_forward=1 in /etc/sysctl.conf)

# 2. Assign addresses (example: eth0 = Internet side, eth1 = LAN side):
sudo ip addr add 192.168.56.1/24 dev eth1

# 3. The NAT rule — MASQUERADE everything leaving eth0:
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

On VM-B:
```bash
sudo ip addr add 192.168.56.10/24 dev eth0
sudo ip route add default via 192.168.56.1
ping -c 3 8.8.8.8               # works! VM-B has Internet through VM-A's NAT
```

On VM-A, watch the translations happen:
```bash
sudo iptables -t nat -L -v -n              # your rules + packet counters
sudo tcpdump -i eth1 -n icmp               # see 192.168.56.10's ping arrive...
sudo tcpdump -i eth0 -n icmp               # ...and leave with VM-A's public source
```
**That moment — same ping, different source address on each interface — is NAT made visible.**

### Exercise 4 — Port forwarding (DNAT) lab
From the same setup, run a web server on VM-B (`python3 -m http.server 8000`) and add on VM-A:
```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
     -j DNAT --to-destination 192.168.56.10:8000
```
Now connections to VM-A's public address on port 8080 land on VM-B's server. Watch the PREROUTING counter rise per connection.

### Exercise 5 — Modern syntax (nftables)
Most current distros use `nftables` instead of iptables:
```bash
sudo nft add table nat
sudo nft 'add chain nat postrouting { type nat hook postrouting priority 100; }'
sudo nft add rule nat postrouting oifname "eth0" masquerade
sudo nft list ruleset
```

### Exercise 6 — Detect CGNAT
```bash
# Compare your router's WAN IP (check its admin page, or):
curl -s ifconfig.me; echo
traceroute -m 3 8.8.8.8
# Hop 2 in a private or 100.64.x.x range? → your ISP NATs you (CGNAT)
```

---

## 7. Quick Reference Card

| Concept | One-liner |
|---------|-----------|
| Why NAT exists | IPv4 exhaustion — share public IPs |
| What it rewrites | Source IP + port on outgoing packets |
| The NAT table | Memory of who-is-who, so replies route back |
| PAT/Overload | Many private IPs → one public IP via ports |
| MASQUERADE | Linux SNAT that follows a dynamic public IP |
| Port forwarding (DNAT) | Manual rule to allow specific incoming traffic |
| Incoming by default | Dropped — no table entry exists |
| Traversal | UPnP, STUN, TURN, hole punching |
| CGNAT | ISP-level NAT → double NAT |
| IPv6 | Makes NAT unnecessary |
| Linux rule | `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` |

---

Natural next topics: **DNS** (the other service your router provides), **firewalls in depth** (`iptables`/`nftables`/`ufw` — you've now seen the `nat` table, the `filter` table is next), or **how TCP connections work end-to-end** through everything we've covered (DHCP → ARP → NAT → routing → TLS). Which one?