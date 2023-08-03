---
title: MDH Lab - PVST/Rapid-PVST
date: 2013-09-14 22:19
comments: true
categories: [Switch]
tags: [pvst,rpvst]
---
Topologi
--------

![lab3-3](/assets/images/2013/09/lab3-31.png)

Objectives
----------

• Observe the behavior of a separate spanning tree instance per VLAN. • Change spanning tree mode to rapid spanning tree.

Background
----------

Four switches have just been installed. The distribution layer switches are Catalyst 3560s, and the access layer switches are Catalyst 2960s. There are redundant uplinks between the access layer and distribution layer. Because of the possibility of bridging loops, spanning tree logically removes any redundant links. In this lab, you will see what happens when spanning tree is configured differently for different VLANs.

Genomförande
------------

Vi börjar med lite grundkonfig för respektive switch. 

S1
```
Switch(config)#hostname S1
S1(config)#line con 0
S1(config-line)#logging synchronous 
S1(config-line)#exit
S1(config)#int range fa0/1 - 2
S1(config-if-range)#switchport trunk encapsulation dot1q
S1(config-if-range)#switchport mode dynamic desirable 
S1(config-if-range)#description to S2
S1(config-if-range)#int range fa0/3 - 4
S1(config-if-range)#switchport trunk encapsulation isl
S1(config-if-range)#switchport mode dynamic desirable 
S1(config-if-range)#description to S3
S1(config-if-range)#exit
S1(config)#vtp mode server
Device mode already VTP SERVER.
S1(config)#vtp domain Cisco
Changing VTP domain name from NULL to Cisco
S1(config)#vtp password cisco
Setting device VLAN database password to cisco
S1(config)#vtp version 2
S1(config)#vlan 10,20,50,60,70,80,90,100
S1(config-vlan)#exit

S1#sh interface trunk
Port Mode Encapsulation Status Native vlan
Fa0/1 desirable 802.1q trunking 1
Fa0/2 desirable 802.1q trunking 1
Fa0/3 desirable isl trunking 1
Fa0/4 desirable isl trunking 1
Port Vlans allowed and active in management domain
Fa0/1 1,10,20,50,60,70,80,90,100
Fa0/2 1,10,20,50,60,70,80,90,100
Fa0/3 1,10,20,50,60,70,80,90,100
Fa0/4 1,10,20,50,60,70,80,90,100
```
S3
```
Switch(config)#hostname S3
S3(config)#line con 0
S3(config-line)#logging synchro
S3(config-line)#int range fa0/1 - 2
S3(config-if-range)#switchport trunk encapsulation dot1q
S3(config-if-range)#switchport mode dynamic desirable 
S3(config-if-range)#description to S2
S3(config-if-range)#int range fa0/3 - 4
S3(config-if-range)#switchport trunk encapsulation isl
S3(config-if-range)#switchport mode dynamic auto
S3(config-if-range)#description to S1
S3(config-if-range)#exit
S3(config)#vtp mode client
Setting device to VTP CLIENT mode.
S3(config)#vtp domain Cisco
Changing VTP domain name from NULL to Cisco
S3(config)#vtp password cisco
Setting device VLAN database password to cisco

3#sh interface trunk
Port Mode Encapsulation Status Native vlan
Fa0/1 desirable 802.1q trunking 1
Fa0/2 desirable 802.1q trunking 1
Fa0/3 auto isl trunking 1
Fa0/4 auto isl trunking 1
Port Vlans allowed and active in management domain
Fa0/1 1,10,20,50,60,70,80,90,100
Fa0/2 1,10,20,50,60,70,80,90,100
Fa0/3 1,10,20,50,60,70,80,90,100
Fa0/4 1,10,20,50,60,70,80,90,100
```
S2
```
Switch(config)#hostname S2
S2(config)#line con 0
S2(config-line)#logging sync
S2(config-line)#int range fa0/1 - 2
S2(config-if-range)#switchport trunk encaps
S2(config-if-range)#switchport mode dynamic auto
S2(config-if-range)#description to S1
S2(config-if-range)#int range fa0/3 - 4
S2(config-if-range)#switchport mode dynamic auto
S2(config-if-range)#description to S3
S2(config-if-range)#exit
S2(config)#vtp mode client
Setting device to VTP CLIENT mode.
S2(config)#vtp domain Cisco
Changing VTP domain name from NULL to Cisco
S2(config)#vtp 
*Mar 1 00:27:37.639: %SW_VLAN-6-VTP_DOMAIN_NAME_CHG: VTP domain name changed to Cisco.
S2(config)#vtp password cisco
Setting device VLAN database password to cisco

S2#sh interface trunk
Port Mode Encapsulation Status Native vlan
Fa0/1 auto 802.1q trunking 1
Fa0/2 auto 802.1q trunking 1
Fa0/3 auto 802.1q trunking 1
Fa0/4 auto 802.1q trunking 1
Port Vlans allowed and active in management domain
Fa0/1 1,10,20,50,60,70,80,90,100
Fa0/2 1,10,20,50,60,70,80,90,100
Fa0/3 1,10,20,50,60,70,80,90,100
Fa0/4 1,10,20,50,60,70,80,90,100
```
Enligt uppgiften ska S1 vara:

