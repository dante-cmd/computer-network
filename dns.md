# DNS — Full Explanation

## 1. What DNS Is and Why It Exists

**DNS (Domain Name System)** translates human-friendly names into machine-friendly IP addresses:

```
google.com  →  142.250.72.14
```

**Why it exists:** the Internet routes packets by IP address, but humans can't memorize `142.250.72.14` for every site. In the early 1980s there was a single text file (`HOSTS.TXT`) listing every computer on the Internet, copied by hand to every machine. That obviously couldn't scale — so DNS was born (1983, RFC 1034/1035) as a **distributed, hierarchical database**: no single machine knows everything, but together they can answer anything.

**The perfect analogy: a phone book** — or better, a **librarian who doesn't know every book's location but knows exactly who to ask next.**

**Key facts:**
- One of the **oldest and most critical** Internet protocols — if DNS dies, the Internet "stops working" for most people even when every cable is fine
- Runs over **UDP port 53** (and TCP 53 for large responses)
- Globally **distributed** — no central server holds all the answers
- Built on **caching** — answers are remembered to avoid repeating work

---

## 2. The Domain Name Hierarchy

Domain names are a **tree**, read from right to left (most general → most specific):

```
                    www.example.com.
                     │      │     │  │
                     │      │     │  └── root (invisible trailing dot!)
                     │      │     └───── TLD: Top-Level Domain (.com, .org, .pe)
                     │      └─────────── Second-level domain (example)
                     └────────────────── Subdomain / host (www)
```

**The hidden dot:** every fully qualified domain name technically ends in a dot representing the **root**: `www.example.com.` — your browser just hides it.

### Levels of the tree

- **Root (`.`)** — the top; managed by 13 logical root server clusters
- **TLDs** — `.com`, `.org`, `.net`, country codes like `.pe`, `.us`, `.br`, plus hundreds of new ones (`.dev`, `.io`...)
- **Second-level domains** — what you register from a registrar (`example.com`)
- **Subdomains** — free to create once you own the domain (`mail.google.com`, `docs.google.com`...)

**Delegation is the key design:** the root doesn't know where `google.com` is — it only knows who manages `.com`. The `.com` servers don't know Google's IPs — they only know Google's own servers. Each level **delegates** to the next. Nobody carries the whole phone book.

---

## 3. The Four Types of DNS Servers

| Server | Role | Analogy |
|--------|------|---------|
| **Recursive resolver** | Does the full lookup work *for you* | The librarian |
| **Root servers** | Point to TLD servers | The library's index of sections |
| **TLD servers** | Point to each domain's authoritative server | The section's catalog |
| **Authoritative servers** | Hold the actual records for a domain | The bookshelf itself |

Your recursive resolver is usually run by your **ISP**, or a public one:

| Public resolver | Addresses |
|-----------------|-----------|
| Google DNS | `8.8.8.8`, `8.8.4.4` |
| Cloudflare | `1.1.1.1`, `1.0.0.1` |
| Quad9 (security-filtered) | `9.9.9.9` |

Remember DHCP? Option 6 handed you exactly this — your resolver's address. Your home router usually gives out *itself* as DNS and forwards queries to the ISP.

---

## 4. The Full Resolution Journey (Step by Step)

You type `www.example.com` in your browser. What actually happens:

```
YOUR LAPTOP
  │
  ├─ 1. Check local cache (browser → OS cache → /etc/hosts)
  │     Already know it? Done. No network traffic at all.
  │
  └─ 2. Ask the RECURSIVE RESOLVER (e.g., 8.8.8.8)
        "Please get me www.example.com — do all the work"
                │
                ├─ Resolver checks ITS cache first
                │
                └─ Not cached? Start at the top — ITERATIVE queries:
                      │
                      ├─ 3. ROOT server: "Who handles .com?"
                      │        ← "Ask the .com TLD servers: a.gtld-servers.net"
                      │
                      ├─ 4. TLD server: "Who handles example.com?"
                      │        ← "Ask ns1.example.com's authoritative servers"
                      │
                      └─ 5. AUTHORITATIVE server: "What's www.example.com?"
                               ← "93.184.216.34, and you may cache this for 3600s (TTL)"
                │
                ├─ 6. Resolver caches the answer (for TTL seconds)
                └─ 7. Returns 93.184.216.34 to your laptop
  │
  └─ 8. Your laptop caches it too, then opens the TCP connection
        (and only NOW does the HTTP traffic begin)
```

### Two query types to name correctly
- **Recursive query** — client → resolver: *"don't come back without the final answer"*
- **Iterative queries** — resolver → root/TLD/authoritative: *"give me your best answer or referral"*; the resolver chases referrals itself

