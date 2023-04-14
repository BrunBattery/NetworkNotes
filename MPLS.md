---
title: MPLS
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

- Uses shim headers inserted between IP and MAC headers to switch packets - labels
- Allows for BGP-free core in SP network where connectivity between BGP nodes is achieved by label switching
- MPLS labels switched hop by hop (think ethernet, not IP)
- **LDP** - Label Distribution Protocol used for MPLS operation
	- Uses all-routers multicast (**224.0.0.2**), TCP port **646** src and dst for Hellos, unicast to neighbors once TCP session established
	- UDP also used for initial discovery messages
	- LDP requires **CEF** in order to exchange labels
	- Only IGP-learned or local routes are labelled, BGP routes not assigned a label
- **PHP**
	- Penultimate Hop Popping - popping MPLS label on second to last router to avoid unnecessary overhead (processing shim header & IP)
	- In order to facilitate this, LDP routers advertise routes they own with an *implicit null/POP* label (label value of 3) that tells neighbors to pop LDP labels before sending for these routes
- **LIB** - Label Information Base - functions like L3 RIB
- **LFIB** - Label Forward Information Base - functions like L3 FIB
- **LSR** - Label Switched Router - any router that participates in MPLS
- **LSP** - Label Switched Path - path through LSRs labelled packet takes throughout MPLS domain
- **MP-BGP** (VPNv4) used with VRFs and LDP to form MPLS L3VPNs
- **RSVP** - Resource Reservation Protocol - used for traffic engineering
- **MPLS packet format**
	- Protocol identifier (**PID**) in layer-2 frame specifies that an MPLS packet will follow
	- Bottom-of-stack bit in shim header indicates whether additional labels follow
	- Nesting of LDP labels used for MPLS VPN, MPLS TE (Traffic Engineering)
