---
title: MDH Lab - Basic Trunking & VTP
date: 2013-09-14 14:13
comments: true
categories: [Switch]
tags: [vtp]
---
Topologi
--------

![lab2-1](/assets/images/2013/09/lab2-11.png)

Objectives
----------

*   Set up a VTP domain
*   Create and maintain VLANs
*   Configure ISL and 802.1Q trunking

Background  
------------

VLANs logically segment a network by function, team, or application, regardless of the physical location of the users. End stations in a particular IP subnet are often associated with a specific VLAN. VLAN membership on a switch that is assigned manually for each interface is known as static VLAN membership. Trunking, or connecting switches, and the VLAN Trunking Protocol (VTP) are technologies that support VLANs. VTP manages the addition, deletion, and renaming of VLANs on the entire network from a single central switch.  Note: This lab uses Cisco WS-C2960-24TT-L switches with the Cisco IOS image c2960-lanbasek9-mz.122- 46.SE.bin, and Catalyst 3560-24PS with the Cisco IOS image c3560-advipservicesk9-mz.122-46.SE.bin. You can use other switches (such as a 2950 or 3550) and Cisco IOS Software versions if they have comparable capabilities and features. Depending on the switch model and Cisco IOS Software version, the commands available and output produced might vary from what is shown in this lab.

Genomförande
------------

Första labben av ~10 vi fått från MDH att genomföra närmaste 2 veckorna. De första verkar väldigt basic dock så får se hur långt jag hinner idag, skönt att vara igång och konfa lite och släppa teorin ;) Det är ingen idé att börja med VTP då det kräver fungerande trunk-länkar för att kunna utbyta paket, så vi tar trunkarna & vlan1 först. S1

