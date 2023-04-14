---
title: VRF
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

- Allows separation of routing tables into distinct virtual tables
- Can be applied to interfaces, routing protocols, used for advanced techniques like MPLS L3VPNs
- **Front door VRFs (FVRF)**
	- Used to prevent recursive routing over tunnels without requiring separate routing protocols for under & overlay
	- Underlay networking kept in VRF table, overlay can freely form associations in global table or another VRF
	- Very useful in DMVPN environments
	- Great article [here](https://networkingwithfish.com/tunnels-and-the-use-of-front-door-vrfs/)

---

#### Useful show/debug commands
- `show ip vrf <name> [detail]` - Verify existing VRFs & their status
- `show ip vrf interfaces <name>` Show interfaces participating in VRF table
- `show ip route vrf <vrf-name>` - VRF routing table
- In enable, `routing-context vrf <name>` - Puts you in config mode with all typical commands applying to that VRF, for example `show ip route` would show the table for the selected VRF
- Plenty more, as tons of commands have VRF flags

---

#### Standard VRF-lite config

```VRF-lite
vrf definition <vrf-name>
 address-family ipv4

interface gi0/0
 vrf forwarding <vrf-name>
 ip address !must be reassigned after attaching interface to VRF
!
router eigrp <name>
 address-family ipv4 unicast vrf <vrf-name> autonomous-system <as>
  network 0.0.0.0
!
router ospf <as> vrf <vrf-name>
 network 0.0.0.0
!
router bgp 100
 neighbor 150.1.5.5 remote-as 100
 !
 address-family ipv4 vrf <vrf-name>
  neighbor 150.1.5.5 activate
```

#### Standard Front-door VRF config (FVRF)

```FVRF
vrf definition <vrf-name>
 address-family ipv4
!
int gi0/0
 description WAN INTERFACE
 vrf forwarding <vrf-name>
!
int tu0
 tunnel vrf <vrf-name>
!
!if running IKEv1:
!
crypto keyring VRF_AWARE_PSK vrf INET
!
!if running IKEv2:
!
crypto ikev2 profile IKE-PROFILE
match fvrf INET
```