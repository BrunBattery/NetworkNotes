---
title: IP SLA & Tracking
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

- Allows monitoring of network performance and network availability testing with probes
- Track objects allow you to dynamically control things like static routes, FHRP priorities, etc
- Possibilities for object tracking include:
	- IP SLA instances
	- Interface status
	- Groups of objects
	
---

#### Useful show commands
- `show ip sla configuration` - Shows configuration for all IP SLAs set up on device
- `show ip sla statistics` - Shows statistics for all IP SLAs set up on device
- `show track` - Shows configuration and status of tracking object

---

#### Standard IP SLA w/ track for HSRP Config
```
! Tracking line protocols of WAN interfaces
track 11 interface GigabitEthernet8 line-protocol
track 12 interface FastEthernet0 line-protocol
! Tracking ip routing status of tunnel interfaces for backup route
track 21 interface Tunnel1 ip routing
track 22 interface Tunnel2 ip routing
! Track 23 will be up if both tunnels are not routing - set up for backup route to be installed
track 23 list boolean and
 object 21 not
 object 22 not
! Tracking for IP SLA 101
track 100 ip sla 101 reachability
 delay down 15 up 60
! Decrementing HSRP based on IP SLA and line protocols of WAN interfaces
int vlan 100
 standby 1 track 11 decrement 10
 standby 1 track 12 decrement 10
 standby 1 track 100 decrement 10
! Routing management traffic to secondary router if neither tunnel is routing
ip route 10.100.100.0 255.255.255.0 10.1.174.253 track 23
! Testing reachability over DMVPN overlay and starting SLA
ip sla 101
 icmp-echo 172.16.1.254 source-interface Tunnel1
 request-data-size 100
 frequency 5
ip sla schedule 101 life forever start-time now
```