# IP Addresses — Full Explanation

## 1. What an IP Address Is

An **IP address** is a logical identifier assigned to a device on a network — it answers the question *"where are you?"* so packets can be routed to you. Think of it like a postal address: the street + house number lets anyone in the world deliver mail to you.

- **MAC address** = who you are (burned into the hardware, like your name)
- **IP address** = where you are (assigned by the network, changes if you move)

---

## 2. IPv4 Anatomy

An IPv4 address is **32 bits**, written as 4 decimal numbers (octets) separated by dots:

```
192.168.1.10
```

Each octet is 8 bits (0–255). In binary:

```
192      . 168      . 1        . 10
11000000 . 10101000 . 00000001 . 00001010
```

Try it yourself on Linux:
```bash
# Convert decimal to binary quickly:
echo "obase=2; 192" | bc        # 11000000
```

### Why max 255?
8 bits → 2⁸ = 256 combinations → values 0 to 255.

### Total IPv4 space
2³² ≈ **4.3 billion** addresses — which ran out, hence IPv6 and NAT (more below).

---

## 3. Network Part vs Host Part

Every IP has two parts:
- **Network portion** — which network you're on (the "street")
- **Host portion** — which device you are (the "house number")

The **subnet mask** tells you where the split is. A mask has 1s for the network bits and 0s for host bits:

```
IP:        192.168.1.10
Mask:      255.255.255.0   =  11111111.11111111.11111111.00000000
                                 └─── network ───┘ └── host ──┘
```

Devices are on the **same network** if their network portions match. Same network → talk directly (via switch/ARP). Different network → send through the **default gateway** (router).

---

## 4. CIDR Notation (The `/24` Thing)

CIDR (Classless Inter-Domain Routing) just counts the 1-bits in the mask:

| CIDR | Mask | Host bits | Usable hosts |
|------|------|-----------|--------------|
| /8 | 255.0.0.0 | 24 | 16,777,214 |
| /16 | 255.255.0.0 | 16 | 65,534 |
| /24 | 255.255.255.0 | 8 | 254 |
| /25 | 255.255.255.128 | 7 | 126 |
| /26 | 255.255.255.192 | 6 | 62 |
| /30 | 255.255.255.252 | 2 | 2 (point-to-point links) |

**Usable hosts formula:** 2^(host bits) − 2 (minus network address and broadcast address).

### Three special addresses in every subnet (`192.168.1.0/24` example)
- **Network address**: `192.168.1.0` — names the network itself, not assignable
- **Broadcast address**: `192.168.1.255` — "everyone on this LAN"
- **Usable range**: `192.168.1.1` – `192.168.1.254`

Linux exercise — compute these automatically:
```bash
sudo apt install ipcalc
ipcalc 192.168.1.10/24
```
Output shows network, broadcast, first/last host, and total hosts — great for checking your homework.

---

## 5. Public vs Private Addresses

**Private ranges** (reserved by RFC 1918, never routed on the Internet — free for anyone to reuse internally):

| Range | CIDR | Typical use |
|-------|------|-------------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | Big companies |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | Medium networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | Home routers |

Your laptop has a **private** IP (e.g. `192.168.1.10`). Your router has the **public** IP assigned by your ISP. **NAT** on the router translates between them — that's how your whole household shares one public address.

Check yours:
```bash
ip addr show                    # your private IP
curl ifconfig.me                # your public IP as seen from the Internet
```
Different results? That's NAT at work.

---

## 6. Special-Purpose IPv4 Addresses

| Address | Meaning |
|---------|---------|
| `127.0.0.0/8` | **Loopback** — your own machine. `127.0.0.1` = "me" |
| `0.0.0.0` | "Any interface" / unspecified — used when a server listens on all NICs |
| `169.254.0.0/16` | **APIPA/link-local** — self-assigned when DHCP fails (a symptom of trouble) |
| `255.255.255.255` | Limited broadcast |
| `224.0.0.0/4` | **Multicast** (one-to-many, e.g. streaming to a group) |

Try it:
```bash
ping -c 2 127.0.0.1       # always works — tests your own TCP/IP stack
ip addr show lo           # the loopback interface
```

---

## 7. Static vs Dynamic (DHCP)