```
Switch(config)#hostname S1
S1(config)#line con 0
S1(config-line)#logging synchronous
S1(config-line)#int range fa0/1 - 2
S1(config-if-range)#switchport trunk encapsulation dot1q 
S1(config-if-range)#switchport mode dynamic desirable 
S1(config-if-range)#description To S2
S1(config-if-range)#int range fa0/3 - 4
S1(config-if-range)#switchport trunk encapsulation isl
S1(config-if-range)#switchport mode dynamic desirable
S1(config-if-range)#description To S3
S1(config)#int vlan 1
S1(config-if)#ip add 10.1.1.101 255.255.255.0
S1(config-if)#no shut
```
S3
```
Switch(config)#hostname S3
S3(config)#line con 0
S3(config-line)#logging synchronous 
S3(config-line)#exit
S3(config)#int range fa0/1 - 2
SS3(config-if-range)#switchport trunk encapsulation dot1q
S3(config-if-range)#switchport mode dynamic desirable 
S3(config-if-range)#description to S2
S3(config-if-range)#int range fa0/3 - 4
S3(config-if-range)#switchport trunk encapsulation ISL
S3(config-if-range)#switchport mode dynamic auto
S3(config-if-range)#description to S1
S3(config-if-range)#end
S3(config)#int vlan1
S3(config-if)#ip add 10.1.1.102 255.255.255.0
S3(config-if)#no shut
```
S2
```
Switch(config)#hostname S2
S2(config)#line con 0
S2(config-line)#logging synchronous 
S2(config-line)#exit
S2(config)#inte range fa0/1 - 2
S2(config-if-range)#switchport mode dynamic auto
S2(config-if-range)#description to S1
S2(config-if-range)#int range fa0/3 - 4
S2(config-if-range)#switchport mode dynamic auto
S2(config-if-range)#description to S3
S2(config)#int vlan 1
S2(config-if)#ip add 10.1.1.103 255.255.255.0
S2(config-if)#no shut
```
S2 är en 2960 och har ej stöd för ISL, vi behöver därför endast sätta dynamic auto på upplänkarna mot S1 & S3.
```
S1#sh interface trunk
Port Mode Encapsulation Status Native vlan
Fa0/1 desirable 802.1q trunking 1
Fa0/2 desirable 802.1q trunking 1
Fa0/3 desirable isl trunking 1
Fa0/4 desirable isl trunking 1
```
Allt ok så långt, vi tar och sätter S1 som VTP-server & skapar vlanen där, S2+S3 konfas som VTP klienter. S1
```
S1(config)#vlan 100
S1(config-vlan)#name ServerFarm1
S1(config-vlan)#vlan 110
S1(config-vlan)#name ServerFarm2
S1(config-vlan)#vlan 120
S1(config-vlan)#name Net-Eng
S1(config-vlan)#exit
S1(config)#vtp mode server
Device mode already VTP SERVER.
S1(config)#vtp domain CCIE 
Changing VTP domain name from NULL to CCIE
S1(config)#vtp password cc1e
Setting device VLAN database password to cc1e
S1(config)#vtp version 2
```
S3
```
S3(config)#vtp mode client
Setting device to VTP CLIENT mode.
S3(config)#vtp domain CCIE
**Domain name already set to CCIE. <- Kom ihåg att VTP-clienter automatiskt byter till den domän som annonseras om de inte redan är med i en domän.**
S3(config)#vtp password cc1e
Setting device VLAN database password to cc1e
S3(config)#vtp version 2 <- Annonseras av VTP-servern
Cannot modify version in VTP client mode
```
S2
```
S2(config)#vtp mode client
Setting device to VTP CLIENT mode.
S2(config)#vtp password cc1e
Setting device VLAN database password to cc1e
S2#sh vlan brief
VLAN Name Status Ports
---- -------------------------------- --------- -------------------------------
1 default active Fa0/5, Fa0/6, Fa0/7, Fa0/8
 Fa0/9, Fa0/10, Fa0/11, Fa0/12
 Fa0/13, Fa0/14, Fa0/15, Fa0/16
 Fa0/17, Fa0/18, Fa0/19, Fa0/20
 Fa0/21, Fa0/22, Fa0/23, Fa0/24
 Gi0/1, Gi0/2
**100 ServerFarm1 active** 
**110 ServerFarm2 active** 
**120 Net-Eng active** 
1002 fddi-default act/unsup 
1003 token-ring-default act/unsup 
1004 fddinet-default act/unsup 
1005 trnet-default act/unsup
```
Sen kan vi väl även ta och slänga in några interface i respektive vlan med grundläggande säkerhetsfunktioner. S1
```
S1(config)#int range fa0/5 - 10
S1(config-if-range)#switchport mode access
S1(config-if-range)#switchport access vlan 100
S1(config-if-range)#spanning-tree portfast
S1(config-if-range)#description ServerFarm1, vlan 100
S1(config-if-range)#switchport port-security 
S1(config-if-range)#switchport port-security violation shutdown
S1(config-if-range)#switchport port-security max 1
```
S3
```
S3(config)#int range fa0/5 - 10
S3(config-if-range)#switchport mode access
S3(config-if-range)#switchport access vlan 120
S3(config-if-range)#spanning-tree portfast 
S3(config-if-range)#description Net-Eng
S3(config-if-range)#switchport port-security 
S3(config-if-range)#switchport port-security violation shutdown
S3(config-if-range)#switchport port-security max 1
```
S2
```
S2(config)#int range fa0/5 - 10
S2(config-if-range)#switchport mode access
S2(config-if-range)#switchport access vlan 110
S2(config-if-range)#spanning-tree portfast
S2(config-if-range)#description ServerFarm2, vlan 110
S2(config-if-range)#switchport port-security 
S2(config-if-range)#switchport port-security violation shutdown 
S2(config-if-range)#switchport port-security max 1
```
Klart! Har tyvärr inga hostar jag kan testa mot men genom show interface trunk, show vlan, show vtp status kan vi verifera att allt ser ok ut.
```
S1#sh vtp status
VTP Version : running VTP2
Configuration Revision : 4
Maximum VLANs supported locally : 1005
Number of existing VLANs : 8
VTP Operating Mode : Server
VTP Domain Name : CCIE
VTP Pruning Mode : Disabled
VTP V2 Mode : Enabled
VTP Traps Generation : Disabled
MD5 digest : 0x27 0x6D 0xB4 0x0D 0xA5 0xB5 0x1E 0x45 
Configuration last modified by 0.0.0.0 at 3-1-93 01:06:54
Local updater ID is 0.0.0.0 (no valid interface found)

S1#sh interface trunk
Port Mode Encapsulation Status Native vlan
Fa0/1 desirable **802.1q** trunking 1
Fa0/2 desirable **802.1q** trunking 1
Fa0/3 desirable **isl** trunking 1
Fa0/4 desirable **isl** trunking 1

VLAN Name Status Ports
---- -------------------------------- --------- -------------------------------
1 default active Fa0/11, Fa0/12, Fa0/13, Fa0/14
 Fa0/15, Fa0/16, Fa0/17, Fa0/18
 Fa0/19, Fa0/20, Fa0/21, Fa0/22
 Fa0/23, Fa0/24, Gi0/1, Gi0/2
**100 ServerFarm1 active Fa0/5, Fa0/6, Fa0/7, Fa0/8**
 **Fa0/9, Fa0/10**
**110 ServerFarm2 active** 
**120 Net-Eng active** 
1002 fddi-default act/unsup 
1003 trcrf-default act/unsup 
1004 fddinet-default act/unsup 
1005 trbrf-default act/unsup
```
