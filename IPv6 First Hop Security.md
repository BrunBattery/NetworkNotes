---
title: IPv6 First Hop Security
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
- Features to increase IPv6 layer 2 security, typically applied on switches
- No broadcasts, but certain packets almost always flooded in IPv6 networks
	- FF02::1 (All IPv6 hosts)
	- FF02::2 (All IPv6 routers)
- First-hop security features assist in preventing exploits around common multicast addresses used by NDP, DHCPv6

---

#### Features
- **RA Guard**
	- Analyzes RAs and filters traffic from unauthorized devices
	- Does not rely on binding table
	- Configured with `ipv6 nd raguard attach-policy [policy-name]` under interface
- **DHCP v6 Guard**
	- Basically DHCP snooping in IPv6
	- Does not rely on binding table
	- Configured with `ipv6 dhcp guard attach-policy [policy-name]` under interface
- **IPv6 Snooping**
	- Required for following features that rely on the IPv6 binding table
	- **Glean** populates bind table without verifying messages
	- **Inspect** gleans addresses and validates messages
	- **Guard** gleans *and* inspects messages, drops RA and DHCP messages by default
	- Configured like:
```
ipv6 snooping policy <name>
 security-level [glean|inspect|guard]
!
int gi0/1
 ipv6 snooping attach-policy <name>
```
- **ND inspection**
	- Inspects neighbor discovery messages and drops messages from hosts that already exist in the bind table but on different hardware addresses, preventing address spoofing
	- Relies on IPv6 snooping
	- Configured with `ipv6 nd inspection` under interfaces
- **Source/Prefix Guard**
	- Blocks traffic that does not come from an IPv6 address already known in the binding table
	- Relies on IPv6 snooping
	- Configured with `ipv6 source-guard` under interfaces
	
---

#### Useful show commands
- `show ipv6 nd raguard policy` - Shows RA Guard details
- `show ipv6 dhcp guard policy` - Shows DHCP v6 details
- `show ipv6 neighbors binding` - Shows IPv6 bind table

---

#### Config

##### Standard RA Guard Config
```RA_Guard
ipv6 nd raguard policy <policy-name>
 device-role host
!
int gi0/0
 ipv6 nd raguard attach-policy <policy-name>
!
!This will block all RAs on this port
```

##### Standard DHCP v6 Guard Config
```DHCP_v6_Guard
ipv6 dhcp guard policy DHCP_SERVER
 device-role server
ipv6 dhcp guard policy DHCP_CLIENT
 device-role client
!
interface GigabitEthernet 0/1
 ipv6 dhcp guard attach-policy DHCP_SERVER
interface range GigabitEthernet 0/2 - 3
 ipv6 dhcp guard attach-policy DHCP_CLIENT
!
!This will only allow DHCP offers from the server ports
```
