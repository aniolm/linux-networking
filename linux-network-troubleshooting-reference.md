# Linux Network Troubleshooting & Analysis — Command Reference

A practical reference for diagnosing Linux networking problems from Layer 1 through Layer 7.

---

## 1. Interface State

### List interfaces

```bash
ip link
ip -br link
```

Useful for quickly seeing interface state:

```bash
ip -br link
```

Example:

```text
lo               UNKNOWN        127.0.0.1/8
enp0s31f6         UP             ...
```

### Show detailed interface information

```bash
ip -d link show
ip -d link show dev eth0
```

`-d` displays additional information such as VLAN, bridge, VXLAN, bonding, etc.

### Bring an interface up/down

```bash
sudo ip link set dev eth0 up
sudo ip link set dev eth0 down
```

### Inspect kernel interface information

```bash
cat /sys/class/net/eth0/address
cat /sys/class/net/eth0/operstate
cat /sys/class/net/eth0/carrier
cat /sys/class/net/eth0/mtu
```

---

# 2. IP Addresses

### List addresses

```bash
ip addr
ip -br addr
ip addr show dev eth0
```

### Add an address

```bash
sudo ip addr add 192.168.1.10/24 dev eth0
```

### Remove an address

```bash
sudo ip addr del 192.168.1.10/24 dev eth0
```

### Flush addresses

Be careful — this removes configured addresses.

```bash
sudo ip addr flush dev eth0
```

### IPv6

```bash
ip -6 addr
ip -6 addr show dev eth0
```

---

# 3. MAC Addresses

```bash
ip link show dev eth0
cat /sys/class/net/eth0/address
```

Change a MAC temporarily:

```bash
sudo ip link set dev eth0 down
sudo ip link set dev eth0 address 02:11:22:33:44:55
sudo ip link set dev eth0 up
```

---

# 4. Ethernet / PHY Diagnostics

`ethtool` is one of the most useful tools for physical Ethernet troubleshooting.

### Link status

```bash
sudo ethtool eth0
```

Look for:

```text
Speed:
Duplex:
Auto-negotiation:
Link detected:
```

### Driver information

```bash
sudo ethtool -i eth0
```

### Statistics

```bash
sudo ethtool -S eth0
```

Useful for detecting:

- RX errors
- TX errors
- dropped packets
- CRC errors
- carrier errors
- FIFO errors
- PHY problems

### Interface counters

```bash
ip -s link show dev eth0
```

### Offloading features

```bash
sudo ethtool -k eth0
```

Temporarily disable GRO:

```bash
sudo ethtool -K eth0 gro off
```

Disable checksum offloading:

```bash
sudo ethtool -K eth0 tx off rx off
```

This can be useful when packet captures appear confusing.

### Ring buffers

```bash
sudo ethtool -g eth0
```

---

# 5. Routing

### Show routing table

```bash
ip route
```

IPv6:

```bash
ip -6 route
```

### Ask Linux how it will route a packet

This is one of the most useful networking commands:

```bash
ip route get 8.8.8.8
```

Example:

```text
8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.100
```

It tells you:

- outgoing interface
- gateway
- source address
- selected route

### Add a route

```bash
sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0
```

### Delete a route

```bash
sudo ip route del 192.168.2.0/24
```

### Default route

```bash
ip route | grep default
```

Example:

```text
default via 10.42.0.1 dev eth0 proto dhcp metric 100
```

### Route metrics

When multiple routes match, Linux uses the route with the longest prefix first. If routes have the same prefix length, the metric can influence which route is preferred.

Example:

```text
default via 10.42.0.1 dev eth0   metric 100
default via 10.42.0.1 dev eth0.1 metric 400
```

The metric 100 route is preferred over metric 400 when otherwise equivalent.

---

# 6. ARP / Neighbor Discovery

Modern Linux uses the neighbor table rather than the old `arp` command.

### Show neighbors

```bash
ip neigh
```

Example:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

### IPv6 neighbors

```bash
ip -6 neigh
```

### Show neighbors for an interface

```bash
ip neigh show dev eth0
```

### Delete a neighbor entry

```bash
sudo ip neigh del 192.168.1.1 dev eth0
```

### Trigger ARP manually

```bash
sudo arping -I eth0 192.168.1.1
```

Useful for determining:

- whether a host is reachable at L2
- which MAC responds to an IP
- duplicate IP addresses
- VLAN/L2 problems

---

# 7. ICMP / Ping

