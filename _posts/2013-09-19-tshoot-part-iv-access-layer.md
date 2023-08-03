---
title: TSHOOT - Part IV, Access-layer
date: 2013-09-19 15:36
comments: true
categories: [Troubleshoot]
---
![tshoot-access](/assets/images/2013/09/tshoot-access.png) 
Vi fortsätter från [tidigare inlägg](http://www.jonascollen.se/posts/tshoot-part-iii-eigrp-dist-layer/) där vi avslutade med att konfa upp L3/Routing/Redist. mellan R4 & DSW1 & 2. Innan vi ger oss på att konfa NAT & DHCP så fixar vi klart vårat access-layer först. Är lite osäker på om det ens är möjligt att sätta upp HSRP över switch-modulen men det märker vi väl.. :)

### Layer 2

För att lägga till VLAN i NM-16ESW måste vi använda legacy-metoden "vlan database", observera även att vi måste använda "show vlan-switch". 

DSW1

```
DSW1#vlan database
 DSW1(vlan)#vlan 10
 VLAN 10 added:
 Name: VLAN0010
 DSW1(vlan)#vlan 20
 VLAN 20 added:
 Name: VLAN0020
 DSW1(vlan)#vlan 200
 VLAN 200 added:
 Name: VLAN0200
 DSW1(vlan)#exit
 DSW1(config)#int range fa1/9 - 10
 DSW1(config-if-range)#switchport mode trunk
 DSW1(config-if-range)#switchport trunk encapsulation dot1q
 DSW1(config-if-range)#description to ASW1
 DSW1(config-if-range)#channel-group 1 mode on
 Creating a port-channel interface Port-channel1
 DSW1(config-if-range)#
 DSW1(config)#int range fa1/5 - 6
 DSW1(config-if-range)#switchport trunk encapsulation dot1q
 DSW1(config-if-range)#switchport mode trunk
 DSW1(config-if-range)#descrip to ASW2
 DSW1(config-if-range)#channel-group 4 mode on
 Creating a port-channel interface Port-channel4
 DSW1(config-if-range)#
```
DSW2
```
DSW2#vlan database
 DSW2(vlan)#vlan 10
 VLAN 10 added:
 Name: VLAN0010
 DSW2(vlan)#vlan 20
 VLAN 20 added:
 Name: VLAN0020
 DSW2(vlan)#vlan 200
 VLAN 200 added:
 Name: VLAN0200
 DSW2(vlan)#exi
 APPLY completed.
 Exiting....
DSW2(config)#int range fa1/9 - 10
 DSW2(config-if-range)#switchport trunk encaps dot1q
 DSW2(config-if-range)#switchport mode trunk
 DSW2(config-if-range)#desc to ASW2
 DSW2(config-if-range)#channel-group 1 mode on
 Creating a port-channel interface Port-channel1
DSW2(config-if-range)#int range fa1/5 - 6
 DSW2(config-if-range)#switchport trunk encaps dot1q
 DSW2(config-if-range)#switchport mode trunk
 DSW2(config-if-range)#desc to ASW1
 DSW2(config-if-range)#channel-group 5 mode on
 Creating a port-channel interface Port-channel5
```
ASW1
```
ASW1#vlan database
 ASW1(vlan)#vlan 10
 VLAN 10 added:
 Name: VLAN0010
 ASW1(vlan)#vlan 20
 VLAN 20 added:
 Name: VLAN0020
 ASW1(vlan)#vlan 200
 VLAN 200 added:
 Name: VLAN0200
 ASW1(vlan)#exit
 APPLY completed.
 Exiting....
ASW1(config)#no ip routing
 ASW1(config)#int range fa1/9 - 10
 ASW1(config-if-range)#switchport trunk encaps dot1q
 ASW1(config-if-range)#switchport mode trunk
 ASW1(config-if-range)#desc to DSW1
 ASW1(config-if-range)#channel-group 1 mode on
 Creating a port-channel interface Port-channel1
ASW1(config)#inte range fa1/5 - 6
 ASW1(config-if-range)#switchport trunk encaps dot1q
 ASW1(config-if-range)#switchport mode trunk
 ASW1(config-if-range)#desc to DSW2
 ASW1(config-if-range)#channel-group 5 mode on
 Creating a port-channel interface Port-channel5
```
ASW2
```
ASW2#vlan database
 ASW2(vlan)#vlan 10
 VLAN 10 added:
 Name: VLAN0010
 ASW2(vlan)#vlan 20
 VLAN 20 added:
 Name: VLAN0020
 ASW2(vlan)#vlan 200
 VLAN 200 added:
 Name: VLAN0200
 ASW2(vlan)#exit
 APPLY completed.
 Exiting....
ASW2(config)#no ip routing
 ASW2(config)#int range fa1/9 - 10
 ASW2(config-if-range)#switchport trunk encap dot1q
 ASW2(config-if-range)#switchport mode trunk
 ASW2(config-if-range)#desc to DSW2
 ASW2(config-if-range)#channel-group 1 mode on
 Creating a port-channel interface Port-channel1
ASW2(config-if-range)#int range fa1/5 - 6
 ASW2(config-if-range)#switchport trunk encap dot1q
 ASW2(config-if-range)#switchport mode trunk
 ASW2(config-if-range)#desc to DSW1
 ASW2(config-if-range)#channel-group 4 mode on
 Creating a port-channel interface Port-channel4
```
### Access-ports

ASW1
```
ASW1(config)#int range fa1/0 - 4
 ASW1(config-if-range)#switchport mode access
 ASW1(config-if-range)#switchport access vlan 10
 ASW1(config-if-range)#description Vlan10
```
ASW2
```
ASW2(config)#int range fa1/0 - 4
 ASW2(config-if-range)#switchport mode access
 ASW2(config-if-range)#switchport access vlan 20
 ASW2(config-if-range)#descrip Vlan20
```
### Management-Vlan

DSW1
```
DSW1(config)#interface vlan 200
 DSW1(config-if)#ip add 192.168.1.129 255.255.255.248
 DSW1(config-if)#desc Management
```
DSW2

DSW2(config)#interface vlan 200
 DSW2(config-if)#ip add 192.168.1.130 255.255.255.248
 DSW2(config-if)#desc Management

ASW1
```
ASW1(config)#int vlan 1
 ASW1(config-if)#shut
 ASW1(config-if)#int vlan 200
 ASW1(config-if)#ip add 192.168.1.131 255.255.255.248
 ASW1(config-if)#desc Management
```
ASW2
```
ASW2(config-if-range)#ex
 ASW2(config)#int vlan 1
 ASW2(config-if)#shut
 ASW2(config-if)#inte vlan 200
 ASW2(config-if)#ip add 192.168.1.132 255.255.255.248
 ASW2(config-if)#desc Management
```
### MLS Routing

DSW1
```
DSW1(config)#interface vlan 10
 DSW1(config-if)#ip add 10.2.1.1 255.255.255.0
 DSW1(config-if)#desc Vlan 10, Users
 DSW1(config-if)#interface vlan 20
 DSW1(config-if)#ip add 10.2.2.2 255.255.255.0
 DSW1(config-if)#desc Vlan 20, Servers
 DSW1(config-if)#exit
 DSW1(config)#router eigrp 10
 DSW1(config-router)#network 10.2.1.0 0.0.0.255
 DSW1(config-router)#network 10.2.2.0 0.0.0.255
 DSW1(config-router)#exit
```
DLSW2
```
DSW2(config)#interface vlan 10
 DSW2(config-if)#ip add 10.2.1.2 255.255.255.0
 DSW2(config-if)#desc Vlan 10, Users
 DSW2(config-if)#interface vlan 20
 DSW2(config-if)#ip add 10.2.2.1 255.255.255.0
 DSW2(config-if)#desc Vlan 20, Servers
 DSW2(config-if)#exit
 DSW2(config)#router eigrp 10
 DSW2(config-router)#network 10.2.1.0 0.0.0.255
 DSW2(config-router)#network 10.2.2.0 0.0.0.255
 DSW2(config-router)#end
```
### HSRP

Endast Vlan10 använde HSRP och enligt spec. ska DSW1 vara Active, vi sätter därför prion över 100 som annars är default. DSW1
```
DSW1(config)#interface vlan 10
 DSW1(config-if)#standby 1 ip 10.2.1.254
 DSW1(config-if)#standby 1 preempt
 DSW1(config-if)#standby 1 priority 150
```
DSW2
```
DSW2(config)#interface vlan 10
 DSW2(config-if)#standby 1 ip 10.2.1.254
 DSW2(config-if)#standby 1 preempt
 DSW2(config-if)#standby 1 priority 100
DSW1#sh standby brief
 P indicates configured to preempt.
 |
 Interface Grp Prio P State Active Standby Virtual IP
 Vl10 1 150 P Active local 10.2.1.2 10.2.1.254
```
Tyvärr verkade det som jag stött på en bugg här. Klienter kan pinga DSW1's riktiga IP-adress 10.2.1.1, men inte den virtuella gatewayen.
```
HostA#ping 10.2.1.254
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 10.2.1.254, timeout is 2 seconds:
 .....
 Success rate is 0 percent (0/5)

 HostA#ping 10.2.1.1
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 10.2.1.1, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 12/43/68 ms
```
Slår vi över till DSW2 istället så fungerar det utan problem märkligt nog.
```
DSW2(config)#int vlan 10
 DSW2(config-if)#standby 1 prio
 DSW2(config-if)#standby 1 priority 160
 DSW2(config-if)#end
 DSW2#
 *Mar 1 03:58:36.031: %HSRP-5-STATECHANGE: Vlan10 Grp 1 state Standby -> Active
 *Mar 1 03:58:36.355: %SYS-5-CONFIG_I: Configured from console by console
HostA#ping 10.2.1.254
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 10.2.1.254, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 8/16/32 ms
 HostA#ping 10.2.1.1
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 10.2.1.1, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 12/24/48 ms
 HostA#ping 10.2.1.2
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 10.2.1.2, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 12/228/1032 ms
```
Skumt! Verkar inte vara den enda som har samma problem för den delen men har inte lyckats hitta någon vettig lösning på problemet. Har tillsvidare lämnat DSW2 som Active istället, huvudsaken är väl att vi har en "virtuell gateway" till vår host. Vi har nu åtminstone fullt flöde genom hela nätet!

```
HostA#ping 10.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 100/136/176 ms
HostA#traceroute 10.1.1.1
Type escape sequence to abort.
Tracing the route to 10.1.1.1
1 10.2.1.2 28 msec 56 msec 24 msec
 2 10.1.4.9 64 msec 28 msec 32 msec
 3 10.1.1.9 72 msec 108 msec 68 msec
 4 10.1.1.5 180 msec 120 msec 56 msec
 5 10.1.1.1 152 msec 200 msec 176 msec
```

Då återstår endast NAT på R1 & DHCP-servern på R4. :)