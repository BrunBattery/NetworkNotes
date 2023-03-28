---
title: IPSec
format:
  html:
    toc: true
    toc-depth: 2
    css: styles.css
editor_options:
  markdown:
    mode: gfm
---

## Security features of IPSec
- **Include:**
	- **Data origin authentication**
		- Verifying where packet came from
	- **Data integity**
		- Ensuring packet wasn't changed in transit
	- **Confidentiality**
		- Encryption, preventing packet from being intercepted and read
		- Prevented by limited allowed window
	- Confidentiality is the mainly thought-of core feature
- IPSec uses symmetric key encryption
- Point-to-point tunnels in almost all cases, GETVPN can be P2MP
- Tunnel keys dynamically negotiated through IKE
- IKEv2 is superior but not supported by all platforms

## IPsec Phases
- Two phases:
	- Phase one establishes ISAKMP SA, secure tunnel to negotiate phase two through
	- Phase two establishes IPsec SA, tunnel used to protect actual data traffic
	- **In Phase 1:**
		- Auth usually either PSK (Pre-shared key) or certificate based
		- DH^[Diffie Hellman - asymmetric key algorithm used for IPsec, SSL, SSH and others. Peers create public and private keys, exchange public key and generate symmetric key (shared secret) with peer's private key] used to exchange crypto keys
			- Higher DH number is more secure
		- Encryption algorithms used to protect traffic
			- Possibilities include DES, 3DES, AES-128, AES-256, etc
		- Authentication hashes used to ensure packet wasn't modified in transit
			- IKEv1 supports MD5, SHA-1
		- IKE parameters *must match* in order for phase 1 to complete successfully
		- Main mode and aggressive mode both options for completing this step
			- Main mode preferred as aggressive has security issues
	- **In Phase 2, following are negotiated:**
		- Security protocol
			- ESP (Encapsulating Security Payload) or AH (Authentication Header)
			- AH does not support encryption, rarely used
		- Encapsulation modes
			- Tunnel or transport
			- Tunnel adds new IP header, used for actually tunneling traffic
			- Transport mode used for host-to-host applications or combined with GRE which does the tunneling itself
		- Encryption (used for actual data traffic, not just tunnel)
			- DES, 3DES, AES-128, AES-256, etc
		- Authentication (hashing)
			- MD5, SHA, etc
		- Combination of the above called IPsec **Transform Set**
		- Only performed in QM (quick mode)
- UDP 500, 4500 are ports used to establish control plane
	- 4500 is only used with NAT-T^[NAT-Traversal add an additional UDP header which encapsulates the IPsec ESP header. Since this header is not encrypted it allows NAT without breaking packet integrity requirements of IPsec]
- IPsec dataplane uses ports 50 (ESP), 51 (AH) or 4500 with NAT-T

---

## Useful show/debug commands
- Phases:
	- **Phase 1:**
		- `show crypto isakmp sa` - should show **QM_IDLE** as state and status as **ACTIVE**
		- `debug crypto isakmp`, or `debug crypto condition peer ipv4 <IP>` to debug for one specific peer
	- **Phase 2:**
		- `show crypto ipsec sa [<peer IP>`]
		- `debug crypto ipsec`
		
---

## Crypto Profile IPSec Config w/ IKEv1 (with & without GRE)
```   
    interface Tunnel0  
	 ip address <ip & mask>
	 tunnel source <ip>
	 tunnel destination <ip>
      
    crypto isakmp policy 10
	 encryption aes 256
	 hash md5
	 authentication pre-share
	 group 14
	 lifetime 3600 
    !
    crypto isakmp key <PSK> address <remote outside interface IP with subnet mask>
    !
    crypto ipsec transform-set <ts-name> esp-3des esp-md5-hmac
	 mode transport
	 !mode tunnel if using VTI
	
	crypto ipsec profile <profile-name>
	 set transform-set <ts-name>
      
    interface Tunnel0
	 tunnel protection ipsec profile <profile-name>
	 ip mtu 1400
	 ip tcp adjust-mss 1360
	 !optionally can define tunnel mode as ipsec to bypass GRE and use VTI - 'tunnel mode ipsec ipv4'
	
	!note that routing through tunnel must exist for ipsec sa to form - routing not included in config
	!can do underlay and overlay routing protocols, static routing, front door vrfs, many options
```

## Crypto Profile IPSec Config w/ IKEv2 (with & without GRE)
```   
    interface Tunnel0  
	 ip address <ip & mask>
	 tunnel source <ip>
	 tunnel destination <ip>
      
	crypto ikev2 keyring <keyring-name>
	 peer ANY
	 address 0.0.0.0 0.0.0.0 ! any host for DMVPN, specific host for site-to-site
	 pre-shared-key <psk>
	!
	crypto ikev2 profile <ike-prof-name>
	 match identity remote address 0.0.0.0 ! any host for DMVPN, specific host for site-to-site
	 authentication remote pre-share
	 authentication local pre-share
	 keyring local <keyring-name>
	!
	crypto ipsec transform-set <ts-name> esp-aes 256 esp-sha-hmac
	 mode transport
	 !mode tunnel if using VTI
	!
	crypto ipsec profile <profile-name>
	 set transform-set <ts-name>
	 set ikev2-profile <ike-prof-name>
	!
    interface Tunnel0
	 tunnel protection ipsec profile <profile-name>
	 !optionally can define tunnel mode as ipsec to bypass GRE and use VTI - 'tunnel mode ipsec ipv4'
	
	!note that routing through tunnel must exist for ipsec sa to form - routing not included in config
	!can do underlay and overlay routing protocols, static routing, front door vrfs, many options
```

## Crypto Map IPSec Config w/ IKEv1 (with GRE)
```   
    interface Tunnel0  
    ip address <ip & mask>
    tunnel source <ip>
    tunnel destination <ip>
    ip mtu 1400
    ip tcp adjust-mss 1360
      
    crypto isakmp policy 10
	 encryption aes 256
	 hash md5
	 authentication pre-share
	 group 14
	 lifetime 180 
    !
    crypto isakmp key <PSK> address <remote outside interface IP with 32 bit subnet mask>
    !
    crypto ipsec transform-set <ts-name> esp-3des esp-md5-hmac
	    mode transport
    !
    access-list 120 permit gre host <local outside interface ip> host <remote outside interface IP>
      
    crypto map <cm-name> 10 ipsec-isakmp   
    set peer <ip>
    set transform-set <ts-name> 
    match address 120  
      
    interface Ethernet0/0   
    crypto map <cm-name>
```

##### See DMVPN page for details on DMVPN IPsec config