- **Dynamic**: a DHCP server (usually your router) leases you an IP + gateway + DNS automatically.
- **Static**: you configure it manually — used for servers, printers, routers so their address never changes.

See your lease info:
```bash
cat /var/lib/dhcp/dhclient.leases 2>/dev/null || journalctl -u NetworkManager | grep -i dhcp
```

---

## 8. IPv6 — The Successor

IPv6 fixes address exhaustion: **128 bits** ≈ 3.4 × 10³⁸ addresses (every grain of sand on Earth could have billions).

Written as 8 groups of 4 hex digits:
```
2001:0db8:0000:0000:0000:ff00:0042:8329
```

**Shortening rules:**
1. Drop leading zeros in each group: `0db8` → `db8`
2. One run of all-zero groups → `::`

```
2001:db8::ff00:42:8329
```

IPv6 address types:
- `::1` — loopback
- `fe80::/10` — link-local (every interface auto-generates one; check `ip -6 addr`)
- Global unicast — the "public" IPv6
- No broadcast in IPv6 — replaced by multicast

Key differences from IPv4: no NAT needed (in principle), no broadcast, auto-configuration (SLAAC) built in, headers simplified.

Linux commands:
```bash
ip -6 addr show              # your IPv6 addresses
ping -c 3 ipv6.google.com    # test IPv6 connectivity (if your ISP supports it)
```

---

## 9. Old "Classes" (Historical — You'll See Them in Exams)

Before CIDR, addresses were carved into fixed classes:

| Class | First octet | Default mask | Purpose |
|-------|------------|--------------|---------|
| A | 1–126 | /8 | Huge networks |
| B | 128–191 | /16 | Medium |
| C | 192–223 | /24 | Small |
| D | 224–239 | — | Multicast |
| E | 240–255 | — | Experimental |

It's obsolete in practice (CIDR replaced it in 1993), but courses still test it — now you know why `192.168.x.x` is "Class C".

---

## 10. Subnetting — The Core Skill

**Problem:** Split `192.168.1.0/24` into 4 equal subnets.

You need 4 subnets → borrow 2 bits (2² = 4) → new mask `/26` → 64 addresses each, 62 usable:

| Subnet | Network | Range | Broadcast |
|--------|---------|-------|-----------|
| 1 | 192.168.1.0/26 | .1 – .62 | .63 |
| 2 | 192.168.1.64/26 | .65 – .126 | .127 |
| 3 | 192.168.1.128/26 | .129 – .190 | .191 |
| 4 | 192.168.1.192/26 | .193 – .254 | .255 |

**Quick mental method:** block size = 256 − last mask octet. For /26, mask is .192 → block size 64 → subnets start at 0, 64, 128, 192.

Verify with:
```bash
ipcalc 192.168.1.0/26
```

---

## 11. Linux Practice Exercises

1. **Read your own config:** `ip addr`, `ip route` — identify IP, mask, network address, gateway.
2. **Same-network test:** ping your gateway (same network — direct). Then `traceroute 8.8.8.8` (different network — routed). Notice hop 1 is always your gateway.
3. **Subnetting drill:** use `ipcalc` to check your work — split `10.0.0.0/24` into 8 subnets. What mask? How many hosts each?
4. **Loopback server:** run `python3 -m http.server 8080`, then `curl 127.0.0.1:8080` — traffic never leaves your machine (verify with `sudo tcpdump -i lo port 8080`).
5. **Watch ARP:** `ip neigh show`, ping a LAN device, then `ip neigh show` again — see the new IP→MAC entry.
6. **Public vs private:** compare `ip addr` output with `curl ifconfig.me`.

---

### Quick Reference

| Concept | One-liner |
|---------|-----------|
| IPv4 | 32 bits, dotted decimal |
| Subnet mask | Separates network bits from host bits |
| CIDR /n | n = number of network bits |
| Private IP | LAN only, needs NAT for Internet |
| Public IP | Globally unique, ISP-assigned |
| Loopback | 127.0.0.1 = yourself |
| Gateway | Router to leave your subnet |
| IPv6 | 128 bits, hex, fixes exhaustion |

Want to go deeper into **subnetting with a full practice worksheet**, or move on to **how routing actually works between networks**?