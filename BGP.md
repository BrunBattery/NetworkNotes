---
title: BGP
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

- AD 200 for iBGP, 20 for eBGP
- Forms connection over TCP port 179 - does not need direct connection
- Only advertises best routes, all others not advertised
- **Best-path Algorithm**
	1. Weight (Highest)
	2. Local Preference (Highest)
	3. Originated locally
	4. AS_PATH (Shortest)
	5. Origin code (IGP > EGP > ?)
	6. MED (Lowest)
	7. Path (eBGP > iBGP)
	8. IGP metric (Lowest)
	9. Route age (Oldest, only compared when routes from different eBGP peers)
	10. RID (Lowest)
	- **W**e **L**ove **O**ranges **AS** **O**ranges **M**ean **P**ure **R**efreshment
- **iBGP**
	- TTL of 255 by default
	- Does not change next-hop by default - can be changed with `neighbor <ip-address | peer-group-name> next-hop-self`
	- Uses split horizon^[BGP split horizon prevents loops in iBGP by disallowing advertisements received from one peer to be advertised on to another. This makes full mesh a requirement for iBGP unless using route reflectors and/or confederation] for loop prevention
- **eBGP**
	- TTL of 1 by default - can be increased with `neighbor <ip> ebgp-multihop 255`
	- Can also use `neighbor <ip> ttl-security hops <hop-count>`
	- Difference between above is that `eBGP-multihop` sets the *maximum range* allowed, whereas `TTL-security` sets the *exact range*
	- If peering with different update source but not through additional hops, can use `neighbor <ip> disable-connected-check`
	- Does change next-hop by default
- **Peer groups** 
	- Can be used when multiple neighbors have the same requirements
	- Use `neighbor <name> peer-group` to set up peer-group, then perform config under `neighbor <name> <command>`
	- Peer groups applied to neighbor with `neighbor <ip> peer-group <name>`
	- Can check with `show ip bgp peer-group`
	- Peer groups are natively set up on the back end to limit CPU usage - only purpose is simplification of config
- **ECMP**
	- Can enable load-balancing with `maximum paths <path#>` command
	- Only takes effect when many parameters on routes match - Weight, local preference, AS-PATH (both numbers and lengths), origin code, MED, IGP metric
- Exact match of prefix in route table required for advertisement - configuring null route to prefix common
- Public range for 2-byte AS: 1 - 64511, Private range: 64512 - 65535
	- Also 4-byte ASs available now due to exhaustion of 2-byte ASs
- Good idea to change update source to a loopback interface with `neighbor <ip> update-source <loopback>`
- `clear ip bgp * soft` useful to refresh route table without tearing down neighborships
- Selectively advertise default route with `neighbor <IP> default-originate [route-map <CONDITION>]`
- Advertise a different AS to a peer than `router bgp` AS with `neighbor <IP> local-as <OldAS>` - useful in migration
- `neighbor <IP> fall-over` can be used to immediately tear down session if IGP route to neighbor disappears

---

#### Features

##### Summarization
- Primarily done with the `aggregate address <prefix> <mask>` command
	- Only works for prefixes already in BGP table
	- Router performing summary installs null route into table for aggregate
	- Should be used with `as-set` suffix - this will include AS-PATH information and prevent loops/suboptimal routes
		- Inherits community values of any specific prefixes
		- For example, if one of four specific prefixes has a `no-export` community, summary with `as-set` will as well
		- These can be modified or stripped with `attribute-map` suffix
		- Can also use `advertise-map` suffix to select specific prefix which will determine attributes of summary
	- By default does not suppress specific prefixes - `summary-only` at the end of command will
	- Prefixes summarized in this way have *ATOMIC_AGGREGATE* attribute assigned
	- Also assigned *AGGREGATOR* attribute specifying AS number and RID of aggregator
	- Can selectively suppress specific prefixes with `suppress-map` - all prefixes in map will be suppressed:
