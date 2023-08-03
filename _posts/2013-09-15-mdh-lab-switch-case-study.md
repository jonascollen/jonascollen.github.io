---
title: MDH Lab - Switch Case Study
date: 2013-09-15 17:05
comments: true
categories: [Switch]
---
Topologi
--------

![lab4-3clean](/assets/images/2013/09/lab4-3clean.png)

Objectives
----------

*   Plan and design the International Travel Agency switched network as shown in the diagram and described below.
*   Implement the design on the switches and router.
*   Verify that all configurations are operational and functioning according to the requirements.

Requirements
------------

You will configure a group of switches and a router for the International Travel Agency. The network includes two distribution switches, S1 and S3, and two one access layer switches, S2. External router R3 and S1 provide inter-VLAN routing. Design the addressing scheme using the address space 172.16.0.0/16 range. You can subnet it any way you want, although it is recommended to use /24 subnets for simplicity.

1.  Place all switches in the VTP domain CISCO. Make S1 the VTP server and all other switches VTP clients.
2.  On S1, create the VLANs shown in the VLAN table and assign the names given. For subnet planning, allocate a subnet for each VLAN.
3.  Configure S1 as the primary spanning-tree root bridge for all VLANs. Configure S3 as the backup root bridge for all VLANs.
4.  Configure Fa0/4 between S1 and S3 as a Layer 3 link and assign a subnet to it.
5.  Create a loopback interface on S1 and assign a subnet to it.
6.  Configure the Fa0/3 link between S1 and S3 as an ISL trunk.
7.  Statically configure all inter-switch links as trunks.
8.  Configure all other trunk links using 802.1Q.
9.  Bind together the links from S1 & S3 to the access-switch together in an EtherChannel.
10.  Enable PortFast on all access ports.
11.  On S2, place Fa0/15 through Fa0/17 in VLAN 10. Place Fa0/19 and Fa0/25 in VLAN 20. Place Fa0/21-22 in VLAN 30.
12.  Create an 802.1Q trunk link between R3 and S3. Only VLANs 10 and 40 to pass through the trunk.
13.  Configure R2 subinterfaces for VLANs 10 and 40.
14.  Create an SVI on S1 in VLANs 20, 30, and 40. Create an SVI on S3 in VLAN 10 and 30, an SVI on S2 in VLAN 40.
15.   Enable IP routing on S1 and S3. On R2 and S1, configure EIGRP for the whole major network (172.16.0.0/16) and disable automatic summarization.

**VLANs:**

*   Vlan 10 - Red
*   Vlan 20 - Blue
*   Vlan 30 - Orange
*   Vlan 40 - Green

Genomförande
------------

### Subnetting

Jag har som synes redan lagt in den subnetting jag gjorde i topologin men såhär ser den ut iaf: **172.16.0.0/16**

Vlan 10 - Red 172.16.10.0/24
 Vlan 20 - Blue 172.16.20.0/24
 Vlan 30 - Orange 172.16.30.0/24
 Vlan 40 - Green 172.16.40.0/24

**S1**

Lo0 - 172.16.1.1/24
Vlan 20 - 172.16.20.1/24
Vlan 30 - 172.16.30.1/24
Vlan 40 - 172.16.40.1/24
S1-S3 Link - 172.16.13.1/24

**S3**

Vlan 10 - 172.16.10.3/24
S1-S3 Link - 172.16.13.3/24

**S2**

Vlan 40 - 172.16.40.2/24

**R3**

Vlan 40 - 172.16.40.200/24
Vlan 10 - 172.16.10.200/24

Med den information vi fått ovan kan vi uppdatera vår topologi lite: ![](/assets/images/2013/09/lab4-32.png)

### Basic L2-konfig

