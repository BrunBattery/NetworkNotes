---
title: Syslog
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

- Used to manage system logs and alerts
- Can be stored locally, sent to vty, console lines, or forwarded to an external syslog server
- **Logging levels**
	- For the logging trap command, these are the levels & their associated numbers:

		Emergency: **0**  
		Alert: **1**  
		Critical: **2**  
		Error: **3**  
		Warning: **4**  
		Notice: **5**  
		Informational: **6**  
		Debug: **7**
		
---

#### Useful show commands
- `show logging` - Shows information on local and external syslog

---

##### Standard Syslog Config
```
logging buffered 90000 - ! Increase local logging buffer size
logging <logging-server>
service timestamps debug datetime localtime show-timezone msec
service timestamps log datetime localtime show-timezone msec
logging trap warning ! includes this level of message and all lower levels (warning to emergency)
logging source-interface loopback0
exit
!
show logging
```