**Total time:** first lookup maybe 50–200 ms across 3–4 servers. Second time (cached): **~0 ms**. Caching is what makes this planet-scale system fast.

---

## 5. TTL and Caching — The Performance Engine

Every DNS record carries a **TTL (Time To Live)** in seconds — how long anyone may cache it:

```
www.example.com.  3600  IN  A  93.184.216.34
                   └─── cache me for 1 hour
```

- **High TTL** (hours/days): less DNS traffic, faster for everyone — but slow to change records
- **Low TTL** (60–300s): used before migrations/failover so changes propagate fast
- TTL counts **down** in caches — a record cached 10 minutes ago shows TTL 3000 instead of 3600. That's how you can tell if an answer came fresh or from cache (Exercise 4 shows this live!)

**Every layer caches:** your browser, your OS, your router, the resolver. This is why "DNS changes take up to 24–48h to propagate" — you're waiting for old caches worldwide to expire.

---

## 6. DNS Record Types

A zone file (the database on the authoritative server) holds records of many types:

| Type | Purpose | Example |
|------|---------|---------|
| **A** | Name → IPv4 | `example.com → 93.184.216.34` |
| **AAAA** | Name → IPv6 | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias → another name | `www.example.com → example.com` |
| **MX** | Mail servers (with priority) | `example.com → 10 mail.example.com` |
| **NS** | Which servers are authoritative | `example.com → ns1.example.com` |
| **TXT** | Text data: SPF/DKIM (anti-spam), verification | `"v=spf1 include:_spf.google.com ~all"` |
| **PTR** | Reverse: IP → name | `34.216.184.93.in-addr.arpa → ...` |
| **SOA** | Zone metadata: serial, admin, timers | Every zone has exactly one |
| **SRV** | Service location (host + port) | Used by SIP, XMPP, AD |
| **CAA** | Which CAs may issue TLS certs | Security control |

**CNAME rules:** an alias must point to another *name* (never an IP), and you can't put a CNAME at the zone root (`example.com` itself) alongside other records.

**Reverse DNS:** the PTR system flips the IP into a special domain — `1.2.3.4` becomes `4.3.2.1.in-addr.arpa`. Mail servers check PTR records to fight spam.

---

## 7. Transport: Why UDP 53 (and When TCP)

Same logic as DHCP — DNS defaults to **UDP port 53**:
- Queries are tiny (one question), answers usually tiny (one response) — a 3-way TCP handshake would cost more than the entire exchange
- No connection state to maintain on massively-loaded root servers

**TCP 53 is used when:**
- Response exceeds ~512 bytes historically (now 1232 with **EDNS**) — e.g., DNSSEC-signed answers
- **Zone transfers** (AXFR) — copying an entire zone between authoritative servers

The query itself includes a 16-bit **ID**, a flags section (QR = query/response, RD = recursion desired), the question, and in responses: answer/authority/additional sections.

---

## 8. Local Resolution: The Files Your OS Checks First

Before any network query, Linux checks locally:

1. **`/etc/hosts`** — static, administrator-controlled, always wins:
```
127.0.0.1   localhost
192.168.1.20  myserver myserver.local
```
Great for lab shortcuts or blocking sites (`0.0.0.0 ads.example.com`).

2. **`/etc/resolv.conf`** — which resolvers to ask:
```
nameserver 192.168.1.1
nameserver 8.8.8.8
```
(Often auto-written by DHCP/NetworkManager — editing it by hand may be overwritten.)

3. **`/etc/nsswitch.conf`** — the *order* of checks: `hosts: files dns` = hosts file first, then DNS.

---

## 9. DNS Security

DNS was designed in 1983 with **zero authentication**, creating classic attacks:

- **Cache poisoning**: attacker floods a resolver with forged replies to make it cache *"bank.com → attacker's IP"*. Historic Kaminsky attack (2008) forced source-port randomization.
- **DNS spoofing on the LAN**: attacker simply answers your broadcast-range query faster than the real resolver (easy on open Wi-Fi).
- **DDoS via amplification**: small query ("give me ANY record") → huge response sent with *spoofed victim source address* — DNS becomes a traffic multiplier.

### The fixes
- **DNSSEC**: authoritative servers cryptographically **sign** records; resolvers verify. Protects integrity — you know the answer is authentic — but does *not* encrypt anything.
- **DoH (DNS over HTTPS, port 443)** and **DoT (DNS over TLS, port 853)**: **encrypt** the query itself so ISPs/observers on the LAN can't see or tamper with your lookups. Firefox/Chrome use DoH by default in many regions.
- **Pi-hole / AdGuard Home**: run your own filtering resolver at home — blocks ads/trackers at the DNS level for every device.

---