### Basic ping

```bash
ping 192.168.1.1
```

### Send a fixed number of packets

```bash
ping -c 4 192.168.1.1
```

`-c` = number of packets.

### Specify interface

```bash
ping -I eth0 192.168.1.1
```

### Specify source address

```bash
ping -I 192.168.1.100 192.168.1.1
```

### IPv6

```bash
ping6 2001:db8::1
```

or:

```bash
ping -6 2001:db8::1
```

---

# 8. Path Analysis

### traceroute

```bash
traceroute 8.8.8.8
```

TCP mode:

```bash
sudo traceroute -T -p 443 example.com
```

### tracepath

```bash
tracepath 8.8.8.8
```

Useful because it can detect Path MTU.

---

# 9. MTU

### Show MTU

```bash
ip link show dev eth0
```

### Change MTU temporarily

```bash
sudo ip link set dev eth0 mtu 1500
```

### Test packet size

```bash
ping -M do -s 1472 192.168.1.1
```

For IPv4:

```text
1472 payload + 28 bytes IP/ICMP headers = 1500
```

If this fails, try a smaller value:

```bash
ping -M do -s 1400 192.168.1.1
```

This is useful for detecting:

- MTU mismatches
- broken PMTU discovery
- tunnels
- VLAN/encapsulation issues

---

# 10. tcpdump — Packet Capture

If you remember only one packet-debugging command, remember:

```bash
sudo tcpdump -eni eth0
```

Options:

- `-e` — show Ethernet headers/MAC addresses
- `-n` — don't resolve names
- `-i` — interface

### Capture everything

```bash
sudo tcpdump -eni eth0
```

### Specific host

```bash
sudo tcpdump -eni eth0 host 192.168.1.10
```

### Source/destination

```bash
sudo tcpdump -eni eth0 src host 192.168.1.10
sudo tcpdump -eni eth0 dst host 192.168.1.10
```

### Specific port

```bash
sudo tcpdump -eni eth0 port 443
```

### TCP

```bash
sudo tcpdump -eni eth0 tcp
```

### UDP

```bash
sudo tcpdump -eni eth0 udp
```

### DHCP

```bash
sudo tcpdump -eni eth0 'udp port 67 or udp port 68'
```

### ARP

```bash
sudo tcpdump -eni eth0 arp
```

### ICMP

```bash
sudo tcpdump -eni eth0 icmp
```

### IPv6 Neighbor Discovery

```bash
sudo tcpdump -eni eth0 icmp6
```

### TCP SYN packets

```bash
sudo tcpdump -eni eth0 'tcp[tcpflags] & tcp-syn != 0'
```

### TCP SYN + ACK

```bash
sudo tcpdump -eni eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
```

### VLAN-tagged traffic

```bash
sudo tcpdump -eni eth0 vlan
```

Specific VLAN:

```bash
sudo tcpdump -eni eth0 'vlan 10'
```

### Save capture

```bash
sudo tcpdump -eni eth0 -w capture.pcap
```

Read it later:

```bash
tcpdump -nr capture.pcap
```

Verbose:

```bash
tcpdump -nnr capture.pcap
```

---

# 11. VLANs

### Show VLAN interfaces

```bash
ip -d link show type vlan
```

### Create VLAN interface

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
```

Bring it up:

```bash
sudo ip link set eth0.10 up
```

Assign an address:

```bash
sudo ip addr add 192.168.10.10/24 dev eth0.10
```

Delete VLAN:

```bash
sudo ip link delete eth0.10
```

### Inspect VLAN configuration

```bash
ip -d link show eth0.10
```

### Capture VLAN tags

```bash
sudo tcpdump -eni eth0 vlan
```

Important concept:

A Linux VLAN sub-interface such as:

```text
eth0.10
```

causes Linux to transmit frames with an 802.1Q VLAN tag of ID 10 on the parent interface `eth0`.

---

# 12. Linux Bridges

### List bridges

```bash
bridge link
bridge vlan show
```

### Show bridge interfaces

```bash
ip link show type bridge
```

### Show bridge ports

```bash
bridge link
```

### Show forwarding database

```bash
bridge fdb show
```

This shows learned MAC addresses.

### Show VLAN configuration

```bash
bridge vlan show
```

Example:

```text
port    vlan ids
eth0    1 PVID Egress Untagged
        10
