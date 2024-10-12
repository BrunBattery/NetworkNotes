---
title: Palo Alto Notes
format:
  html:
    toc: true
    toc-depth: 5
    css: styles.css
editor_options:
  markdown:
    mode: gfm
---

#### Links
- [PAN-OS packet flow sequence](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClVHCA0)
- [PAN-OS CLI Cheat Sheets](https://docs.paloaltonetworks.com/pan-os/10-2/pan-os-cli-quick-start/cli-cheat-sheets)
- [Generate API key](https://docs.paloaltonetworks.com/pan-os/10-2/pan-os-panorama-api/get-started-with-the-pan-os-xml-api/get-your-api-key)
- [Security policy useful filters](https://live.paloaltonetworks.com/t5/general-articles/tips-and-tricks-filtering-the-security-policy/ta-p/573184)
- [Newly added AD users don't appear on the firewall immediately](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClVtCAK)
- [Remove admin login sessions](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000Cm6RCAS)
- [Print device config in XML](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClHoCAK)
- [Migrating config between locations or devices via API](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClENCA0)
- [App-ID and application dependency](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClV0CAK)
- [Custom applications & app override](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClRoCAK)
- [Brute force prevention on GP portal](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClJ2CAK)
---

- **App-ID**
	- Should be used where possible over service filtering - App-ID is more reliable for filtering traffic as it does not rely on just what ports the traffic is riding on
	- Where possible, using 'application-default' under service is also ideal - this will allow only the ports required for this application
	- The first few packets will be permitted based on the 6-tuple key[^2] defined by the rule until the app can be identified
	- This further reinforces using correct service filtering as otherwise unintended traffic will be permitted (e.g. 'any' service would permit any destination ports until app was identified)

- **URL filtering**
	- Only works when used on web traffic (80 & 443)
	- Can work even without traffic being decrypted by inspecting the SNI[^3] field of the TLS handshake
	- URL filters do not work properly when URLs exist in multiple custom URL groups
		- When one URL matches multiple categories, this creates unpredictable results with URL filtering[^1]

- **Asymmetric routing**
	- By default, PAs will not allow asymmetric traffic between zones - if a PA doesn't get a SYN for the first packet of a TCP handshake, the traffic will be discarded
	- You can allow asymmetric traffic[^4], however this is done on a per-zone basis.

- **Proxy-ID**
	- Used in VPN configurations where one side is a policy-based VPN (defining interesting traffic, source/destination)
	- Palo Alto is strictly a route-based VPN and will by default advertise 0.0.0.0/0 as its interesting traffic - if the other side is policy-based, this won't match and the tunnel will fail to come up unless proxy-id is used

- **Hits on Rules but No Logs**
	- Few possible causes for this occurrence
	- First, improper log forwarding to Panorama if looking there or no logging on the rule in general
	- Second, rule is logging at end and, through gathering more information, the end of the session is marked as hitting another rule - there will still be a hit for the initial connection in this case but no log
	- Third, this could be a licensing issue with a VM firewall causing no logs to be generated

---
- **Useful commands**
	- `find command keyword <keyword>` - Allows searching CLI for useful commands
	- `show counters global filter yes packet-filter yes delta yes | match drop` - This gives you any drop counters that have incremented since the command was last run and matches the PCAP filter - useful for finding the reason for dropped packets in a PCAP

[^1]: [Attached article](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClzSCAS) validates this.
[^2]: The 6-tuple key consists of the following: source-address, destination-address, source-port, destination-port, protocol, and security-zone.
[^3]: The SNI field (server name indication) is transmitted in clear text during the TLS handshake - more detail on how PA inspects this located [here](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000PObsCAG&lang=en_US%E2%80%A9&refURL=http%3A%2F%2Fknowledgebase.paloaltonetworks.com%2FKCSArticleDetail).
[^4]: [Attached article](https://knowledgebase.paloaltonetworks.com/KCSArticleDetail?id=kA10g000000ClG2CAK) describes how to allow asymmetric traffic.