## 10. Linux Hands-On Exercises

### Exercise 1 — Your basic toolkit
```bash
sudo apt install dnsutils       # gives you dig, nslookup, host
dig google.com                  # the power tool — full detail
nslookup google.com             # quick and classic
host google.com                 # terse output
```
In `dig` output find: `status: NOERROR`, the `ANSWER SECTION` (the actual records), `Query time`, and `SERVER:` (which resolver answered).

### Exercise 2 — Query every record type
```bash
dig A google.com +short         # IPv4
dig AAAA google.com +short      # IPv6
dig MX gmail.com +short         # mail servers (note priorities)
dig NS google.com +short        # authoritative servers
dig TXT google.com +short       # SPF/verification strings
dig -x 8.8.8.8 +short           # reverse lookup (PTR)
```

### Exercise 3 — Trace the full resolution path (best exercise!)
```bash
dig +trace www.example.com
```
Watch `dig` start at the **root servers**, get referred to **.com TLD**, then to the **authoritative servers** — exactly the journey from section 4, live. Read each hop's `NS` records.

### Exercise 4 — See caching and TTL countdown
```bash
dig example.com                 # note TTL (e.g., 3600) and Query time: 40 ms
dig example.com                 # again, immediately
```
Second run: `Query time: 0-1 ms` and a **lower TTL** — proof your resolver answered from cache. You just watched caching work.

### Exercise 5 — Ask a specific server
```bash
dig @8.8.8.8 example.com        # Google resolver
dig @1.1.1.1 example.com        # Cloudflare
dig @192.168.1.1 example.com    # your router — compare query times!
```

### Exercise 6 — Capture DNS traffic
```bash
sudo tcpdump -i any -n port 53  # terminal 1
dig google.com                  # terminal 2
```
You'll see your query go to the resolver from `/etc/resolv.conf`, on **UDP 53**. Try `dig +tcp google.com` and capture again — TCP 53 this time.

### Exercise 7 — Break it and diagnose (learn the #1 failure mode)
```bash
ping -c 2 8.8.8.8               # works? Internet connectivity is FINE
ping -c 2 google.com            # fails? → DNS problem, not connectivity
```
This two-command test diagnoses half of all "the Internet is down" complaints. Also try corrupting `/etc/resolv.conf` in a VM and observe every app failing while IPs still ping.

### Exercise 8 — Play with /etc/hosts
```bash
echo "192.168.56.10 mylab.local" | sudo tee -a /etc/hosts
ping mylab.local                # resolves without any DNS server
```

### Exercise 9 — Run your own resolver/forwarder (lab)
```bash
sudo apt install dnsmasq
# /etc/dnsmasq.conf:  listen-address=127.0.0.1
sudo systemctl restart dnsmasq
dig @127.0.0.1 google.com       # now YOU are the DNS server (forwarding upstream)
```

---

## 11. How DNS Connects Everything You've Learned

Your complete "open a website" story now:

1. **DHCP** → your laptop gets IP, gateway, and **DNS resolver** address (UDP 67/68)
2. **DNS** → `example.com` becomes `93.184.216.34` via resolver → root → TLD → authoritative (UDP 53)
3. **Routing decision** → destination is outside my subnet → send to default gateway
4. **ARP** → "what's the gateway's MAC?" — frame gets gateway's MAC (Layer 2)
5. **NAT** → router rewrites private source to public IP:port, logs it in the table
6. **TCP** 3-way handshake to `93.184.216.34:443`, then TLS, then HTTP

Every protocol you've studied appears in a single page load. 

---

## 12. Quick Reference Card

| Concept | One-liner |
|---------|-----------|
| Job | Name → IP (and much more) |
| Port | UDP 53 (TCP 53 for big answers/transfers) |
| Structure | Tree: root `.` → TLD → domain → subdomain, right to left |
| Resolver | Does recursive work for you (8.8.8.8, 1.1.1.1, ISP) |
| Root/TLD/Authoritative | Answer with referrals down the tree |
| TTL | How long answers may be cached |
| A / AAAA | IPv4 / IPv6 address records |
| CNAME / MX / NS / TXT / PTR | Alias / mail / authority / text / reverse |
| /etc/hosts | Local static resolution, checked first |
| Attacks | Cache poisoning, spoofing → fixed by DNSSEC, DoH/DoT |
| Key diagnostic | `ping 8.8.8.8` works + `ping google.com` fails = DNS problem |

---

Want the **capstone lab** next — capture one full HTTPS page load in tcpdump/Wireshark and identify every single protocol we've covered in action (DHCP, DNS, ARP, NAT, TCP, TLS)? That exercise ties the whole series together. Or if you prefer, we can dive into **TCP/UDP transport in depth** first.