```
ip prefix-list SUPPRESS_PREFIX 150.1.1.0/24
!
route-map SUPPRESS_MAP permit 10
 match ip address prefix-list SUPPRESS_PREFIX
!
router bgp 200	
 aggregate-address 150.1.0.0 mask 255.255.0.0 suppress-map SUPPRESS_MAP as-set
```
	- Can also use `unsuppress-map` combined with summary-only to advertise specific prefixes
	- Since this is per-neighbor it can be used for inbound path manipulation - you would advertise the same summary prefix to multiple neighbors and use an unsuppress map towards the preferred peer for inbound traffic:
```
ip prefix-list NET_1 permit 10.0.1.0/24 
!
route-map UNSUPPRESS_MAP permit 10
 match ip address prefix-list NET_1
!		
router bgp 200
 aggregate-address 10.0.0.0 255.255.252.0 summary-only as-set
 neighbor 155.1.37.7 unsuppress-map UNSUPPRESS_MAP
```
	- Alternatively, can advertise a summary prefix in IGP with the `network` command - commonly done with a static null route

##### Route Reflection
- Used to bypass full mesh requirement in iBGP
- Configured with `neighbor <ip> route-reflector-client` on route reflector
- Reflector must have full mesh if full reachability is required (unless using multiple reflectors)
- Abide by following 3 rules:
1.  Routes learned from eBGP peers can be sent to other eBGP peers, clients, and non-clients.
2.  Routes learned from client peers can be sent to eBGP peers, other client peers, and non-clients.
3.  Routes learned from non-client peers can be sent to eBGP peers, and client peers, _but not other non-clients_.
- Functions like DR/BDR in OSPF by default - does not insert itself into transit path
- `next-hop-self` command only affects eBGP routes when used with route reflectors

##### Confederation
- Used to reduce full-mesh BGP requirements by splitting one public AS into smaller sub-as
- Sub-as use eBGP advertisement rules between each other, removing full mesh requirement
- Notable exception to eBGP behavior is that *next-hop is not modified* between confederation peers
- Configured by using sub-as with `router bgp` command and using `bgp confederation identifier <as>` under router bgp
- Other sub-as defined with `bgp confederation peers <sub-as>` under router bgp
- Can be used in combination with route reflection

##### Communities
- Activated for transport to peers with `neighbor <ip> send-community`
- Made more readable with global config command `ip bgp-community new-format`
- Applied to prefixes with `set community <value1> <value2> ... <valueN>` in route-map
- To add communities without affecting existing, use `set community additive <value1> <value2> ... <valueN>`
- Matched with `community-lists` - like ACLs, both standard and expanded (extended) versions
	- Standard version uses 1-99, only permits or denies communities
```
ip community-list 1 permit 100:10 100:20
ip community-list 1 deny no-export
```
	- Standard version uses AND logic with multiple communities in one line, OR logic with multiple lines
	- Expanded version functions the same but adds **regular expression** functionality
- **Sample config of setting community:**
```
ip as-path access-list 1 permit 60$
!
route-map SET_COMMUNITY permit 10
 match as-path 1
 set community 100:200
!
route-map SET_COMMUNITY permit 100
!
router bgp 200
 neighbor 155.1.45.4 send-community
 neighbor 155.1.45.4 route-map SET_COMMUNITY out
```
- **Sample config of matching community:**
```
ip community-list standard 100:200 permit 100:200
!
route-map SET_LOCAL_PREFERENCE permit 10
 match community 100:200
 set local-preference 200
!
route-map SET_LOCAL_PREFERENCE permit 100
!
router bgp 100
 neighbor 155.1.13.3 route-map SET_LOCAL_PREFERENCE in
```
- **Well-known communities:**
	- **no-advertise**
		- Set with `set community no-advertise`
		- Signals to not advertise the prefix with this community to *any peer*
	- **no-export**
		- Set with `set community no-export`
		- Signals to not export this prefix from the AS - can be advertised *within AS only*
	- **local-as**
		- Set with `set community local-as`
		- Functions the same as `no-export` but also disallows advertisement *between confederation sub-as*
- Deleting communities done with a `community-list` referencing community to be deleted, then using `set comm-list <comlist> delete` under route-map

