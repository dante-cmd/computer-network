# MAC Addresses — Full Explanation

## 1. What a MAC Address Is

A **MAC address** (Media Access Control address) is the identifier burned into a **network interface** — the "hardware address" of your network card. It operates at **Layer 2 (Data Link)** of the OSI model, meaning it's used for communication *within the local network only*.

Every device that connects via Ethernet or Wi-Fi has one:

```
aa:bb:cc:11:22:33
```

**Key distinction to lock in:**

| | MAC address | IP address |
|--|-------------|------------|
| Layer | 2 (Data Link) | 3 (Network) |
| Scope | Local network only | End-to-end, across the Internet |
| Assigned by | Manufacturer (hardware) | Network admin / DHCP (logical) |
| Changes when... | Almost never (unless spoofed) | You change networks |
| Analogy | Your **name** (who you are) | Your **hotel room number** (where you currently are) |

Remember from the NAT lesson: a packet traveling to Google keeps the same source/destination **IP** end to end, but its **MAC addresses change at every hop** — MACs only matter between you and the next device.

---

## 2. Anatomy: 48 Bits, Two Halves

A MAC address is **48 bits (6 bytes)**, written as 6 hex pairs separated by colons (or hyphens on Windows):

```
        AA : BB : CC : 11 : 22 : 33
        └──── OUI ────┘ └── device ──┘
         (manufacturer)   (serial #)
```

### First 3 bytes — OUI (Organizationally Unique Identifier)
The manufacturer prefix, assigned by the IEEE:

| OUI prefix | Manufacturer |
|------------|--------------|
| `3C:5A:B4` | Google |
| `F0:18:98` | Apple |
| `08:00:27` | VirtualBox VMs |
| `52:54:00` | QEMU/KVM VMs |

You can look up any prefix: `curl https://api.maclookup.app/v2/macs/AA:BB:CC` or search "OUI lookup" online. This is why network scanners can label "Apple device" next to an unknown gadget on your Wi-Fi.

### Last 3 bytes — Device identifier
The manufacturer's serial number for that specific card. Together: 2⁴⁸ ≈ **281 trillion** possible MAC addresses.

### Two special bits in the first byte
- **I/G bit** (least significant): `0` = unicast (one device), `1` = multicast (a group)
- **U/L bit**: `0` = universally assigned (from factory), `1` = **locally administered** (manually set/spoofed — more on this below)

Quick check: if the second hex digit is 2, 6, A, or E → it's a locally administered (likely randomized or spoofed) address.

---

## 3. Special MAC Addresses

| Address | Meaning |
|---------|---------|
| `ff:ff:ff:ff:ff:ff` | **Broadcast** — "everyone on this LAN" (remember DHCP Discover!) |
| `01:00:5e:xx:xx:xx` | IPv4 multicast range |
| `33:33:xx:xx:xx:xx` | IPv6 multicast range |

A broadcast frame is delivered to *every* device on the local network — that's how ARP and DHCP's Discover work when you don't know who to address yet.

---

## 4. How MAC Addresses Are Actually Used

### Inside the frame
When your laptop sends data on the LAN, it wraps the IP packet in an **Ethernet frame**:

```
┌──────────────── ETHERNET FRAME ────────────────┐
│ Dst MAC │ Src MAC │ Type │   IP packet   │ FCS │
└─────────────────────────────────────────────────┘
   "to router" "from me"        (the cargo)
```

### The hop-by-hop rule (critical concept!)

Traveling from your laptop in Lima to a server in the US:

```
Laptop → Router:    Src MAC = laptop,    Dst MAC = router
Router → ISP:       Src MAC = router,    Dst MAC = ISP router     ← REWRITTEN
ISP → ... → Server: Src MAC = last hop,  Dst MAC = server         ← REWRITTEN
```

**IP addresses: same end-to-end. MAC addresses: die at every router.** Your MAC never leaves your LAN — Google has never seen it.

### The switch's MAC table
A switch learns which MAC lives on which port by watching source addresses:

| Port | MAC address |
|------|-------------|
| 1 | aa:bb:cc:11:22:33 (your PC) |
| 2 | 08:00:27:44:55:66 (printer) |

When a frame for `aa:bb:cc:11:22:33` arrives → forward only to port 1 (not flood everywhere, like an old hub). This learning is what makes switches efficient.

### The ARP bridge (connecting two lessons)
You know the destination **IP** (from DNS/routing) but need its **MAC** to build the frame. **ARP** solves it:

1. Broadcast: *"Who has 192.168.1.1? Tell 192.168.1.10"* (sent to `ff:ff:ff:ff:ff:ff`)
2. The gateway replies: *"That's me, my MAC is 08:00:27:aa:bb:cc"*
3. Cached in your **ARP table** for a few minutes

---

## 5. MAC Addresses Aren't Fixed: Spoofing & Randomization

