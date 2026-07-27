# Computer Networking Notes

A collection of hands-on networking guides built around the Linux command line. Each topic is explained from first principles and paired with concrete exercises you can run on a real machine or a small VM lab.

## What's here

| File | Topic | What you'll learn |
|------|-------|-------------------|
| [`computer-network.md`](computer-network.md) | Networks overview | OSI/TCP-IP models, switches, routers, firewalls, and a broad set of Linux exercises |
| [`ip-addreses.md`](ip-addreses.md) | IP addressing | IPv4/IPv6, subnetting, CIDR, public vs private addresses, static vs dynamic |
| [`mac.md`](mac.md) | MAC addresses | Layer 2 hardware addressing, ARP, switches, MAC spoofing and security |
| [`dhcp.md`](dhcp.md) | DHCP | The DORA process, leases, reservations, DHCP relay, and running your own server |
| [`dns.md`](dns.md) | DNS | Resolution hierarchy, record types, caching, DNSSEC, DoH/DoT, and diagnostics |
| [`nat.md`](nat.md) | NAT | Source/destination NAT, port forwarding, CGNAT, NAT traversal, and `iptables`/`nftables` labs |

## Who this is for

- Students learning networking fundamentals
- Anyone who wants to stop guessing why "the Internet is down"
- Linux users who prefer to understand a protocol by watching its packets

## How to use these notes

1. Start with [`computer-network.md`](computer-network.md) for the big picture.
2. Read the protocol-specific files in any order — they cross-reference each other where it helps.
3. Run the commands in the exercises. Most only need a Linux terminal; a few use two VMs for safety.
4. Watch the live traffic with `tcpdump`/`Wireshark` — that's where the concepts click.

## Lab safety

- Most commands are read-only and safe to run on your own machine.
- Commands that change network settings (bringing interfaces down, setting static IPs, enabling firewalls, or running DHCP/DNS servers) are marked as lab-only. Run those inside a VM or a throwaway environment.
- Only scan networks you own or have permission to test.

## Suggested path

| Week | Focus |
|------|-------|
| 1–2 | Models, IP/subnetting, exercises 1–3 in `computer-network.md` and `ip-addreses.md` |
| 3 | TCP/UDP, ports, MAC, ARP, exercises 4–7 |
| 4 | DNS, DHCP, NAT, firewalls, exercises 8–10 |
| 5+ | Two-VM labs, subnetting drills, packet captures |

## Tools used throughout

- `ip`, `ss`, `ip route`, `ip neigh`
- `ping`, `traceroute`, `tracepath`
- `dig`, `nslookup`, `host`
- `tcpdump`, `Wireshark`
- `nmap`
- `iptables` / `nftables` / `ufw`
- `dhclient`, `isc-dhcp-server`, `dnsmasq`

Read in the order that matches your curiosity — all files are written to stand alone when needed.