##### Next-hop modification
- Other than next-hop-self, you can also manually define next-hop with a route-map:
```
route-map SET_NEXT_HOP_FROM_R8 permit 10
set ip next-hop 155.1.58.8
!
router bgp 100
neighbor 155.1.58.8 route-map SET_NEXT_HOP_FROM_R8 in
```

##### BGP regexp
- Can be used to search for specifc AS-PATHs in table - for example: `show ip bgp regexp ^$`
- Above will show prefixes originated locally with the AS. Cheat sheet [here.](http://gponsolution.com/bgp-regular-expressions-cheat-sheet.html)
- Can be used to filter with ACLs with ip as-path: `ip as-path access-list 1 permit _54$`
- Then use `match as-path` under a route-map to apply modifications

##### BGP backdoor
- Created to prefer links via IGP over eBGP (raises AD of specific prefix to 200)
- Configured with `network <subnet> mask <netmask> backdoor` under router BGP

##### Conditional Advertising
- Allows advertising based on existence of another prefix in table
- Configured with `neighbor <IP> advertise-map MAP1 [non-exist|exist-map] MAP2`
- First map is prefix to advertise, second is prefix to monitor
- BGP checks on existence of MAP2 every 60 seconds

##### Conditional Route Injection
- Functions similar to unsuppress map in allowing specific prefix advertisement from aggregate
- Difference is it can be configured on routers not originating the aggregate route
- One map matches aggregate and router originating summary, other map matches specific prefix to advertise
- Configuration:
```
ip prefix-list INJECT_PREFIX permit 10.0.1.0/24
ip prefix-list AGGREGATE permit 10.0.0.0/22
ip prefix-list ROUTE_SOURCE permit 155.1.37.3/32
!
route-map INJECT_MAP permit 10
 set ip address prefix-list INJECT_PREFIX
 set origin igp
!
route-map EXIST_MAP permit 10
 match ip address prefix-list AGGREGATE
 match ip route-source prefix-list ROUTE_SOURCE
! 
router bgp 300
 bgp inject-map INJECT_MAP exist-map EXIST_MAP
```

##### Route Filtering
- Preferably done per-neighbor with route-maps, but can be applied directly with prefix list or ACL
- `neighbor 155.1.79.9 route-map FROM_R9 in` for a route-map, with a deny in the route-map
- `neighbor 192.10.1.254 prefix-list BLOCK_222 in` for prefix-list
- `neighbor 192.10.1.254 distribute-list BLOCK_222 in` can be used for ACL filtering, not recommended

##### Maximum prefix
- `maximum-prefix` can be used to limit allowed prefix # from a peer
- By default shuts down connection, can use `warning-only` or `restart <minutes>` prefixes if desired
- For example: `neighbor 155.1.108.10 maximum-prefix 20 80 restart 3`

##### BGP Dampening
- Can be used to prevent table instability from flapping routes
- Assigned to all prefixes with `bgp dampening [<Half_Life> <ReuseLimit> <SuppressLimit> <MaximumSuppressTime>]`
- Can also be assigned to specific prefixes with route-maps:
```
ip as-path access-list 100 permit _100$
!
route-map DAMPENING
 match as-path 100
 set dampening 4 750 2000 16
!
router bgp 200
 bgp dampening route-map DAMPENING
```

##### BGP Outbound Route Filtering
- Allows you to push a route filter to a remote neighbor
- Benefit is decrease in unnecessary route information as filtering occurs on neighbor before prefixes sent
- Both peers must have ORF capability enabled for it to function
- Verified locally with: `show ip bgp neighbors <ip>`, remotely with: `show ip bgp neighbors <ip> received prefix-filter`
```
ip prefix-list ORF deny 112.0.0.0/8
ip prefix-list ORF permit 0.0.0.0/0 le 32
!
router bgp 100
 neighbor 155.1.45.5 capability orf prefix-list both
 neighbor 155.1.45.5 prefix-list ORF in
```
- **See Redistribution page for details on that**

---

#### Useful debug/show commands
- `show ip bgp` - Displays BGP table and attributes for each route
- `show ip bgp summary` - Displays BGP neighbors, prefixes received from them, ASs, etc
- `show ip bgp neighbor <ip-address> [advertised-routes|received-routes]` - See routes sent, received from peer
- `show ip protocols` - Shows various information about active routing protocols
- `debug ip bgp` - Debug all BGP address families, information
- `debug ip bgp updates` - Debug BGP updates specifically
	
---

#### BGP Attributes

##### Weight
- Only used on the router where it is configured
- Used to affect outbound routing.
- Higher value is better - default is 0 (32768 if originated locally)
- Highest in BGP path selection algorithm
- Cisco proprietary
- Configuration:
```
route-map SET_WEIGHT
 match ip address|ip as-path ...
 set weight 100
!
router bgp 100
 neighbor <ip> route-map SET_WEIGHT in
 !
 neighbor <ip> weight <weight>
```

##### Local Preference
- Locally significant (to the AS), does not transit outside AS.
- Used to affect outbound routing.
- Higher value is better - defaults to 100
- Configuration:
```
route-map SET_LP
 match ip address|ip as-path ...
 set local-preference 1000
 !
router bgp 100
 neighbor <ip> route-map SET_LP in
 !
 bgp default local-preference <locpref>
```

##### AS-PATH
- Changed for each AS prefix transits through
- Used to affect inbound routing
- Can be used to influence path selection with as-path prepending
- Can be ignored with `bgp bestpath as-path ignore` - this is dangerous and not recommended
- Can be limited in length with `bgp maxas-limit <#>`
- Configuration:
```
route-map PREPEND
 match ip address|ip as-path ...
 set as-path prepend 100 100 100
	!
router bgp 100
 neighbor <ip> route-map PREPEND out
```

##### Origin Code
- Rarely used to influence path selection - inflexible
- IGP > EGP > Incomplete
- Could affect inbound or outbound routing
- Configuration:
```
route-map ORIGIN
 match ip address|ip as-path ...
 set origin [igp|egp|incomplete]
	
router bgp 100
 neighbor <ip> route-map ORIGIN out
```

##### MED
- By default only compared when originating from same AS
- Lowest is best - defaults to 0
- Used to affect inbound routing
- Can use `bgp always-compare-med` under router BGP to allow comparison of MED between different ASs
- Configuration:
```
route-map MED
 match ip address|ip as-path ...
 set metric 1000
	
router bgp 100
 neighbor <ip> route-map MED out
```

---

#### Standard BGP Config
```
router bgp 100
 bgp router-id 4.4.4.4
 ! iBGP Neighbor
 neighbor 150.1.1.1 remote-as 100
 neighbor 150.1.1.1 update-source loopback0
 neighbor 155.1.1.1 route-reflector-client
 neighbor 155.1.1.1 next-hop-self # will only affect eBGP routes for RR clients
 ! eBGP Neighbor
 neighbor 150.1.3.3 remote-as 200
 neighbor 150.1.3.3 update-source loopback0
 neighbor 150.1.3.3 ebgp-multihop
 neighbor 150.1.3.3 send-community both
 neighbor 150.1.3.3 route-map FROM_R3 in
 neighbor 150.1.3.3 route-map TO_R3 in
 !
 network 155.1.4.0 255.255.255.0 route-map SET_COMMUNITY_50
 network 150.1.4.0 255.255.255.0
!
ip prefix-list BGP_PREPEND_1 seq 5 permit 150.1.4.0/24
!
ip prefix-list DEFAULT seq 5 permit 0.0.0.0/0
!
route-map FROM_R3 deny 10
 match ip address prefix-list DEFAULT
!
route-map FROM_R3 permit 10000
!
route-map TO_R3 permit 10
 match ip address prefix-list BGP_PREPEND_1
 set as-path prepend 100 100 100 100
!
route-map TO_R3 permit 10000
!
route-map SET_COMMUNITY_50 permit 10
 set community 50
```
---