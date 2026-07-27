# DHCP — Full Explanation

## 1. What DHCP Is and Why It Exists

**DHCP (Dynamic Host Configuration Protocol)** automatically assigns network configuration to devices when they join a network. Without it, an admin would have to manually configure on *every* device:

- IP address
- Subnet mask
- Default gateway
- DNS servers
- (optionally) domain name, NTP servers, TFTP/boot servers, and more

Imagine doing that by hand for 500 laptops — and redoing it every time a laptop moves to another floor. DHCP solves this: plug in or join Wi-Fi → get a working config in under a second.

**Key facts:**
- Runs over **UDP**: server listens on port **67**, client on port **68**
- Defined by **RFC 2131**
- Your home router almost always runs a DHCP server; in companies it's a dedicated server or the core switch/firewall

---

## 2. The DORA Process — How You Get an Address

When your laptop joins a network, it has **no IP yet**, so it can't address anyone directly — everything happens via **broadcasts** (sent to `255.255.255.255` from `0.0.0.0`). Four steps:

```
CLIENT                                          SERVER
  |                                               |
  |--- 1. DISCOVER  (broadcast) ---------------->|  "Is there a DHCP server?"
  |<-- 2. OFFER     ------------------------------|  "Here's 192.168.1.50 for you"
  |--- 3. REQUEST   (broadcast) ---------------->|  "I'll take that one"
  |<-- 4. ACK       ------------------------------|  "Confirmed, it's yours"
  |                                               |
  ✓ Client configures: IP, mask, gateway, DNS
```

**Mnemonic: DORA** — Discover, Offer, Request, Ack.

### Step by step in detail

1. **DHCPDISCOVER** — Client broadcasts: *"I'm MAC aa:bb:cc:dd:ee:ff, I need an address."* Source `0.0.0.0:68` → destination `255.255.255.255:67`.
2. **DHCPOFFER** — Every server that hears it offers a config: IP, mask, lease time, gateway, DNS, plus the server's own identifier.
3. **DHCPREQUEST** — The client picks one offer (usually the first) and *broadcasts* its acceptance. Broadcasting matters: it implicitly tells the *other* servers "thanks, but no" so they can reclaim their offered IPs.
4. **DHCPACK** — The chosen server finalizes the lease. The client now configures its interface.

If something goes wrong:
- **DHCPNAK** — server refuses (e.g., the offered IP is no longer valid) → client restarts with Discover.
- **DHCPDECLINE** — client found the IP is already in use (it ARPs to check) → server marks it bad, client restarts.

---

## 3. The Lease — You Don't Own the IP, You Rent It

Every assignment is a **lease** with a time limit (home routers: often 24h; enterprises: hours to days). This lets the server reclaim addresses from devices that left.

### Renewal timeline (T1 and T2 timers)

```
Lease granted at t=0
        |
        ├── T1 (50% of lease): RENEWING
        |     Client unicasts DHCPREQUEST directly to its server
        |     "Can I keep 192.168.1.50?" → server ACKs, lease restarts
        |
        └── T2 (87.5% of lease): REBINDING
              If the original server never answered, client *broadcasts*
              a REQUEST — any server may renew it
              |
              └── Lease expires (100%): address lost, start over with DORA
```

**Why renew at 50%?** So a temporarily unreachable server doesn't cause an outage — the client has half the lease to retry.

### When a client rejoins
If it reboots and remembers its old lease, it skips straight to **REQUEST** for its previous address (2 packets instead of 4). If the address is still free → ACK; if taken → NAK → full DORA.

### Other DHCP message types

| Message | Purpose |
|---------|---------|
| DHCPRELEASE | Client politely gives the address back (e.g., `dhclient -r`) |
| DHCPINFORM | Client already has an IP (static) but wants other options (DNS, etc.) |

---

## 4. What a DHCP Server Actually Decides

A server manages **scopes** (pools) per subnet, e.g.:

```
Subnet:        192.168.1.0/24
Pool:          192.168.1.100 – 192.168.1.200   (handed out dynamically)
Excluded:      192.168.1.1 – .99               (for gateway, servers, printers)
Gateway:       192.168.1.1
DNS:           192.168.1.1 (or 8.8.8.8, 1.1.1.1)
Lease:         86400 seconds (24h)
```

