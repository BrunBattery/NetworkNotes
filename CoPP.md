---
title: CoPP
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

- Stands for **Control Plane Policing**
- Used to protect the control plane of network equipment from being saturated with less preferred traffic, DOS attempts, etc
- Policy-maps function in top-down fashion like ACLs
- `class-default` will match all traffic not matched earlier in a policy-map
- **Match any vs Match all**
	- Used in class-maps, behavior differenet when matching multiple access groups
	- Match any functions as boolean **OR**, match all functions as boolean **AND**
	
---

#### Useful show commands
- `show policy-map control-plane input` - Verify number of packets hitting COPP policy

---

#### Standard CoPP Config
```
! Identify traffic with ACLs
!
ip access-list extended COPP-ICMP-ACL-EXAMPLE
 permit udp any any range 33434 33463 ttl eq 1
 permit icmp any any unreachable
 permit icmp any any echo
 permit icmp any any echo-reply
 permit icmp any any ttl-exceeded
 exit
ip access-list extended COPP-MGMT-TRAFFIC-ACL-EXAMPLE
 permit udp any eq ntp any
 permit udp any any eq snmp
 permit tcp any any eq 22
 permit tcp any eq 22 any established
 permit tcp any any eq 23
 exit
ip access-list extended COPP-ROUTING-PROTOCOLS-ACL-EXAMPLE
 permit tcp any eq bgp any established
 permit eigrp any host 224.0.0.10
 permit ospf any host 224.0.0.5
 permit ospf any host 224.0.0.6
 permit pim any host 224.0.0.13
 permit igmp any any
!
! Create class maps to define traffic classes
!
class-map match-all COPP-ICMP-CLASSMAP-EXAMPLE
 match access-group name COPP-ICMP-ACL-EXAMPLE
class-map match-all COPP-MGMT-TRAFFIC-CLASSMAP-EXAMPLE
 match access-group name COPP-MGMT-TRAFFIC-ACL-EXAMPLE
class-map match-all COPP-ROUTING-PROTOCOLS-CLASSMAP-EXAMPLE
 match access-group name COPP-ROUTING-PROTOCOLS-ACL-EXAMPLE
!
! Create policy map to define service policy
!
policy-map COPP-POLICYMAP-EXAMPLE
 class COPP-MGMT-TRAFFIC-CLASSMAP-EXAMPLE
  police 32000 conform-action transmit exceed-action transmit violate-action transmit
 class COPP-ROUTING-PROTOCOLS-CLASSMAP-EXAMPLE
  police 34000 conform-action transmit exceed-action transmit violate-action transmit
 class COPP-ICMP-CLASSMAP-EXAMPLE
  police 8000 conform-action transmit exceed-action transmit violate-action drop
!
! Apply policy to control plane
!
control-plane
 service-policy input COPP-POLICYMAP-EXAMPLE
```