Despite being "burned in," the MAC your OS *uses* is just software-configurable:

### Why change it?
- **Privacy**: networks track devices by MAC (shopping malls, airports log them)
- **MAC filtering bypass**: some networks only allow whitelisted MACs (weak security — easily cloned)
- **Testing/labs**

### MAC randomization (modern privacy feature)
Your **phone already does this**: iOS and Android generate a random MAC per Wi-Fi network (that's the U/L locally-administered bit in action). Windows offers it too. That's why your home router may show your phone with a strange manufacturer.

### Change it on Linux:
```bash
sudo ip link set dev wlan0 down
sudo ip link set dev wlan0 address 02:11:22:33:44:55
sudo ip link set dev wlan0 up
# or with macchanger:
sudo apt install macchanger
sudo macchanger -r wlan0          # random MAC
```

---

## 6. Security Notes

- **MAC filtering is NOT real security**: MACs travel in cleartext in every frame; anyone on the LAN can sniff (`tcpdump`) and clone an allowed one in seconds
- **ARP spoofing/poisoning**: attacker sends fake ARP replies — *"the gateway is at MY MAC"* — and intercepts all LAN traffic (man-in-the-middle). Defense: static ARP entries, Dynamic ARP Inspection on managed switches
- **Port security**: switches can lock a port to a specific MAC, shutting down if another appears

---

## 7. Linux Hands-On Exercises

### Exercise 1 — See all your MAC addresses
```bash
ip link show
#  link/ether a4:b1:c1:...  ← MAC of each interface
#  brd ff:ff:ff:ff:ff:ff    ← broadcast MAC
cat /sys/class/net/eth0/address        # another way
```
Note: `lo` (loopback) has `00:00:00:00:00:00` — no hardware, no MAC needed.

### Exercise 2 — Identify your hardware manufacturer
Take your first 3 bytes and look them up:
```bash
curl -s "https://api.maclookup.app/v2/macs/$(cat /sys/class/net/$(ip route | awk '/default/ {print $5}' | head -1)/address | cut -d: -f1-3)"
```

### Exercise 3 — Explore your ARP table (IP ↔ MAC mapping)
```bash
ip neigh show
# 192.168.1.1 dev wlan0 lladdr 08:00:27:... REACHABLE
ping -c 1 192.168.1.1        # force an ARP exchange
ip neigh show                # check again
```

### Exercise 4 — Watch ARP and MACs live
```bash
sudo tcpdump -i any -e arp              # -e shows MAC addresses
# in another terminal:
ping -c 2 192.168.1.1
```
You'll see: *"Request who-has 192.168.1.1 tell 192.168.1.10"* and the reply, with real MACs on each frame. Then:
```bash
sudo tcpdump -i any -e -n icmp          # watch ping frames: src/dst MAC + src/dst IP
```
**Notice:** ping to your gateway — dst MAC = gateway. Now ping `8.8.8.8` — dst MAC is **still your gateway** (the frame only needs to reach the next hop; the IP carries the final destination). That's the hop-by-hop rule made visible.

### Exercise 5 — Spoof your MAC (lab only!)
```bash
ip link show wlan0 | grep ether         # save your real MAC first!
sudo ip link set wlan0 down
sudo macchanger -r wlan0                # random MAC
sudo ip link set wlan0 up
ip link show wlan0                      # verify — second digit is 2/6/A/E (local bit)
```
Reconnect to Wi-Fi and check your router's client list — you appear as a "new" device.

### Exercise 6 — Spot the VMs
```bash
nmap -sn 192.168.1.0/24                 # scan your LAN
```
Look at the MAC prefixes in the results: any `08:00:27` (VirtualBox) or `52:54:00` (KVM)? Now you know who's virtual.

---

## 8. Quick Reference Card

| Concept | One-liner |
|---------|-----------|
| What | 48-bit hardware ID of a network interface |
| Layer | 2 — local network only |
| Format | 6 hex pairs: `aa:bb:cc:11:22:33` |
| First 3 bytes | OUI = manufacturer |
| Broadcast | `ff:ff:ff:ff:ff:ff` = everyone |
| vs IP | MAC = who, IP = where; MAC changes every hop, IP end-to-end |
| ARP | Asks the LAN: "which MAC has this IP?" |
| Switch table | Learns MAC → port mappings |
| Spoofing | Software-changeable; phones randomize for privacy |
| Security | MAC filtering ≈ useless; ARP spoofing = MITM risk |
| Linux view | `ip link show`, `ip neigh show` |

---

You've now covered the full journey of a packet leaving your machine: **DHCP** (get an IP) → **ARP/MAC** (find the next hop) → **NAT** (cross to the Internet). The remaining big piece of the puzzle: **DNS** (how names become IPs in the first place) — want that next, or would you rather do a **capstone lab** that ties everything together (packet-capture a full web request and identify every protocol in action)?