S1 - Kom ihåg att S1 även ska vara Root-bridge för samtliga VLAN & VTP-server
```
Switch(config)#hostname S1
 S1(config)#line con 0
 S1(config-line)#logging sync
 S1(config-line)#!Trunk-links till S2
 S1(config-line)#int range fa0/1 - 2
 S1(config-if-range)#switchport trunk encaps dot1q
 S1(config-if-range)#switchport mode trunk
 S1(config-if-range)#description to S2
 S1(config-if-range)#channel-protocol lacp
 S1(config-if-range)#channel-group 1 mode active
 Creating a port-channel interface Port-channel 1
S1(config-if-range)#!L3-link till S3
 S1(config-if-range)#inte fa0/4
 % Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
 S1(config-if)#no switchport
 S1(config-if)#ip add 172.16.13.1 255.255.255.0
 S1(config-if)#description to S3 L3-port
 S1(config-if)#!ISL-trunk till S3
 S1(config-if)#int fa0/3
 S1(config-if)#switchport trunk encapsulation isl
 S1(config-if)#switchport mode trunk
 S1(config-if)#description Trunklink to S3
 S1(config-if)#!VTP
 S1(config-if)#exit
 S1(config)#vtp mode server
 Device mode already VTP SERVER.
 S1(config)#vtp domain CISCO
 Changing VTP domain name from NULL to CISCO
 S1(config)#
 \*Mar 1 00:14:20.226: %SW\_VLAN-6-VTP\_DOMAIN\_NAME\_CHG: VTP domain name changed to CISCO.
 S1(config)#!VLANs
 S1(config)#vlan 10
 S1(config-vlan)#name Red
 S1(config-vlan)#vlan 20
 S1(config-vlan)#name Blue
 S1(config-vlan)#vlan 30
 S1(config-vlan)#name Orange
 S1(config-vlan)#vlan 40
 S1(config-vlan)#name Green
 S1(config-vlan)#exit
 S1(config)#spanning-tree vlan 1,10,20,30,40 root primary
```
S3 - Ska även vara Secondary Root-bridge för samtliga vlan
```
Switch(config)#hostname S3
 S3(config)#line con 0
 S3(config-line)#logging sync
 S3(config-line)#!Trunk-links till S2
 S3(config-line)#int range fa0/1 - 2
 S3(config-if-range)#switchport trunk encaps dot1q
 S3(config-if-range)#switchport mode trunk
 S3(config-if-range)#channel-protocol lacp
 S3(config-if-range)#channel-group 1 mode active
 Creating a port-channel interface Port-channel 1
S3(config-if-range)#
 S3(config-if-range)#description to S2
 S3(config-if-range)#inte fa0/3
 % Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
 S3(config-if)#!ISL-trunk till S1
 S3(config-if)#switchport trunk encaps ISL
 S3(config-if)#switchport mode trunk
 S3(config-if)#description ISL-trunk to S1
 S3(config-if)#!L3-port till S1
 S3(config-if)#int fa0/4
 S3(config-if)#no switchport
 S3(config-if)#ip add 172.16.13.3 255.255.255.0
 S3(config-if)#description L3-link to S1
 S3(config-if)#exit
 S3(config)#vtp mode client
 Setting device to VTP CLIENT mode.
 S3(config)#vtp domain CISCO
 Domain name already set to CISCO.
 S3(config)#spanning-tree vlan 1,10,20,30,40 root secondary
```
S2
```
Switch(config)#hostname S2
 S2(config)#line con 0
 S2(config-line)#logging sync
 S2(config-line)#!Etherchannels till S1 & S3
 S2(config-line)#inte range fa0/1 - 2
 S2(config-if-range)#switchport mode trunk
 S2(config-if-range)#description to S1
 S2(config-if-range)#channel-protocol lacp
 S2(config-if-range)#channel-group 1 mode passive
 Creating a port-channel interface Port-channel 1
S2(config-if-range)#int range fa0/3 - 4
 S2(config-if-range)#switchport mode trunk
 S2(config-if-range)#description to S3
 S2(config-if-range)#channel-protocol lacp
 S2(config-if-range)#channel-group 2 mode passive
 Creating a port-channel interface Port-channel 2
S2(config-if-range)#exit
 S2(config)#!VTP
 S2(config)#vtp mode client
 Setting device to VTP CLIENT mode.
 S2(config)#vtp domain CISCO
 Domain name already set to CISCO.
S2(config)#!Host-interface
 S2(config)#int range fa0/15 - 17
 S2(config-if-range)#switchport mode access
 S2(config-if-range)#switchport access vlan 10
 S2(config-if-range)#spanning-tree portfast
 S2(config-if-range)#int range fa0/19 - 20
 S2(config-if-range)#switchport mode access
 S2(config-if-range)#switchport access vlan 20
 S2(config-if-range)#spanning-tree portfast
 S2(config-if-range)#int range fa0/21-22
 S2(config-if-range)#switchport mode access
 S2(config-if-range)#switchport access vlan 30
 S2(config-if-range)#spanning-tree portfast
 S2(config-if-range)end
```
### L3-Konfig

