---
title: Redistribution
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

- Routers doing mutual redistribution will not readvertise routes between protocols (Ex: EIGRP > OSPF > EIGRP)
- Routes will not be redistributed if they don't exist in the RIB
- Not possible to create loops with a single redistribution point
- EIGRP and OSPF will not accept routes advertised from their own router ID for loop prevention
- Tags can be used to prevent loops caused by advertising routes from one domain back into the same domain
- Modifying AD can prevent loops caused by higher AD protocols being redistributed into lower AD protocols and then preferred through that protocol
- Normally applied with route-map to advertise specific routes, set tags, etc

---

#### Redistribution per Protocol

- **EIGRP**
	- Redistribution into EIGRP can be done anywhere in the topology
	- Redistributed routes are marked external, AD of **170**
	- When redistributing into EIGRP, must assign metric either directly or with the `default-metric` command
- **OSPF**
	- Redistribution can happen anywhere aside from stubby, totally stubby networks - redistributing router becomes ASBR and has type 4 LSA advertised to any areas outside of its own local area to ensure reachability to border router
	- External routes will be LSA 5 in regular area, LSA 7 in NSSA and translated to type 5 at ABR
	- By default routes will be external type 2, e1 always preferred over e2 in OSPF algorithm
	- Must use the `subnets` flag or only classful routes will be redistributed
- **BGP**
	- Redistribution from BGP can be dangerous if using eBGP as IGPs will not be able to handle a full BGP table
	- Must manually match externals if redistributing OSPF, otherwise only internal routes redistributed
	
---

#### Useful show commands
- `ip route profile` - Shows changes in routing topology, useful to see route flapping
	
---

#### Standard Redistribution Config


**RouteMaps**
```
ip prefix-list PL_REDIST seq 5 permit 10.10.10.0/24 ge 28
!
route-map RM_REDIST permit 10  
match ip address prefix-list PL_REDIST 
set tag 100
!
redistribute connected route-map RM_REDIST
```

**EIGRP**
```
redistribute connected
redistribute static
redistribute ospf 1 metric 10000 100 255 1 1500
default-metric 10000 100 255 1 1500
```

**OSPF**
```
redistribute connected metric 10 subnets
redistribute static metric 200 metric-type 1 subnets
redistribute eigrp 1 metric 100 subnets
distance ospf external 171 ! for AD manipulation if required
```

**BGP**
```
redistribute connected
redistribute static
redistribute ospf 1 match external 1 external 2s
```