```

Important fields:

- `PVID` — VLAN assigned to untagged ingress traffic
- `Egress Untagged` — frames leave without a VLAN tag
- VLAN without `Egress Untagged` — normally leaves tagged

---

# 13. DHCP

### Capture DHCP

```bash
sudo tcpdump -eni eth0 'udp port 67 or udp port 68'
```

Typical sequence:

```text
DHCPDISCOVER
DHCPOFFER
DHCPREQUEST
DHCPACK
```

### DHCP client — NetworkManager

```bash
nmcli device status
nmcli connection show
```

### systemd-networkd

```bash
networkctl status eth0
```

### Check DHCP-related logs

```bash
journalctl -u NetworkManager
journalctl -u systemd-networkd
```

---

# 14. DNS

### Query DNS

```bash
dig example.com
```

### Query a specific DNS server

```bash
dig @8.8.8.8 example.com
```

### Short answer

```bash
dig +short example.com
```

### Reverse lookup

```bash
dig -x 192.168.1.1
```

### systemd-resolved

```bash
resolvectl status
resolvectl query example.com
```

### Inspect resolver configuration

```bash
cat /etc/resolv.conf
```

---

# 15. TCP / UDP Sockets

`ss` is the modern replacement for most `netstat` use cases.

### Listening sockets

```bash
ss -lnt
```

With process information:

```bash
sudo ss -lntup
```

Options:

- `-l` — listening
- `-n` — numeric
- `-t` — TCP
- `-u` — UDP
- `-p` — process

### All TCP connections

```bash
ss -nt
```

### All sockets

```bash
ss -an
```

### Filter by port

```bash
ss -lntp 'sport = :443'
```

### TCP states

```bash
ss -tan
```

Look for:

```text
LISTEN
SYN-SENT
SYN-RECV
ESTABLISHED
FIN-WAIT-1
FIN-WAIT-2
TIME-WAIT
CLOSE-WAIT
```

---

# 16. Test TCP/UDP Connectivity

### netcat

```bash
nc -vz 192.168.1.10 443
```

Verbose TCP connection test.

### Listen

```bash
nc -l 12345
```

Connect from another machine:

```bash
nc 192.168.1.10 12345
```

Useful for quickly determining whether a TCP port is reachable.

---

# 17. HTTP / HTTPS

### Test HTTP

```bash
curl -v http://192.168.1.10
```

### Test HTTPS

```bash
curl -vk https://192.168.1.10
```

`-v` shows connection details.

`-k` disables certificate verification and is useful for testing with self-signed certificates.

### Show only headers

```bash
curl -I https://example.com
```

### Specify interface

```bash
curl --interface eth0 https://example.com
```

---

# 18. Firewall — nftables

### Show complete ruleset

```bash
sudo nft list ruleset
```

### List tables

```bash
sudo nft list tables
```

### List a table

```bash
sudo nft list table inet filter
```

### Monitor ruleset changes

```bash
sudo nft monitor
```

When debugging connectivity, always check whether the packet is being dropped by the firewall.

---

# 19. iptables

Some systems still use iptables compatibility layers.

```bash
sudo iptables -L -n -v
```

NAT:

```bash
sudo iptables -t nat -L -n -v
```

Forwarding:

```bash
sudo iptables -L FORWARD -n -v
```

Check whether iptables is actually using nftables underneath:

```bash
iptables --version
```

---

# 20. Conntrack

Connection tracking is particularly important when debugging NAT/firewall behavior.

### List connections

```bash
sudo conntrack -L
```

### Count entries

```bash
sudo conntrack -C
```

### Monitor new connection events

```bash
sudo conntrack -E
```

### Filter by source

```bash
sudo conntrack -L --src 192.168.1.100
```

Typical states include:

```text
UNREPLIED
ESTABLISHED
TIME_WAIT
CLOSE
```

For NAT troubleshooting, compare:

```text
packet -> conntrack -> NAT -> routing -> firewall
```

---

# 21. IP Forwarding

### Check IPv4 forwarding

```bash
sysctl net.ipv4.ip_forward
```

Enable temporarily:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### IPv6 forwarding

```bash
sysctl net.ipv6.conf.all.forwarding
```

### Reverse Path Filtering

```bash
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.default.rp_filter
```

`rp_filter` can cause surprising behavior on systems with:

- multiple interfaces
- asymmetric routing
- policy routing
- VLANs
- multiple gateways

---

# 22. Network Statistics

### Interface statistics

```bash
ip -s link
```

### Kernel network statistics

```bash
nstat
```

Useful examples:

```bash
nstat -az
```

Look for counters related to:

- TCP retransmissions
- failed connections
- ICMP errors
- IP forwarding
- packet drops

---

# 23. Traffic Control

### Show qdiscs

```bash
tc qdisc show
```

### Show qdisc for an interface

```bash
tc qdisc show dev eth0
```

### Show traffic classes

```bash
tc class show dev eth0
```

### Show filters

```bash
tc filter show dev eth0
```

Useful when investigating:

- packet scheduling
- bandwidth limits
- shaping
- QoS
- packet loss
- latency

---

# 24. Network Namespaces

### List namespaces

```bash
ip netns list
```

### Execute command inside namespace

```bash
sudo ip netns exec ns1 ip addr
```

### Show namespace routing table

```bash
sudo ip netns exec ns1 ip route
```

### Capture packets in namespace

```bash
sudo ip netns exec ns1 tcpdump -eni eth0
```

For containers, namespaces are often the reason an interface or route appears to be "missing".

---

# 25. Monitor Network Changes in Real Time

### Monitor address/link changes

```bash
ip monitor
```

### Link changes only

```bash
ip monitor link
```

### Address changes

```bash
ip monitor address
```

### Route changes

```bash
ip monitor route
```

### Neighbor changes

```bash
ip monitor neigh
```

This is extremely useful when a network configuration is being modified by:

- NetworkManager
- systemd-networkd
- DHCP
- container runtimes
- systemd services
- hotplug scripts

---

# 26. Kernel and System Logs

### Kernel messages

```bash
dmesg
```

### Follow kernel messages

```bash
dmesg -w
```

### Kernel journal

```bash
journalctl -k
```

### Follow kernel journal

```bash
journalctl -kf
```

### Search network-related messages

```bash
dmesg | grep -i -E 'net|eth|link|vlan|bridge|phy'
```

---

# 27. PCI Network Hardware

### List PCI devices

```bash
lspci
```

### Find Ethernet controllers

```bash
lspci | grep -i ethernet
```

### Detailed information

```bash
lspci -nnk
```

Look for:

```text
Kernel driver in use:
Kernel modules:
```

---

# 28. USB Network Hardware

```bash
lsusb
```

Detailed USB tree:

```bash
lsusb -t
```

Useful for USB Ethernet adapters and USB Wi-Fi devices.

---

# 29. Find Which Process Owns a Socket

### lsof

```bash
sudo lsof -i
```

Specific port:

```bash
sudo lsof -i :443
```

TCP:

```bash
sudo lsof -iTCP:443
```

UDP:

```bash
sudo lsof -iUDP:53
```

`ss -p` is usually preferable, but `lsof` can be useful when you need broader process/file information.

---

# 30. IPv6

### Addresses

```bash
ip -6 addr
```

### Routes

```bash
ip -6 route
```

### Neighbors

```bash
ip -6 neigh
```

### Neighbor Discovery traffic

```bash
sudo tcpdump -eni eth0 icmp6
```

Useful ICMPv6 message types include:

- Router Solicitation
- Router Advertisement
- Neighbor Solicitation
- Neighbor Advertisement

### Ping IPv6

```bash
ping -6 2001:db8::1
```

For link-local addresses, specify the interface:

```bash
ping -6 fe80::1%eth0
```

---

# 31. Multicast / Broadcast

### Capture broadcast traffic

```bash
sudo tcpdump -eni eth0 ether broadcast
```

### Capture multicast

```bash
sudo tcpdump -eni eth0 ether multicast
```

### IPv4 multicast

```bash
ip maddr
```

### IPv6 multicast

```bash
ip -6 maddr
```

Useful protocols include:

- mDNS
- SSDP
- DHCP
- ARP
- IPv6 Neighbor Discovery
- service discovery protocols

---

# 32. LLDP

If available:

```bash
lldpctl
```

or:

```bash
sudo lldpcli show neighbors
```

LLDP can reveal:

- switch identity
- switch port
- VLAN information
- link capabilities

This is especially useful when you don't know which switch port your device is connected to.

---

# 33. Wi-Fi

### Show Wi-Fi interfaces

```bash
iw dev
```

### Link information

```bash
iw dev wlan0 link
```

### Scan

```bash
sudo iw dev wlan0 scan
```

### NetworkManager Wi-Fi

```bash
nmcli device wifi list
```

### Signal and connection status

```bash
nmcli device status
nmcli device show wlan0
```

---

# 34. iperf3 — Throughput Testing

On server:

```bash
iperf3 -s
```

On client:

```bash
iperf3 -c 192.168.1.10
```

Reverse direction:

```bash
iperf3 -c 192.168.1.10 -R
```

UDP:

```bash
iperf3 -c 192.168.1.10 -u
```

Useful for distinguishing:

- link problems
- packet loss
- bandwidth limitations
- TCP performance problems
- asymmetric performance

---

# 35. NetworkManager

### Device status

```bash
nmcli device status
```

### Connections

```bash
nmcli connection show
```

### Detailed connection

```bash
nmcli connection show "<connection-name>"
```

### Device details

```bash
nmcli device show eth0
```

### Activate connection

```bash
nmcli connection up "<connection-name>"
```

### Deactivate

```bash
nmcli connection down "<connection-name>"
```

### Monitor NetworkManager

```bash
journalctl -u NetworkManager -f
```

---

# 36. systemd-networkd

### Interface status

```bash
networkctl status eth0
```

### All links

```bash
networkctl list
```

### Follow logs

```bash
journalctl -u systemd-networkd -f
```

---

# 37. OpenWrt

### Network configuration

```bash
uci show network
```

### Network status

```bash
ubus call network.interface dump
```

### Interfaces

```bash
ip addr
ip route
```

### Firewall

```bash
nft list ruleset
```

### Logs

```bash
logread
```

### Process/service state

```bash
/etc/init.d/network status
/etc/init.d/firewall status
```

### Restart networking

```bash
/etc/init.d/network restart
```

Be careful when doing this over SSH.

---

# 38. A Practical Troubleshooting Workflow

When a host cannot communicate, don't immediately jump to the application.

Work from the bottom upward.

## Step 1 — Physical link

```bash
ip link show eth0
ethtool eth0
```

Check:

```text
LOWER_UP
Link detected: yes
Speed
Duplex
```

## Step 2 — Interface address

```bash
ip addr show dev eth0
```

Check:

- correct address
- correct subnet
- duplicate addresses
- unexpected secondary addresses

## Step 3 — Neighbor resolution

```bash
ip neigh
```

Then:

```bash
sudo arping -I eth0 <gateway>
```

If ARP fails, investigate L2 before routing/application layers.

## Step 4 — Routing

```bash
ip route
ip route get <destination>
```

Verify:

- correct interface
- correct source address
- correct gateway
- route metric
- policy routing

## Step 5 — Packet capture

```bash
sudo tcpdump -eni eth0
```

This tells you whether packets actually leave and arrive.

## Step 6 — Firewall

```bash
sudo nft list ruleset
sudo conntrack -L
```

## Step 7 — TCP socket

```bash
ss -lntup
```

Check whether the application is actually listening.

## Step 8 — Application

```bash
curl -v ...
nc -vz ...
```

---

# 39. VLAN Troubleshooting Workflow

For VLAN problems, inspect all layers.

### On Linux

```bash
ip -d link show
bridge vlan show
ip addr
ip route
```

### Capture VLAN tags

```bash
sudo tcpdump -eni eth0 vlan
```

### Verify VLAN interface

```bash
ip -d link show eth0.10
```

### Verify switch configuration

Check:

- trunk/tagged VLANs
- access/untagged VLAN
- PVID
- VLAN membership
- native/untagged VLAN
- MTU-related configuration

### Important distinction

A VLAN interface:

```text
eth0.10
```

is a Layer-3 interface associated with VLAN ID 10.

The parent:

```text
eth0
```

is the physical interface.

Linux adds/removes the 802.1Q tag as frames enter/leave the VLAN interface.

---

# 40. DHCP Troubleshooting Workflow

Start with:

```bash
sudo tcpdump -eni eth0 'udp port 67 or udp port 68'
```

Look for:

```text
DHCPDISCOVER
DHCPOFFER
DHCPREQUEST
DHCPACK
```

Interpretation:

### No DHCPDISCOVER

The client isn't transmitting DHCP.

Investigate:

```bash
ip link
ip addr
NetworkManager/systemd-networkd
```

### DISCOVER but no OFFER

Possible causes:

- VLAN mismatch
- switch configuration
- DHCP server unavailable
- firewall
- DHCP relay issue
- broadcast not reaching server

### OFFER but no REQUEST

Possible client-side problem.

### REQUEST but no ACK

Possible:

- DHCP server problem
- address conflict
- VLAN problem
- firewall
- DHCP server policy

---

# 41. TCP Connection Troubleshooting

For:

```text
client -> server:443
```

Start with:

```bash
ip route get <server>
```

Then:

```bash
sudo tcpdump -eni eth0 host <server>
```

Look for:

```text
SYN
SYN-ACK
ACK
```

### SYN leaves but no SYN-ACK

Investigate:

- routing
- firewall
- server availability
- return path
- VLAN/L2
- NAT

### SYN + SYN-ACK but connection fails

Investigate:

- client firewall
- TCP state
- MTU
- application behavior

### Connection established but application fails

Then move to:

```bash
curl -v ...
ss -ntp
journalctl ...
```

---

# 42. ARP Troubleshooting

For IPv4:

```bash
ip neigh
sudo arping -I eth0 <ip>
sudo tcpdump -eni eth0 arp
```

Typical exchange:

```text
Who has 192.168.1.1? Tell 192.168.1.100
192.168.1.1 is-at aa:bb:cc:dd:ee:ff
```

If the request is visible but no reply arrives, investigate:

- VLAN configuration
- switch port
- host firewall
- host interface state
- duplicate IP
- wrong subnet
- bridge configuration

If the reply arrives with an unexpected MAC, investigate:

- duplicate IP
- proxy ARP
- bridge
- virtualization
- stale neighbor entries

---

# 43. The Most Useful Commands to Memorize

If you work regularly with Linux networking, these are worth memorizing:

```bash
ip -br link
ip -br addr

