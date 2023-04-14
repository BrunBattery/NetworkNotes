---
title: SNMP
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

- Stands for Simple Network Management Protocol
- Main versions in-use are v2c and v3
- v2c does not support encryption and uses a community string for authentication
- v3 supports both mode advanced authentication and data encryption
- SNMP traps are unreliable, SNMP informs are reliable and must be confirmed by receiving host
- Good article on [v3 SNMP](https://www.wiresandwi.fi/blog/cisco-switch-router-snmpv3-configuration-ios-ios-xe)
- **ifindex persist**
	- Used to prevent SNMP index IDs from changing when devices are rebooted or new line cards are attached
	- Configured with `snmp ifmib ifindex persist`
	
---

#### Useful show commands
- `show snmp host` - Displays the SNMP notifications sent as traps, the version of SNMP, and the host IP address of the notifications
- `show snmp group` - Displays SNMP group information (v3)
- `show snmp user` - Displays SNMP user information (v3)

---

#### Config

##### Standard SNMP v2c Config
```
snmp ifmib ifindex persist
snmp-server location <location> 
snmp-server contact <contact-details>
snmp-server community <community-string> [RO|RW] [ACL]
snmp-server enable traps
snmp-server host 10.1.100.100 [traps|informs] version 2c <community-string>
```

##### Standard SNMP v3 Config
```
snmp ifmib ifindex persist
snmp-server location <location> 
snmp-server contact <contact-details>
no snmp-server system-shutdown ! prevents device being shutdown via SNMP
snmp-server view <view-name> iso included ! iso included allows all values
snmp-server group <group-name> v3 priv read <view-name> write <view-name>
snmp-server user <user-name> <group-name> v3 auth sha <password> priv aes 256 <password>
snmp-server host 10.1.100.100 version 3 priv <user-name>
```