### Reservations (static DHCP)
You can bind a specific **MAC address → fixed IP**. The printer with MAC `aa:bb:cc:11:22:33` *always* gets `192.168.1.10`. You keep centralized management but a predictable address — the standard approach for printers, cameras, and small servers.

### DHCP Options (the extras beyond IP/mask)
Numbered fields in the ACK. Common ones:

| Option | Meaning |
|--------|---------|
| 1 | Subnet mask |
| 3 | Router (default gateway) |
| 6 | DNS servers |
| 15 | Domain name |
| 51 | Lease time |
| 53 | DHCP message type |
| 54 | Server identifier |
| 66/67 | TFTP server + bootfile (used by **PXE** network booting) |

---

## 5. DHCP Across Subnets: Relay Agents (DHCP Relay / IP Helper)

Broadcasts **don't cross routers** — so how does one central DHCP server serve 50 subnets? 

**DHCP Relay** (RFC 3046): the router on each subnet listens for DHCP broadcasts and **forwards them as unicast** to the central server, adding a field (`giaddr`) saying which subnet the request came from — so the server picks the right scope. Replies travel back through the relay.

```
Client → broadcast DISCOVER → Router/relay → unicast to DHCP server → OFFER back via relay
```

On Cisco gear this is the `ip helper-address` command. This is why enterprises don't need a DHCP server per floor.

---

## 6. What Happens When DHCP Fails

If no server answers, most OSes self-assign a **link-local address** from `169.254.0.0/16` (**APIPA**). It's a diagnostic red flag:

```bash
ip addr show
# inet 169.254.x.x/16  ← DHCP failed. Cable? Wi-Fi? Server down? Pool exhausted?
```

Devices with APIPA addresses can talk to *each other* on the local link but have **no gateway, no DNS, no Internet**.

---

## 7. Security Aspects (Why Networks Watch DHCP Closely)

DHCP has **no authentication**, which enables classic attacks:

- **Rogue DHCP server**: an attacker plugs in a device that answers Discover faster than the real server → victims get a malicious **gateway/DNS** → all their traffic flows through the attacker (man-in-the-middle). Defense: **DHCP Snooping** on switches (only trusted ports may send Offers).
- **DHCP starvation**: an attacker floods Discovers with fake MACs until the pool is exhausted → legit users get nothing (DoS). Defense: **port security** limiting MACs per port.

---

## 8. DHCPv6 — Quick Contrast

IPv6 introduces **SLAAC** (stateless autoconfiguration): routers advertise the prefix and hosts build their own addresses — often no DHCP at all. **DHCPv6** still exists for "stateful" assignment or to hand out extras like DNS. Differences: uses UDP 546/547, multicast instead of broadcast, and clients identify with a **DUID** rather than just MAC.

---

## 9. Linux Hands-On Exercises

### Exercise 1 — See your current lease
```bash
ip addr show                       # note "dynamic" flag on your address
ip route show                      # gateway came via DHCP
cat /etc/resolv.conf               # DNS came via DHCP

# Lease details (location varies):
sudo cat /var/lib/NetworkManager/dhclient-*.lease 2>/dev/null
sudo cat /var/lib/dhcp/dhclient.leases 2>/dev/null
journalctl -u NetworkManager --no-pager | grep -i dhcp | tail -20
```
Look for: `lease-time`, `dhcp-server-identifier`, `routers`, `domain-name-servers`.

### Exercise 2 — Capture DORA with tcpdump (the best way to learn it)
```bash
# Terminal 1:
sudo tcpdump -i any -n port 67 or port 68 -v

# Terminal 2 — force a renewal / full request:
sudo dhclient -r                   # release (sends DHCPRELEASE)
sudo dhclient -v                   # request again — watch DORA appear in terminal 1
```
You'll see: `DHCP-Message Option 53, length 1: Discover` → `Offer` → `Request` → `ACK`, plus options 1, 3, 6, 51, 54.

⚠️ On Wi-Fi with NetworkManager you can also just toggle the connection: `nmcli con down "SSID" && nmcli con up "SSID"`.