ip route
ip route get <destination>

ip neigh
ip -s link show <interface>

ethtool <interface>
ethtool -S <interface>

ip -d link show
bridge vlan show
bridge fdb show

ss -lntup

sudo tcpdump -eni <interface>
sudo tcpdump -eni <interface> vlan
sudo tcpdump -eni <interface> 'udp port 67 or udp port 68'

sudo nft list ruleset
sudo conntrack -L

ip monitor all

nstat
journalctl -k
```

---

# 44. Layer-by-Layer Mental Model

When troubleshooting, think in layers:

```text
Application
    │
    ▼
TCP / UDP
    │
    ▼
IP routing
    │
    ▼
ARP / Neighbor Discovery
    │
    ▼
Ethernet / VLAN
    │
    ▼
Bridge / Switch
    │
    ▼
NIC / PHY
    │
    ▼
Cable / Radio
```

For example, if:

```bash
ping 192.168.1.1
```

fails, don't immediately assume routing is broken.

Check:

```bash
ip link
ip addr
ip neigh
ip route
tcpdump
```

in that order.

The key debugging question is:

> **At which layer does the packet stop?**

Once you identify that layer, the problem becomes much smaller.

---

# 45. Quick Diagnostic Cheat Sheet

| Problem | First commands |
|---|---|
| No link | `ip link`, `ethtool eth0` |
| Wrong IP | `ip addr` |
| Wrong route | `ip route`, `ip route get <dst>` |
| ARP failure | `ip neigh`, `arping`, `tcpdump arp` |
| VLAN issue | `ip -d link`, `bridge vlan show`, `tcpdump vlan` |
| DHCP failure | `tcpdump 'udp port 67 or udp port 68'` |
| DNS failure | `dig`, `resolvectl` |
| TCP port unreachable | `ss -lntup`, `nc -vz`, `tcpdump` |
| Firewall issue | `nft list ruleset`, `conntrack -L` |
| NAT issue | `conntrack -L`, `nft list ruleset` |
| Packet loss | `ip -s link`, `ethtool -S`, `nstat`, `iperf3` |
| MTU issue | `ip link`, `ping -M do`, `tracepath` |
| Bridge issue | `bridge link`, `bridge fdb show`, `bridge vlan show` |
| Namespace issue | `ip netns`, `nsenter` |
| Network config changing | `ip monitor all`, `journalctl` |
| Wi-Fi issue | `iw dev`, `iw dev wlan0 link`, `nmcli` |
| Physical NIC issue | `ethtool`, `ethtool -S`, `dmesg` |

---

## Principle

The most effective Linux networking troubleshooting approach is:

```text
1. Check the link
2. Check the address
3. Check the neighbor
4. Check the route
5. Capture the packets
6. Check the firewall
7. Check the socket
8. Check the application
```

Don't guess where the problem is.

**Observe the packet flow and determine exactly where it stops.**
