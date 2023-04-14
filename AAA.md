---
title: AAA
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

- Authentication, authorization and accounting
- Framework for controlling access to resources, monitoring usage, ensuring correct permissions
- **if-authenticated**
	- Used with the `aaa authorization` command as a suffix to allow access to higher privilege commands even if the AAA server is unreachable
	- without the `if-authenticated` suffix, the device will continually try to poll the AAA server for authorization and running any commands will be a painfully slow process
	
---

#### Useful Debugs
- `debug aaa authentication` - Debug all AAA auth
- `debug radius authentication` - Debug for radius auth
- `debug tacacs authentication` - Debug for tacacs auth
- `debug aaa protocol local` - Debug local authentication

---

#### Standard AAA Config
```
username <user> privilege 15 secret <pass> ! good idea to have local auth as backup
!
aaa new-model
aaa group server tacacs+ <group-name>
 server-private 10.20.5.35 single-connection key <key>
 ip vrf forwarding Mgmt-intf
 ip tacacs source-interface GigabitEthernet0
!
! alternatively can define the server like this
!
tacacs server <serv-name>
 address ipv4 10.20.5.35
 key <key>
aaa group server tacacs+ <group-name>
 server name <serv-name>
 ip tacacs source-interface GigabitEthernet0
!
aaa authentication login <list-name> group <group-name> local
aaa authorization exec <list-name> group <group-name> local if-authenticated
aaa authorization commands 15 <list-name> group <group-name> local if-authenticated
aaa accounting commands 1 <list-name> start-stop group <group-name>
aaa accounting commands 15 <list-name> start-stop group <group-name>
aaa session-id common
!
! using the list-name 'default' will apply the AAA methods to all lines without additional config
!
line vty 0 4
login authentication <list-name>
```