*   Primary root-bridge för Vlan 10, 50, 60, 70
*   Secondary root-bridge för Vlan 20, 80, 90, 100

Det fixar vi enkelt med följande kommando:
```
S1(config)#spanning-tree vlan 10,50,60,70 root primary
S1(config)#spanning-tree vlan 20,80,90,100 root secondary
```
Och vice versa på S3:
```
S3(config)#spanning-tree vlan 10,50,60,70 root secondary
S3(config)#spanning-tree vlan 20,80,90,100 root primary
```
Verifiera med:
```
S3#sh spanning-tree vlan 10
VLAN0010
 Spanning tree enabled protocol ieee
 Root ID Priority 24586
 Address 0024.c33f.9e80
 Cost 19
 **Port 5 (FastEthernet0/3)**
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
S3#sh spanning-tree vlan 20
VLAN0020
 Spanning tree enabled protocol ieee
 Root ID Priority 24596
 Address 0014.a889.9c80
 **This bridge is the root**
```
Vi skulle sedan ändra cost för Vlan20 till 15 mellan S1-S2's Fa0/4 interface. Observera att fa0/3 just nu används som root-port på S1 till S3 (root-bridge) innan vi ändrar något, detta pga equal cost 19 - fa0/3 vinner med lägre port-id. Behöver endast ändra på S1 då S3 har alla portar som designated (root-bridge).
```
S1(config)#int fa0/4 
S1(config-if)#spanning-tree vlan 20 cost 15
S1(config-if)#do sh spanning-tree vlan 20
VLAN0020
 Spanning tree enabled protocol ieee
 Root ID Priority 24596
 Address 0014.a889.9c80
 Cost 15
 Port 6 (FastEthernet0/4)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
Bridge ID Priority 28692 (priority 28672 sys-id-ext 20)
 Address 0024.c33f.9e80
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 300
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 P2p 
Fa0/2 Desg FWD 19 128.4 P2p 
**Fa0/3 Altn BLK 19 128.5 P2p** 
**Fa0/4 Root FWD 15 128.6 P2p**
```
Vackert! Vi skulle även byta till rapid-pvst:
```
S1(config)#spanning-tree mode rapid-pvst 
S2(config)#spanning-tree mode rapid-pvst 
S3(config)#spanning-tree mode rapid-pvst 
S3#sh spanning-tree vlan 20
VLAN0020
 **Spanning tree enabled protocol rstp**
 Root ID Priority 24596
 Address 0014.a889.9c80
 This bridge is the root
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
```
Klart!
