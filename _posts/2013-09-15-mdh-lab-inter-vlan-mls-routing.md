---
title: MDH Lab - Inter-VLAN MLS Routing
date: 2013-09-15 14:42
comments: true
categories: [Switch, MLS]
tags: [inter-vlan routing]
---
Topologi
--------

![lab4-2real](/assets/images/2013/09/lab4-2real.png)

Objective
---------

*   Route between VLANs using a 3560 switch with an internal route processor using Cisco Express Forwarding (CEF).

Background
----------

The current network equipment includes a 3560 distribution layer switch and two 2960 access layer switches. The network is segmented into three functional subnets using VLANs for better network management. The VLANs include Finance, Engineering, and a subnet for equipment management, which is the default management VLAN, VLAN 1. After VTP and trunking have been configured for the switches, switched virtual interfaces (SVI) are configured on the distribution layer switch to route between these VLANs, providing full connectivity to the internal network.

Genomförande
------------

Easy! Blir inte så mycket förklaringar här då all konfig är rätt självklar. Först fixar vi upp grundkonfigen: S1
```
Switch(config)#hostname S1
S1(config)#line con 0
S1(config-line)#logging sync
S1(config-line)#int range fa0/3 - 4
S1(config-if-range)#switchport trunk encaps dot1q
S1(config-if-range)#switchport mode trunk
S1(config-if-range)#channel-protocol pagp
S1(config-if-range)#channel-group 2 mode desirable 
Creating a port-channel interface Port-channel 2
S1(config-if-range)#int range fa0/1 - 2
S1(config-if-range)#switchport trunk encaps dot1q
S1(config-if-range)#switchport mode trunk
S1(config-if-range)#channel-protocol pagp
S1(config-if-range)#channel-group 1 mode desirable
Creating a port-channel interface Port-channel 1
S1(config-if-range)#exit
S1(config)#vtp mode server
Device mode already VTP SERVER.
S1(config)#vtp domain Cisco
Changing VTP domain name from NULL to Cisco
S1(config)#vlan 100
S1(config-vlan)#name Finance
S1(config-vlan)#vlan 200
S1(config-vlan)#name Engineering
S1(config-vlan)#exit
S1(config)#spanning-tree vlan 1,100,200 root primary 
S1(config)#
```
S3
```
Switch(config)#hostname S3
S3(config)#line con 0
S3(config-line)#logging sync
S3(config-line)#int range fa0/1 - 4
S3(config-if-range)#switchport trunk encaps dot1q
S3(config-if-range)#switchport mode trunk
S3(config-if-range)#int range fa0/1 - 2
S3(config-if-range)#channel-protocol pagp
S3(config-if-range)#channel-group 1 mode desirable 
Creating a port-channel interface Port-channel 1
3(config-if-range)#int range fa0/3 - 4
S3(config-if-range)#channel-protocol pagp
S3(config-if-range)#channel-group 2 mode auto
Creating a port-channel interface Port-channel 2
S3(config-if-range)#exit
S3(config)#vtp domain Cisco
Domain name already set to Cisco.
S3(config)#vtp mode client
Setting device to VTP CLIENT mode.
```
S2
```
Switch(config)#hostname S2
S2(config)#int range fa0/1 - 4
S2(config-if-range)#switchport mode trunk
S2(config-if-range)#int range fa0/1 - 2
S2(config-if-range)#channel-protocol pagp
S2(config-if-range)#channel-group 1 mode auto
Creating a port-channel interface Port-channel 1
S2(config-if-range)#int range fa0/3 - 4
S2(config-if-range)#channel-protocol pagp
S2(config-if-range)#channel-group 2 mode auto
Creating a port-channel interface Port-channel 2
S2(config-if-range)#exit
S2(config)#vtp mode client
Setting device to VTP CLIENT mode.
S2(config)#vtp domain Cisco
Domain name already set to Cisco.
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
1 Po1(SU) PAgP Fa0/1(P) Fa0/2(P) 
2 Po2(SU) PAgP Fa0/3(P) Fa0/4(P)
S3#sh etherchannel summary
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
1 Po1(SU) PAgP Fa0/1(P) Fa0/2(P) 
2 Po2(SU) PAgP Fa0/3(P) Fa0/4(P)
```
Allt ok så långt! Så då återstår det bara att konfa upp lite L3 SVI's, vilket är oerhört enkelt egentligen.
```
S1(config)#interface vlan 1
S1(config-if)#ip add 172.16.1.1 255.255.255.0
S1(config-if)#no shut
S1(config-if)#interface vlan 100
S1(config-if)#ip add 172.16.100.1 255.255.255.0
S1(config-if)#no shut
S1(config-if)#interface vlan 200
S1(config-if)#ip add 172.16.200.1 255.255.255.0
S1(config-if)#no shut
S1(config-if)#exit
```
Lätt att glömma är att vi även måste aktivera routing-funktionen i switchen! **S1(config)#ip routing** Vi har ju tyvärr ingen host att testa med nu men vi kan åtminstone dra en ping från S3 till något av S1's vlan.
```
S3(config)#int vlan 1
S3(config-if)#ip add 172.16.1.3 255.255.255.0
S3(config-if)#no shut
S3(config-if)#exit
S3(config)#ip default-gateway 172.16.1.1
S3(config)#do ping 172.16.200.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.200.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/203/1007 ms
```
Vackert. Om vi tar en titt i CEF-table för 172.16.1.3 kan vi se följande:
```
S1#sh ip cef 172.16.1.3 detail
 172.16.1.3/32, epoch 2, flags attached
 **Adj source: IP adj out of Vlan1, addr 172.16.1.3 038C1420**
 Dependent covered prefix type adjfib cover 172.16.1.0/24
 attached to Vlan1
```
Och switchen har även ett entry i adjacency-table med L2-information för nexthop (S3):
```
S1#sh adjacency detail
Protocol Interface Address
IP Vlan1 172.16.1.3(8)
0 packets, 0 bytes
epoch 0
sourced in sev-epoch 0
Encap length 14
0014A8899CC00024C33F9EC00800
L2 destination address byte offset 0
L2 destination address byte length 6
Link-type after encap: ip
ARP
```
