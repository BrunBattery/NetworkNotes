---
title: DMVPN
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

- Used for large-scale, scalable deployment of mGRE tunnels in a hub-and-spoke topology
- Can establish connectivity through NAT, DHCP
- Utilizes **NHRP** (Next Hop Resolution Protocol)
	- **NHS** (next-hop server) is statically defined on the spoke routers
	- Each spoke router then dynamically registers with the hub(s) and forms tunnel to hub(s)
	- Hub will populate NHRP cache with spoke information and allow communication between all nodes
	- In phase 2, 3, hub will assist with spoke-to-spoke tunnels by sending resolution, redirect NHRP messages to spokes
- Common to advertise summary route from hub, keep all spokes as stubs
- Front door VRFs used to bypass recursive routing issues with overlay

---

#### DMVPN Phases
- **Phase 1**
	- All traffic between spokes in this design must transit the hub, no spoke-spoke tunnels are formed
- **Phase 2**
	- Allows spoke-to-spoke traffic by making all tunnels (including spokes) multipoint
	- This allows spokes to form tunnels directly to each other
	- Must not have routing protocol use next-hop-self or the tunnels will not form
	- Does ***NOT*** work with summarization - since the RIB will not have a route to the other spokes, all traffic will transit the hub
	- If using OSPF, network type must be broadcast or non-broadcast to prevent changing next-hops
		- In this scenario the spokes must be set at priority 0 to prevent them becoming DRs - if spokes become DRs the network will not function as they cannot multicast between each other
- **Phase 3**
	- Allows spoke-to-spoke traffic by using NHRP redirect and shortcut commands that allow NHRP to rewrite next hops in the RIB
	- Main benefit over phase 2 is allowing summarization from hubs to spokes - spokes will still be able to form tunnels to each other in phase 3 even with a default route from the hub
	
---

#### Useful show commands
- `show dmvpn [detail]` - shows tunnel status, peers, up/down time, etc
- `show ip nhrp [brief]` - shows NHRP cache, NHRP-specific details useful for phase 2, 3
- `show ip route next-hop-override` - shows NH overrides for DMVPN phase 3
- `show crypto isakmp sa; show crypto ike2 sa; show crypto ipsec sa` - for troubleshooting IPsec when used with DMVPN
	
---

#### Config

##### DMVPN Phase 1 Configuration (No IPsec)


**Hub Config**
```
! Essential
interface Tunnel100
ip address 192.168.100.11 255.255.255.0
ip nhrp map multicast dynamic
ip nhrp network-id 100
tunnel source GigabitEthernet0/1
tunnel mode gre multipoint

! Discretionary
bandwidth 4000
ip mtu 1400
ip tcp adjust-mss 1360
ip nhrp authentication NHRPAUTH
no ip split-horizon eigrp 100 ! necessary for EIGRP
tunnel key 100 ! necessary in multi-hub designs
```

**Spoke Config**
```
interface Tunnel100
ip address 192.168.100.31 255.255.255.0
ip nhrp network-id 100
ip nhrp nhs <TunnelIP> nbma <PublicIP> multicast
tunnel source GigabitEthernet0/1
tunnel destination <PublicIP>

! Discretionary
bandwidth 4000
ip mtu 1400
ip tcp adjust-mss 1360
ip nhrp authentication NHRPAUTH
tunnel key 100 ! necessary in multi-hub designs
```

##### DMVPN Phase 2 Configuration (No IPsec)

**Hub Config**
```
!same as Phase 1, but including:

no ip next-hop-self eigrp 100

!so that hub doesn't list itself as NH for spokes
```

**Spoke Config**
```
!same as Phase 1, but including:

tunnel mode gre multipoint

!so that spokes can form direct tunnels
```

##### DMVPN Phase 3 Configuration (No IPsec)

**Hub Config**
```
!same as Phase 1, but including:

ip nhrp redirect

!so that hub will forward redirect messages to spokes after initial traffic, 
allowing spoke-to-spoke tunnels to form
```

**Spoke Config**
```
!same as Phase 1, but including:

ip nhrp shortcut
tunnel mode gre multipoint

!to allow the spokes to install NHRP next hops into the routing table and form direct tunnels
```

##### DMVPN IKEv1 IPsec Configuration 

**VRFless**
```
crypto isakmp policy 10
 encr aes 128
 hash sha256
 authentication pre-share
 group 16
!
crypto isakmp key DMVPN_PSK address 0.0.0.0  
!
crypto ipsec transform-set ESP-AES-256-SHA-512 esp-aes 256 esp-sha512-hmac 
 mode transport
!
crypto ipsec profile DMVPN_PROFILE
 set transform-set ESP-AES-256-SHA-512
!
interface Tunnel0
 tunnel protection ipsec profile DMVPN_PROFILE
```

**FVRF**
```
!same as above, but instead of crypto isakmp key:
crypto keyring VRF_AWARE_PSK vrf UNDERLAY_TRANSPORT 
  pre-shared-key address 0.0.0.0 0.0.0.0 key DMVPN_PSK
```
##### DMVPN IKEv2 IPsec Configuration (See IPsec page for deep dive)

**VRFless**
```
crypto ikev2 keyring DMVPN-KEYRING 
peer ANY
address 0.0.0.0 0.0.0.0
pre-shared-key CISCO456
!
crypto ikev2 profile DMVPN-IKE
match identity remote address 0.0.0.0
authentication remote pre-share
authentication local pre-share
keyring local DMVPN-KEYRING
!
crypto ipsec transform-set SET1 esp-aes 256 esp-sha-hmac
mode transport
!
crypto ipsec profile DMVPN-IPSEC
set transform-set SET1
set ikev2-profile DMVPN-IKE
!
interface Tunnel0
tunnel protection ipsec profile DMVPN-IPSEC
```

**FVRF**
```
!Same as above, but under crypto ikev2 profile:
 match fvrf INET
```

##### See VRF page for FVRF config details