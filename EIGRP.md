---
title: EIGRP
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

- AD 90 internal, 170 external
- Multicast **224.0.0.10** standard for hellos, port number **88**
- Uses **DUAL** (Diffusing Update Algorithm)
- **K values** (metric parameters) *must match* for neighbor relationship to form
	- K values set with the `metric weights` command - should be left default
	- By default uses bandwidth (slowest in path) & delay (cumulative) to compose metric
- Only advertises best routes, all others not advertised
- Hello and hold timers do not need to match for adjacency (unlike OSPF)
- Can manually define neighbors (like in BGP) to send hellos unicast instead of multicast
- **Successor route** - best path
- **Successor** - First next-hop along successor route
- Config can be performed classic (`route eigrp 100`) or named (`router eigrp <name>`) modes
	- Can convert to named mode with `eigrp upgrade-cli`, hitless conversion
- Only IGP that can do unequal cost load-balancing
	- All routes must meet feasibility condition
	- Candidacy for routes can be modified with `variance` command
- **Feasibility condition:**
	- Reported distance (**RD** - neighbor's distance to node) must be shorter than feasible distance (**FD** - distance locally) to prevent loops
	- If conditions passed, route is considered feasible successor and included in EIGRP topology table for fast failover in reconvergence event
- When using named-mode in IPv6, all interfaces automatically advertised into EIGRP when address-family initiated
	- Can be shut down individually or under `af-interface default` with `shutdown`, then `no shutdown` under interfaces you want advertised
	
---

#### Features

##### Authentication
- Supports both MD5 & SHA-256 auth (named) or just MD5 (classic)
- MD5 configuration done with key chain in global config
- Key-id numbers must match for successful auth
- Applied under physical interfaces in classic mode or af-interfaces in named mode
```
key chain MD5_KEYS
 key 1
  key-string MD5_PASS
!
router eigrp MULTI-AF
 !
 address-family ipv4 unicast autonomous-system 100
  !
  af-interface default
   authentication mode hmac-sha-256 SHA_DEFAULT
  exit-af-interface
  !
  af-interface GigabitEthernet1.58
   authentication mode md5
   authentication key-chain MD5_KEYS
  exit-af-interface
```

##### Summarization
- Possible anywhere in topology
- Configured with `summary-address <network> <mask>` under af-interface in named mode, under physical interface in classic mode
- Leak map can be used to leak longer mask prefixes included in summary
	- `summary-address 0.0.0.0 0.0.0.0 leak-map <route-map>`
- Null0 installed on router to match summary address to prevent loops
- Can define metric on summary-address with the `summary-metric` command under topology base

##### Route filtering
- Possible anywhere in topology
- Commonly performed with `distribute-list prefix <prefix-list> <in/out> <interface>`
- Can also use route-maps to match on tags, metric
	- For example: 
```
route-map FILTER_ON_TAGS deny 10
 match tag 4
!
route-map FILTER_ON_TAGS permit 20
!
router eigrp 100
 distribute-list route-map FILTER_ON_TAGS in
```

##### Split horizon
- Prevents advertisements of prefixes out same interface they're received
- Used for loop prevention (along with feasibility condition, router-id)
- Can turn off (useful for hub & spoke) with `no ip split-horizon eigrp <AS>`

##### Default routing
- Can be advertised with quad 0 summary route under interface
- Can also be redistributed from static route

##### Stub
- Prevents advertisment of anything other than connected routes *by default*
- Prevents site from being used for transit
- Breaks query domain
- Best used when sites have no downstream neighbors
- Can use leak-maps to advertise normally suppressed routes

##### Stub-site
- Allows advertisement of routes learned on LAN but not WAN interfaces
- Limits EIGRP query-domain and prevents site from being used for transit
- Enabled with `eigrp stub-site`
	- Mutually exclusive with `eigrp stub`
- Interfaces towards hub/WAN identified with `stub-site wan-interface` under af-interface

##### Offset-lists
- Can be used to increase metric for a specific route/interface
- Configured under topology base with:
	- `offset-list <access-list> in <metric> <interface>`

- **See Redistribution page for details on that**

---

#### Useful debug/show commands

- `show ip eigrp interface brief` - Displays interfaces participating in EIGRP, RTT, pending routes, etc
- `show ip eigrp neighbors` - Displays neighbors and neighbor states
- `show ip eigrp topology all-links` - Displays EIGRP topology table, successors, feasible successors
- `show ip protocols` - Shows various information about active routing protocols
- `debug eigrp packet` - Debugs basically everything EIGRP
- `access-list 111 permit eigrp any any`, followed by `debug ip packet detail 111` - for packet debugging

---

#### Standard EIGRP Config

```
router eigrp NAME
 !
 address-family ipv4 unicast autonomous-system 100
  !
  af-interface default
   passive-interface
  exit-af-interface
  !
  af-interface GigabitEthernet0/1
   no passive-interface
  exit-af-interface
  !
  topology base
  exit-af-topology
  network 10.1.1.2 255.255.255.255
  eigrp router-id 150.1.5.5
 exit-address-family
```