- [**RDs and RTs explained here**](https://packetlife.net/blog/2013/jun/10/route-distinguishers-and-route-targets/)

---

#### MPLS L3VPN
- **High level steps**
	- Establish LSP between PEs using an IGP and LDP
	- Exchange routes with customer between PE-CE using IGP or BGP
	- Exchange customer routes between PEs with iBGP and an MPLS VPN label using VPNv4 AF (address family)
	- Label switch between the PE routers using LDP transport label
- **MPLS L3VPN details**
	- Uses MP-BGP between **PE** (provider edge) routers which tag packets with VPNv4 address
	- Uses VRFs to peer with customers over BGP or IGP
	- If using IGP instead of BGP for CE peering, must redistribute into BGP VPNv4 AF
	- VPNv4 address family in iBGP used to form neighborships and share VPNv4 routes
	- VPNv4 neighbor relationships ***must be formed*** from a loopback with a /32 mask
	- `next-hop-self` not required when running VPNv4 address family, does it for you automatically
	- Combination of RD (route distinguisher) and IPv4 address makes up VPNv4 address
	- VPNv4 address only used by MP-BGP, transparent to **P** routers in core
	- **PE** routers tag traffic with two MPLS labels, inner **VPN** label defining customer and outer **LDP** label used for label switching
	- **P** (provider) routers perform label switching along LSP but do not require MP-BGP, do not read **VPN** labels, only read outer **LDP** label
	- **CE** (customer edge) routers - do not see MPLS labels, connect to **PE** routers
	- Route reflectors function in the exact same way, still needed for iBGP peering using VPNv4 if not running a full mesh topology
	- In order to load balance when using RRs in L3VPN, both routes to the same destination must be using different RDs, otherwise only one best path is chosen
	- When running L3VPN via OSPF & CE router is using a VRF with OSPF, `capability vrf-lite` must be entered under router OSPF process or routes will not be installed by OSPF - this is because they have a special 'downward' bit flagged when transiting L3VPN that disallows them to be installed to vrf-aware OSPF to prevent loops in VPNv4 backbone
- **OSPF Domain-ID**
	- OSPF routes redistributed from VRF into VPNv4 BGP will show up as inter-area routes when learned by peers - by default BGP redistributed routes would show up as E2
	- However, the above is only true if your OSPF process id (`router ospf <x>`) matches or you manually set the domain-ID
	- Domain-ID set with `domain-id <x>` under router ospf
	- With domain-ID not matching, OSPF routes will show up as E2
	- Can be used for traffic engineering if CE is peering with multiple PEs, as OSPF will always prefer IA routes over E2 routes
- **OSPF Sham Link**
	- Used to allow OSPF routes across L3VPN to show up as intra-area instead of inter-area
	- Used in scenarios where CE has a backdoor route to another network connected to the MPLS & MPLS should be preferred
	- Configured by creating a /32 loopback route on both PEs, advertising this into BGP within the VRF (NOT VPNv4) and using the `area 1 sham-link <SRC><DST> cost X` command under OSPF
	- The loopbacks used for the sham link must not be advertised into the OSPF process
	
---

#### Useful show commands/debugs
- `show mpls interfaces` - shows interfaces participating in LDP
- `show mpls ldp neighbor` - shows LDP neighbors
- `show mpls ldp binding` - shows LDP label bindings to prefixes
- `show mpls forwarding-table` - shows LFIB
- `show bgp vpnv4 unicast all` - checking that VPNv4 routes being sent/received
- `debug mpls ldp transport events` - self-explanatory
- `debug bgp vpnv4 unicast updates` - useful for troubleshooting VPNv4 issues

---

#### Config

##### Standard MPLS Config
```
router ospf 1
 mpls ldp autoconfig
```

##### Extended MPLS Config
```
mpls ldp router-id Loopback0 force
!
int gi0/1
mpls ip
mpls ldp discovery transport-address interface
!
mpls ldp password required
mpls ldp neighbor 150.1.5.5 password CISCO
mpls ldp neighbor 150.1.6.6 password CISCO
!
access-list 10 permit 150.1.0.0 0.0.255.255
!
no mpls ldp advertise-labels
mpls ldp advertise-labels for 10
```

##### Standard L3VPN MPLS Config (PE config, OSPF to CE)

```
ip vrf VPN_A
 rd 1:1
 route-target export 1:1
 route-target import 1:1

interface Loopback0
 ip address 150.1.7.7 255.255.255.255
!
!
interface GigabitEthernet0/2
 ip vrf forwarding VPN_A
 ip address 155.1.79.7 255.255.255.0
!
router ospf 10 vrf VPN_A
 redistribute bgp 100 subnets
 network 0.0.0.0 255.255.255.255 area 0
!
router ospf 1
 mpls ldp autoconfig
 router-id 150.1.7.7
 network 0.0.0.0 255.255.255.255 area 0
!
router bgp 100
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 150.1.5.5 remote-as 100
 neighbor 150.1.5.5 update-source Loopback0
 !
 address-family vpnv4
  neighbor 150.1.5.5 activate
  neighbor 150.1.5.5 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf VPN_A
  redistribute ospf 10
 exit-address-family
```

##### L3VPN Internet Access Config
```
interface GigabitEthernet0/0
description INTERNET_EXAMPLE
 ip address 10.0.0.1 255.255.255.252
!
ip route vrf VPN_A 0.0.0.0 0.0.0.0 GigabitEthernet 0/0 10.0.0.2 global
ip route 0.0.0.0 0.0.0.0 GigabitEthernet 0/0 10.0.0.2
!
router bgp 100
address-family ipv4 vrf VPN_A
 default-information originate
 redistribute static
!
router ospf 10 vrf VPN_A
 default-information originate
!
interface GigabitEthernet0/0
 ip nat outside
!
interface GigabitEthernet0/1
 ip nat inside
!
interface GigabitEthernet0/2
 ip nat inside
!
ip access-list standard VPN_LOOPBACKS
 permit 150.1.0.0 0.0.255.255
!
ip nat inside source list VPN_LOOPBACKS interface GigabitEthernet0/0 vrf VPN_A overload
```

##### MPLS Config Explanation
- `mpls ip` - used to enable LDP on an interface or globally on the router
- `mpls ldp router-id Loopback0 force` - used to force the router-id used for LDP
- `router ospf 1; mpls ldp autoconfig` - used to automatically enable LDP on OSPF interfaces
- `mpls ldp discovery transport-address interface` - used to enable TCP connection over the physical interface IP instead of router-id
- `mpls ldp neighbor <IP> password <password>; mpls ldp password required` - configuring a password and making the password a requirement
- `no mpls ldp advertise-labels; mpls ldp advertise-labels for 10` - only allow labels to be generated for a specific ACL (10 in this case)