---
title: DHCP
format:
  html:
    toc: true
    toc-depth: 5
    css: styles.css
editor_options:
  markdown:
    mode: gfm
---

#### General Notes

- Used to assign IP addresses to hosts without requiring static assignments locally
- Can operate outside of subnet with `ip helper-address` on interface to forward DHCP requests to the server
	- This also exists in v6 as `ipv6 dhcp relay destination <v6-IP>`
- With IPv6, both **stateful** and **stateless** DHCP exist
	- **Stateful** advertises IPv6 addresses, prefixes, gateways etc like a typical DHCP server in v4
	- **Stateless** allows hosts to configure their own address, gateway etc locally with SLAAC and only advertises other information such as DNS, TFTP server, etc
	- **Stateles**s DHCP is configured with `ipv6 nd other-config-flag` under interface on segment that will advertise the information
- **DORA**
	- **Discover** - Broadcast by client discover DHCP server on segment using port **67**
	- **Offer** - Server responds offering IP, SN Mask, gateway, DNS using port **68**
	- **Request** - Client requests to use offered information
	- **Acknowledge** - Server acknowledges and agrees, records information
- **SARR - IPv6 DORA equivalent**
	- **Solicit** - Client sends v6 DHCP request to all-DHCPv6-server solicit node multicast address FF02::1:2
	- **Advertise** - Server responds with unicast ADVERTISE message to client
	- **Request** - Client requests to use offered information
	- **Reply** - Server acknowledges and agrees, records information (if stateful)
	
---

#### Useful show commands/debugs
- `show ip dhcp binding` - When using a local DHCP server on a router, see DHCP binding info
- `show ip dhcp conflict` - Show any IPs conflicting with DHCP
- `debug ip dhcp server packet` - See DHCP traffic in flow
- `show ipv6 dhcp binding` - Same but for v6
- `show ipv6 dhcp interface` - Show interfaces advertising v6 address information

---

#### Standard DHCP Config
**IPV4**
```
ip dhcp excluded-address 10.8.8.1 10.8.8.10
!
ip dhcp pool POOL-A
network 10.8.8.0 255.255.255.0
default-router 10.8.8.1
dns-server 192.168.1.1
```

**IPv6**
```
ipv6 dhcp pool DHCPV6POOL
address prefix 2001:DB8:A:A::/64
dns-server 2001:DB8:B:B::1
domain-name cisco.com
!
interface GigabitEthernet0/0
ipv6 dhcp server DHCPV6POOL ! required to advertise IPs in v6 unlike v4
```