S1 - Kom ihåg att aktivera routing innan vi lägger in EIGRP-konfig
```
S1(config)#int lo0
 S1(config-if)#ip add 172.16.1.1 255.255.255.0
 S1(config-if)#!SVIs for VLANs
 S1(config-if)#int vlan 20
 S1(config-if)#ip add 172.16.20.1 255.255.255.0
 S1(config-if)#description Red
 S1(config-if)#int vlan 30
 S1(config-if)#ip add 172.16.30.1 255.255.255.0
 S1(config-if)#description Blue
 S1(config-if)#int vlan 40
 S1(config-if)#ip add 172.16.40.1 255.255.255.0
 S1(config-if)#description Green
 S1(config-if)#exit
 **S1(config)#ip routing**
 S1(config)#router eigrp 1
 S1(config-router)#network 172.16.0.0
 S1(config-router)#no auto
 S1(config-router)#no auto-summary
```
S3
```
S3(config)#int vlan 10
 S3(config-if)#ip add 172.16.10.3 255.255.255.0
 S3(config-if)#description Red
 S3(config-if)#int vlan 30
 S3(config-if)#ip add 172.16.30.3 255.255.255.0
 S3(config-if)#description Orange
 S3(config-if)#exit
 S3(config)#ip routing
S3(config-if)#!trunk to R3
S3(config-if)#int fa0/5
S3(config-if)#description trunk to R3
S3(config-if)#switchport mode trunk
S3(config-if)#switchport trunk allowed vlan 10,40
```
S2
```
S2(config)#int vlan 40 
S2(config-if)#ip add 172.16.40.2 255.255.255.0 
S2(config-if)#description Green
```
Då var all switch-konfig klar, endast routern kvar.. R3
```
Router(config)#hostname R3
 R3(config)#inte fa0/1
 R3(config-if)#description to S3-trunklink
 R3(config-if)#no shut
 R3(config-if)#inte fa0/1.10
 R3(config-subif)#encapsulation dot1q 10
 R3(config-subif)#ip add 172.16.10.200 255.255.255.0
 R3(config-subif)#inte fa0/1.40
 R3(config-subif)#encapsulation dot1q 40
 R3(config-subif)#ip add 172.16.40.200 255.255.255.0
 R3(config-subif)#exit
R3(config)#router eigrp 1
 R3(config-router)#network 172.16.0.0
 R3(config-router)#no auto-summary
 R3(config-router)#end
```
### Verifiering - L3
```
S1#sh ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
 D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
 N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
 E1 - OSPF external type 1, E2 - OSPF external type 2
 i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
 ia - IS-IS inter area, \* - candidate default, U - per-user static route
 o - ODR, P - periodic downloaded static route
Gateway of last resort is not set
172.16.0.0/24 is subnetted, 6 subnets
C 172.16.40.0 is directly connected, Vlan40
C 172.16.30.0 is directly connected, Vlan30
C 172.16.20.0 is directly connected, Vlan20
C 172.16.13.0 is directly connected, FastEthernet0/4
D 172.16.10.0 \[90/28416\] via 172.16.40.200, 00:01:51, Vlan40
C 172.16.1.0 is directly connected, Loopback0
R3#sh ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
 D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
 N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
 E1 - OSPF external type 1, E2 - OSPF external type 2
 i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
 ia - IS-IS inter area, \* - candidate default, U - per-user static route
 o - ODR, P - periodic downloaded static route
Gateway of last resort is not set
172.16.0.0/24 is subnetted, 6 subnets
C 172.16.40.0 is directly connected, FastEthernet0/1.40
D 172.16.30.0 \[90/28416\] via 172.16.40.1, 00:02:35, FastEthernet0/1.40
D 172.16.20.0 \[90/28416\] via 172.16.40.1, 00:02:35, FastEthernet0/1.40
D 172.16.13.0 \[90/30720\] via 172.16.40.1, 00:02:35, FastEthernet0/1.40
C 172.16.10.0 is directly connected, FastEthernet0/1.10
D 172.16.1.0 \[90/156160\] via 172.16.40.1, 00:02:35, FastEthernet0/1.40
S1#ping 172.16.40.200
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/4/9 ms
S1#ping 172.16.10.200
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/8 ms
S1#ping 172.16.40.2
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/9 ms
R3#ping 172.16.40.2
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```
### Verifiering L2
```
S3#sh interface trunk
Port Mode Encapsulation Status Native vlan
**Fa0/3 on isl trunking 1**
**Fa0/5 on 802.1q trunking 1**
**Po1 on 802.1q trunking 1**
Port Vlans allowed on trunk
Fa0/3 1-4094
**Fa0/5 10,40**
Po1 1-4094
S1#sh spanning-tree summary
Switch is in pvst mode
**Root bridge for: VLAN0001, VLAN0010, VLAN0020, VLAN0030, VLAN0040**
S2#sh etherchannel summary
Flags: D - down P - bundled in port-channel
 I - stand-alone s - suspended
 H - Hot-standby (LACP only)
 R - Layer3 S - Layer2
 U - in use f - failed to allocate aggregator
M - not in use, minimum links not met
 u - unsuitable for bundling
 w - waiting to be aggregated
 d - default port

Number of channel-groups in use: 2
Number of aggregators: 2
Group Port-channel Protocol Ports
------+-------------+-----------+-----------------------------------------------
1 Po1(**SU)** LACP Fa0/1(P) Fa0/2(P) 
2 Po2(**SU**) LACP Fa0/3(P) Fa0/4(P)
```
Härligt! Stötte på lite problem under labben då det visade sig att interfacet jag tänkte använda mellan Switch & Router inte var directly connected. Det var inga problem att sätta upp trunkingen etc men trafiken fastnade i någon dold switch eller dylikt. Från början var det tänkt att Routern skulle vara ansluten till S2 men det fanns tyvärr inget interface att använda där som fungerade. Fick istället göra om ritningen lite och använda länken mellan R3-S3 men det fungerade ju precis lika bra efter lite mindre modifieringar. :) Kul labb!
