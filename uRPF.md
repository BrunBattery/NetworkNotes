---
title: uRPF
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
- Stands for Unicast Reverse Path Forwarding
- Used to eliminate spoofed packets on a network by validating the return network path of ingress traffic
- Requires CEF to be enabled on the device

---

#### uRPF Modes

- **Strict mode**
	- If the return path to the source IP of the traffic does not match the ingress interface, the traffic will be discarded
	- Configured with the `rx` flag on the `ip verify unicast source` command
- **Loose mode**
	- If a return path to the source IP of the traffic does not exist in the routing table, the traffic will be discarded
	- Configured with the `any` flag on the `ip verify unicast source` command
- **Allow-default**
	- Used to allow a default route to be used as a return path - by default this is not allowed with uRPF
	- Configured with the `allow-default` flag on the `ip verify unicast source` command
	
---

#### Useful debugs
- `show cef interface <interface-name>` - Will show whether CEF, uRPF is enabled
	
---

#### Standard uRPF Config
```
int gi0/0
ip verify unicast source reachable-via [rx | any] [allow-default]
```