---
title: PBR
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

- Conditional forwarding of packets based on something other than destination IP
- Possible filters include source IP, source & destination IP, protocol type (TCP, UDP, ICMP)
- Downsides are operational complexity, increased load on router
- Can specify multiple next-hops - when done the first will be used if the next-hop is adjacent, otherwise it will use the second, third, etc
- If you want to use a non-adjacent route, you must use the `recursive` flag when configuring the route-map `set` statement
- Using the `default` flag will cause the next-hop value to be set only for routes not within the router's RIB
- Can also combine PBR with [[IP SLA & Tracking]] to ensure reachability of next-hops
- **Local PBR**
	- This is a PBR for traffic sourcing from the router (normally not subject to policy routing)
	- Configuration is the same as a typical PBR but applied with `ip local policy route-map <rm-name>`
	
---

#### Useful show/debug commands
- `show ip policy` - Show PBR details
- `show ip local policy` - Show local PBR details
- `debug ip policy` - Shows policy-based decisions in real time
	
---

#### Standard PBR Config

```PBR
ip access-list extended AL_PBR
 permit ip 10.1.1.0 0.0.0.255 10.5.5.0 0.0.0.255
!
route-map RM_PBR-TRANSIT permit 10
 match ip address ACL-PBR
 set ip [default] next-hop 10.23.1.3 <optional second IP>
!
interface GigabitEthernet0/1
 ip policy route-map PBR-TRANSIT
```

#### PBR with IP SLA & Tracking Config

```PBR_w/_ObjectTracking
ip sla 1
icmp-echo 10.3.3.2
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
!
ip access-list standard ACL
 permit ip 10.2.2.0/24 10.1.1.1/32
!
route-map PBR
 match ip address ACL
 set ip next-hop verify-availability 10.3.3.2 track 1
!
interface GigabitEthernet0/1
 ip policy route-map PBR-TRANSIT
```