### Exercise 3 — Renew and release manually
```bash
sudo dhclient -r -v && sudo dhclient -v        # release + obtain
nmcli networking off && nmcli networking on    # NetworkManager alternative
```

### Exercise 4 — Build your own DHCP server (real skill)
In a VM lab (two VMs on a **host-only/internal network** — never run this on a real LAN, you'd become a rogue server!):

```bash
sudo apt install isc-dhcp-server
```

Edit `/etc/dhcp/dhcpd.conf`:
```
subnet 192.168.56.0 netmask 255.255.255.0 {
  range 192.168.56.100 192.168.56.150;
  option routers 192.168.56.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
  default-lease-time 600;
  max-lease-time 7200;

  host printer {                       # reservation example
    hardware ethernet 08:00:27:aa:bb:cc;
    fixed-address 192.168.56.10;
  }
}
```
Tell it which interface to serve (`/etc/default/isc-dhcp-server`: `INTERFACESv4="eth1"`), then:
```bash
sudo systemctl restart isc-dhcp-server
sudo journalctl -u isc-dhcp-server -f      # watch DISCOVER/OFFER/REQUEST/ACK live
```
Set the server VM's interface to a static IP in that subnet first (`192.168.56.1/24`), then boot the client VM and watch it get an address. Check `/var/lib/dhcp/dhcpd.leases` — the server's lease database.

### Exercise 5 — Lighter alternative: dnsmasq
```bash
sudo apt install dnsmasq
# /etc/dnsmasq.conf:
#   interface=eth1
#   dhcp-range=192.168.56.100,192.168.56.150,12h
sudo systemctl restart dnsmasq
```

### Exercise 6 — Break it and diagnose
Unplug the cable / stop the server, then request a lease. Watch the client retry, then eventually self-assign `169.254.x.x`. Now you know how to spot DHCP failure instantly in the field.

---

## 10. Quick Reference Card

| Item | Value |
|------|-------|
| Transport | UDP — server 67, client 68 |
| Process | **DORA**: Discover → Offer → Request → ACK |
| First packets | From `0.0.0.0` to `255.255.255.255` (broadcast) |
| Renewal | Unicast REQUEST at **T1 = 50%** of lease |
| Rebinding | Broadcast REQUEST at **T2 = 87.5%** |
| Failure symptom | `169.254.x.x` (APIPA) |
| Cross-subnet | DHCP relay / IP helper (`giaddr`) |
| Reservations | MAC → fixed IP |
| Main risks | Rogue server, starvation → DHCP snooping |

---

# DHCP Architecture: Who Is Who

Great question — this is the key to really understanding it. DHCP is **not one program**. It's a **protocol** (a set of rules), and different programs implement each role. Let's untangle it.

---

## 1. First, the Distinction: Protocol vs Program

- **DHCP the protocol** = the *rules of the conversation* — what messages exist (DORA), what fields they carry, who answers whom. It's just a document (RFC 2131). You can't "run" a protocol.
- **DHCP software** = the *actual programs* that follow those rules. There are two roles, so there are two kinds of programs:
  - **DHCP client** — runs on *your devices* (laptop, phone, smart TV...)
  - **DHCP server** — runs on the device that *hands out* addresses

Think of it like language: "English" is a protocol (grammar rules); a person speaking English is the program. You need at least two parties speaking the same language for a conversation.

---

## 2. The Client Side (Your Devices)

Every networked device has a small DHCP client program built into its operating system:

| Device | DHCP client software |
|--------|---------------------|
| Linux | `dhclient`, `dhcpcd`, or NetworkManager/systemd-networkd's internal client |
| Windows | "DHCP Client" service (`dhcpcore.dll`) |
| Android/iPhone/macOS | built into the OS network stack |

It starts automatically at boot or when you join Wi-Fi, shouts *"DISCOVER — anybody there?"*, receives the offer, and configures the interface for you. You never see it working.

Check yours on Linux:
```bash
ps aux | grep -i dhc          # see if a dhcp client process is running
systemctl status NetworkManager | head   # NetworkManager has a built-in client
```

---

## 3. The Server Side (This Is Where Your Router Comes In)

Yes — **your home router includes a DHCP server program inside it**. A home router isn't just a router; it's a box running several programs at once:

```
┌───────────── YOUR HOME ROUTER ─────────────┐
│                                            │
│   Routing program  → forwards packets      │
│   NAT program      → private↔public IP     │
│   DHCP SERVER      → hands out addresses   │ ← this one
│   DNS forwarder    → passes DNS queries    │
│   Wi-Fi access point                       │
│   Switch (the 4 LAN ports)                 │
│   Firewall                                 │
│                                            │
└────────────────────────────────────────────┘
```

That's why when you connect your phone to Wi-Fi, it instantly gets `192.168.1.x` — the router's built-in DHCP server answered.

In a **company network**, the DHCP server is usually a **separate, dedicated machine** (a Linux/Windows server), not the router:

| Environment | Where the DHCP server lives | Typical software |
|-------------|------------------------------|------------------|
| Home | Inside the router | router firmware (often `dnsmasq` under the hood!) |
| Small office | Router, firewall, or a small server | dnsmasq, router OS |
| Enterprise | Dedicated servers (redundant pair) | ISC `dhcpd`, Windows Server DHCP, Kea |

---

## 4. Where Does UDP Fit In?

Now the third piece. Server and client are programs — but how do their messages travel? Through the network stack, and DHCP chose **UDP** as its transport:

```
┌──────────────────────────────────────────┐
│  DHCP client program   DHCP server program│  ← the applications
├──────────────────────────────────────────┤
│        UDP  port 68  ←→  UDP  port 67     │  ← the transport
├──────────────────────────────────────────┤
│                 IP                        │  ← addressing/routing
├──────────────────────────────────────────┤
│           Ethernet / Wi-Fi                │  ← the physical link
└──────────────────────────────────────────┘
```

- The **client** sends from its UDP **port 68**
- The **server** listens on UDP **port 67**

**Why UDP and not TCP?** Two reasons:

1. **Broadcasts.** At the start the client has *no IP address* and doesn't know *any* server's address — it can only shout to everyone (`255.255.255.255`). TCP requires a one-to-one connection between two known addresses; UDP can broadcast.
2. **Simplicity.** A DHCP exchange is 4 tiny packets. TCP's connection setup/teardown would be overhead for nothing — and if a packet is lost, the client just retries anyway.

You can verify the server port yourself — on a machine running a DHCP server (or check your router's admin page):
```bash
ss -ulpn | grep :67     # "who is listening on UDP 67?"
```

---

## 5. Putting It All Together — One Complete Scene

You open your laptop at home and join Wi-Fi:

1. Your laptop's **DHCP client** (a program in the OS) crafts a DHCPDISCOVER message
2. It wraps it in **UDP** (src port 68 → dst port 67), inside an IP packet from `0.0.0.0` to broadcast `255.255.255.255`
3. Every device on the LAN receives it, but only the **DHCP server program inside your router** understands and cares
4. The server picks a free address from its pool (`192.168.1.50`) and sends back an OFFER — again via UDP 67 → 68
5. REQUEST and ACK complete DORA
6. Your laptop's client configures the interface: IP, mask, gateway (the router itself, `192.168.1.1`), DNS
7. Done — the programs go quiet until renewal time (50% of the lease)

```
 YOUR LAPTOP                    YOUR ROUTER
┌──────────────┐              ┌───────────────────┐
│ DHCP client  │ ══ UDP ════► │ DHCP server       │
│ (OS program) │   68 ↔ 67    │ (router firmware) │
└──────────────┘              └───────────────────┘
```

---

## 6. The One-Paragraph Summary

> **DHCP is a protocol** — rules for a conversation. It's implemented by **two programs**: a **client** built into every device's OS, and a **server** that hands out addresses. In homes, the server lives **inside the router** (one of several programs the router runs); in companies it's usually a dedicated server. The two programs talk using **UDP** — client port 68, server port 67 — because UDP supports the broadcasts needed when the client doesn't have an IP yet.

---

**Mini-exercise** to cement it: run `sudo tcpdump -i any -n udp port 67 or port 68` while disconnecting and reconnecting your Wi-Fi. You'll literally watch the two programs talking over UDP. Want to go through how to read that output packet by packet?