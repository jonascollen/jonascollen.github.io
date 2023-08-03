---
title: MDH Lab - Etherchannel
date: 2013-09-14 14:58
comments: true
categories: [Switch]
tags: [lacp,etherchannel]
---
Topologi
--------

![lab2-2](/assets/images/2013/09/lab2-21.png?w=630)

Objective
---------

*   Configure EtherChannel

Background
----------

Four switches have just been installed. The distribution layer switches are Catalyst 3560 switches, and the access layer switches are Catalyst 2960 switches. There are redundant uplinks between the access layer and distribution layer. Usually, only one of these links could be used; otherwise, a bridging loop might occur. However, using only one link utilizes only half of the available bandwidth. EtherChannel allows up to eight redundant links to be bundled together into one logical link. In this lab, you configure Port Aggregation  Protocol (PAgP), a Cisco EtherChannel protocol, and Link Aggregation Control Protocol (LACP), an IEEE 802.3ad open standard version of EtherChannel.

Genomförande
------------

Är som synes samma topologi från [förra labben](http://roadtoccie.se/2013/09/14/switching-mdh-lab-2-1/ "MDH Lab – Basic Trunking & VTP") så all gammal konfig ligger fortfarande kvar, bör dock inte ställa till med några problem förhoppningsvis. Har dock tagit bort ip-adresserna från vlan 1 (default interface vlan 1) på samtliga switchar. Låt oss börja med att konfa upp PAgP & LACP. S1 - PAgP
```
S1(config)#int range fa0/1 - 2
S1(config-if-range)#shut
S1(config-if-range)#channel-protocol pagp
S1(config-if-range)#channel-group 1 mode desirable 
Creating a port-channel interface Port-channel 1
S1(config-if-range)#no shut
```
S2 - PAgP
```
S2(config)#int range fa0/1 - 2
S2(config-if-range)#shut
S2(config-if-range)#channel-protocol pagp
S2(config-if-range)#channel-group 1 mode auto
Creating a port-channel interface Port-channel 1
S2(config-if-range)#no shut
```
S2 - LACP (Obs, kom ihåg att använda en ny channel-group!)
```
S2(config)#inte range fa0/3 -4
S2(config-if-range)#shut
S2(config-if-range)#channel-protocol lacp
S2(config-if-range)#channel-group 2 mode passive
Creating a port-channel interface Port-channel 2
S2(config-if-range)#no shut
```
S3 - LACP
```
S3(config)#int range fa0/1 -2 
S3(config-if-range)#shut
S3(config-if-range)#channel-protocol lacp
S3(config-if-range)#channel-group 2 mode active
Creating a port-channel interface Port-channel 2
S3(config-if-range)#no shut
```
Enklast är väl att verifiera på S2 att allt är ok.
```
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
2 Po2(SU) LACP Fa0/3(P) Fa0/4(P)
```
Allt ok så långt. Innan vi sätter upp L3-Etherchannel mellan S1 & S3 måste vi vara noga med att konfa upp det i rätt ordning, se tidigare inlägg för en genomgång av [L3-Etherchanne](http://roadtoccie.se/2013/09/12/switching-etherchannel-l2l3/ "Switching – Etherchannel L2/L3")l. S1
```
S1(config)#int range fa0/3 - 4
S1(config-if-range)#shut
S1(config-if-range)#no switchport 
S1(config-if-range)#channel-group 5 mode on
Creating a port-channel interface Port-channel 5
S1(config-if-range)#int po5
% Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
S1(config-if)#ip add 10.1.1.101 255.255.255.0
```
S3
```
S3(config)#int range fa0/3 - 4
S3(config-if-range)#shut
S3(config-if-range)#no switchport 
S3(config-if-range)#channel-group 5 mode on
Creating a port-channel interface Port-channel 5
S3(config-if-range)#no shut
S3(config-if-range)#int po5
% Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
S3(config-if)#ip add 10.1.1.102 255.255.255.0
S3(config-if)#end
S3#ping 10.1.1.101
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.101, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/3/9 ms
```
Då var det bara last-balanseringen kvar, vilket är väldigt simpelt att konfa. För S1, S2 & S3:
```
S3(config)#port-channel load-balance src-dst-mac
S3#show etherchannel load-balance 
EtherChannel Load-Balancing Configuration:
 src-dst-mac
EtherChannel Load-Balancing Addresses Used Per-Protocol:
Non-IP: Source XOR Destination MAC address
 IPv4: Source XOR Destination MAC address
 IPv6: Source XOR Destination MAC address
```
Klart!