---
title: Netflow
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

- Used to sample and observe network flow information
- Typically exported to a netflow collector which will display the information graphically with reports
- **Flexible netflow**
	- Used to allow more customization of the traffic parameters to be captured
	- Consists of records, exporters, samplers and monitors
	- **Records**
		- Define what parameters will be captured by netflow
		- Can use pre-defined records or define your own
	- **Exporters**
		- Used to set up exportation of the netflow information to the flow collector
	- **Samplers**
		- Used to limit the number of monitored packets if desired for performance reasons
	- **Monitors**
		- Used to apply netflow records to desired interfaces
		- References specific flow record and flow exporter
		
---

#### Useful show commands
- `show ip cache flow` - Shows netflow information locally
- `show ip flow interface` - Shows interfaces participating in netflow
- `show ip flow export` - Shows information on exported flows
- `show flow record` - Displays basic information about each flow record (flexible netflow)
- `show flow monitor` - Displays basic information about each flow monitor (flexible netflow)
- `show flow exporter` - Displays basic information about each flow exporter (flexible netflow)

---

#### Config

##### Basic Netflow Configuration
```
int fa 0/0
 ip flow [ingress|egress]
 exit
ip flow-export source lo 0
ip flow-export version [5|9]
ip flow-export destination <collector-ip> 5000
```

##### Flexible Netflow Configuration

```
flow exporter EXPORTER
 destination <Collector-IP>
 export-protocol netflow-v9
 source Vlan100
 transport udp 2055
!
flow record RECORD
 match ipv4 tos
 match ipv4 protocol
 match ipv4 source address
 match ipv4 destination address
 match transport source-port
 match transport destination-port
 match interface input
 collect interface output
 collect counter bytes
 collect counter packets
!
flow monitor IPv4_NETFLOW
 record RECORD
 exporter EXPORTER
 cache timeout active 60
!
interface fa0/0
 ip flow monitor IPv4_NETFLOW input
```