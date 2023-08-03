---
title: MDH Lab - Inter-VLAN routing
date: 2013-09-15 00:24
comments: true
categories: [Switch]
tags: [inter-vlan routing]
---
Topologi
--------

![lab4-2](/assets/images/2013/09/lab4-21.png)

Objective
---------

*   Configure inter-VLAN routing using an external router, also known as a router on a stick.

Background
----------

Inter-VLAN routing using an external router can be a cost-effective solution when it is necessary to segment a network into multiple broadcast domains. In this lab, you split an existing network into two separate VLANs on the access layer switches, and use an external router to route between the VLANs. An 802.1Q trunk connects the switch and the Fast Ethernet interface of the router for routing and management. Static routes are used between the gateway router and the ISP router. The switches are connected via an 802.1Q EtherChannel link.

Genomförande
------------

För omväxlingsskull kan vi väl börja med routrarna den här gången. ISP
```
Router(config)#hostname ISP
 ISP(config)#line con 0
 ISP(config-line)#logging synchro
 ISP(config-line)#exit
 ISP(config)#int lo0
 ISP(config-if)#ip add 200.200.200.1 255.255.255.0
 ISP(config-if)#int s0/0/0
 ISP(config-if)#ip add 192.168.1.2 255.255.255.0
 ISP(config-if)#no shut
 ISP(config-if)#exit
 ISP(config)#ip route 0.0.0.0 0.0.0.0 s0/0/0 192.168.1.1
```
Gateway
```
Router(config)#hostname Gateway
 Gateway(config)#line con 0
 Gateway(config-line)#logging sync
 Gateway(config-line)#int s0/0/0
 Gateway(config-if)#ip add 192.168.1.1 255.255.255.0
 Gateway(config-if)#no shut
 Gateway(config-if)#clock rate 256000
 Gateway(config-if)#exit
 Gateway(config)#ip route 0.0.0.0 0.0.0.0 s0/0/0 192.168.1.2
 Gateway(config)#do ping 200.200.200.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 200.200.200.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/8/8 ms
```
Vi kan väl vänta lite med att sätta upp Intervlan-routingen tills vi är klara med grundkonfigen så vi fortsätter med S1 och S3 istället. S1
```
Switch(config)#hostname S1
 S1(config)#line con 0
 S1(config-line)#logging sync
 S1(config-line)#exit
 S1(config)#int range fa0/3 - 4
 S1(config-if-range)#switchport trunk encaps dot1q
 S1(config-if-range)#switchport mode dynamic desirable
 S1(config-if-range)#description to S3
 S1(config-if-range)#channel-protocol pagp
 S1(config-if-range)#channel-group 1 mode desirable
 Creating a port-channel interface Port-channel 1
S1(config-if-range)#int vlan 1
 % Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
 S1(config-if)#ip add 172.16.1.2 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#exit
 S1(config)#ip default-gateway 172.16.1.1
 S1(config)#vlan 100,200
```
S3
```
Switch(config)#hostname S3
 S3(config)#line con 0
 S3(config-line)#logging sync
 S3(config-line)#exit
 S3(config)#int range fa0/3 - 4
 S3(config-if-range)#switchport trunk encaps dot1q
 S3(config-if-range)#switchport mode dynamic auto
 S3(config-if-range)#channel-protocol pagp
 S3(config-if-range)#channel-group 1 mode auto
 Creating a port-channel interface Port-channel 1
 S3(config-if-range)#description to S1
 S3(config-if-range)#int vlan 1
 % Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
 S3(config-if)#ip add 172.16.1.3 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#exit
 S3(config)#ip default-gateway 172.16.1.1
 S3(config)#vlan 100,200
```
Då var det dags att konfa upp Inter-VLAN routing. För att kunna använda oss av subinterface för varje vlan (1, 100, 200) behöver vi aktivera trunking mellan S1 & Gateway. Observera att vi ej kan använda DTP-negotiaton när det är en router vi ansluter till (inget stöd för DTP). S1
```
S1(config)#int fa0/5
 S1(config-if)#switchport trunk encapsulation dot1q
 S1(config-if)#switchport mode trunk
 S1(config-if)#description to Gateway
 S1(config-if)#spanning-tree portfast trunk
 %Warning: portfast should only be enabled on ports connected to a single
 host. Connecting hubs, concentrators, switches, bridges, etc... to this
 interface when portfast is enabled, can cause temporary bridging loops.
 Use with CAUTION
```
Gateway
```
Gateway(config)#int fa0/1
 Gateway(config-if)#description to S1
 Gateway(config-if)#no shut
 Gateway(config-if)#inte fa0/1.1
 Gateway(config-subif)#encapsulation dot1q 1 native
 Gateway(config-subif)#ip add 172.16.1.1 255.255.255.0
 Gateway(config-subif)#inte fa0/1.100
 Gateway(config-subif)#encapsulation dot1q 100
 Gateway(config-subif)#ip add 172.16.100.1 255.255.255.0
 Gateway(config-subif)#inte fa0/1.200
 Gateway(config-subif)#encapsulation dot1q 200
 Gateway(config-subif)#ip add 172.16.200.1 255.255.255.0
 Gateway(config-subif)#end
```
Klart! Vi kan verifera med att pinga mellan S3 & ISPs loopback t.ex.:
```
S3#ping 200.200.200.1
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 200.200.200.1, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 8/209/1015 ms
 ```
 