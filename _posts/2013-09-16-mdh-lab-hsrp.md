---
title: MDH Lab - HSRP
date: 2013-09-16 11:42
comments: true
categories: [HSRP]
---
Topologi
--------

![lab5-2](/assets/images/2013/09/lab5-2.png)

Objective
---------

Configure inter-VLAN routing with HSRP to provide redundant, fault-tolerant routing to the internal network.

Background
----------

Hot Standby Router Protocol (HSRP) is a Cisco-proprietary redundancy protocol for establishing a faulttolerant default gateway. It is described in RFC 2281. HSRP provides a transparent failover mechanism to the end stations on the network. This provides users at the access layer with uninterrupted service to the network if the primary gateway becomes inaccessible. The Virtual Router Redundancy Protocol (VRRP) is a standards-based alternative to HSRP and is defined in RFC 3768. The two technologies are similar but not compatible. This lab focuses on HSRP.

Genomförande
------------

Börjar med default-konfig för att få upp vlan/etherchannels/trunkar. S1

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
S1(config-if-range)#
 S1(config-if-range)#!Trunk-links till S3
 S1(config-if-range)#int range fa0/3 - 4
 S1(config-if-range)#switchport trunk encaps dot1q
 S1(config-if-range)#switchport mode trunk
 S1(config-if-range)#description to S2
 S1(config-if-range)#channel-protocol lacp
 S1(config-if-range)#channel-group 2 mode active
 Creating a port-channel interface Port-channel 2
S1(config-if-range)#exit
 S1(config)#
 S1(config)#vtp mode server
 Device mode already VTP SERVER.
 S1(config)#vtp domain CISCO
 Changing VTP domain name from NULL to CISCO
 S1(config)#
 S1(config)#vlan 10
 S1(config-vlan)#name Red
 S1(config-vlan)#vlan 20
 S1(config-vlan)#name Blue
 S1(config-vlan)#vlan 30
 S1(config-vlan)#name Orange
 S1(config-vlan)#vlan 40
 S1(config-vlan)#
```
S3
```
Switch(config)#hostname S3
 S3(config)#line con 0
 S3(config-line)#logging sync
 S3(config-line)#!Trunk-links till S2
 S3(config-line)#int range fa0/1 - 2
 S3(config-if-range)#switchport trunk encaps dot1q
 S3(config-if-range)#switchport mode trunk
 S3(config-if-range)#description to S2
 S3(config-if-range)#channel-protocol lacp
 S3(config-if-range)#channel-group 1 mode active
 Creating a port-channel interface Port-channel 1
S3(config-if-range)#
 S3(config-if-range)#!Trunk-links till S1
 S3(config-if-range)#int range fa0/3 - 4
 S3(config-if-range)#switchport trunk encaps dot1q
 S3(config-if-range)#switchport mode trunk
 S3(config-if-range)#description to S1
 S3(config-if-range)#channel-protocol lacp
 S3(config-if-range)#channel-group 2 mode passive
 Creating a port-channel interface Port-channel 2
S3(config-if-range)#exit
 S3(config)#
 S3(config)#vtp mode client
 Setting device to VTP CLIENT mode.
 S3(config)#vtp domain CISCO
```
S2
```
Switch(config)#hostname S2
 S2(config)#line con 0
 S2(config-line)#logging sync
 S2(config-line)#!Trunk-links till S1
 S2(config-line)#int range fa0/1 - 2
 S2(config-if-range)#switchport mode trunk
 S2(config-if-range)#description to S1
 S2(config-if-range)#channel-protocol lacp
 S2(config-if-range)#channel-group 1 mode passive
 Creating a port-channel interface Port-channel 1
S2(config-if-range)#
 S2(config-if-range)#!Trunk-links till S3
 S2(config-if-range)#int range fa0/3 - 4
 S2(config-if-range)#switchport mode trunk
 S2(config-if-range)#description to S3
 S2(config-if-range)#channel-protocol lacp
 S2(config-if-range)#channel-group 2 mode passive
 Creating a port-channel interface Port-channel 2
S2(config-if-range)#exit
 S2(config)#
 S2(config)#vtp mode client
 Setting device to VTP CLIENT mode.
 S2(config)#vtp domain CISCO
 Domain name already set to CISCO.
```
Då återstår det bara att sätta upp HSRP mellan S1 & S3. Enligt labben ska fördelningen vara enligt följande:

*   S1 Primary - Vl1, 20 & 40
*   S3 Primary - Vl10 & 30

Vi styr detta genom att modfiera priority-värdet för den switch vi vill ska vara active (default = 100, högst värde vinner). S1
```
S1(config)#interface vlan 1
 S1(config-if)#ip add 172.16.1.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.1.1
 **S1(config-if)#standby 1 priority 150**
 S1(config-if)#standby 1 preempt
 S1(config-if)#
 S1(config-if)#interface vlan 10
 S1(config-if)#ip add 172.16.10.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.10.1
 S1(config-if)#standby 1 priority 100
 S1(config-if)#standby 1 preempt
 S1(config-if)#
 S1(config-if)#interface vlan 20
 S1(config-if)#ip add 172.16.20.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.20.1
 **S1(config-if)#standby 1 priority 150**
 S1(config-if)#standby 1 preempt
 S1(config-if)#
 S1(config-if)#interface vlan 30
 S1(config-if)#ip add 172.16.30.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.30.1
 S1(config-if)#standby 1 priority 100
 S1(config-if)#standby 1 preempt
 S1(config-if)#
 S1(config-if)#interface vlan 40
 S1(config-if)#ip add 172.16.40.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.40.1
 **S1(config-if)#standby 1 priority 150**
 S1(config-if)#standby 1 preempt
 S1(config-if)#exit
 **S1(config)#ip routing**
```
S3
```
S3(config)#interface vlan 1
 S3(config-if)#ip add 172.16.1.30 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#standby 1 ip 172.16.1.1
 S3(config-if)#standby 1 priority 100
 S3(config-if)#standby 1 preempt
 S3(config-if)#
 S3(config-if)#interface vlan 10
 S3(config-if)#ip add 172.16.10.30 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#standby 1 ip 172.16.10.1
 **S3(config-if)#standby 1 priority 150**
 S3(config-if)#standby 1 preempt
 S3(config-if)#
 S3(config-if)#interface vlan 20
 S3(config-if)#ip add 172.16.20.30 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#standby 1 ip 172.16.20.1
 S3(config-if)#standby 1 priority 100
 S3(config-if)#standby 1 preempt
 S3(config-if)#
 S3(config-if)#interface vlan 30
 S3(config-if)#ip add 172.16.30.30 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#standby 1 ip 172.16.30.1
 **S3(config-if)#standby 1 priority 150**
 S3(config-if)#standby 1 preempt
 S3(config-if)#
 S3(config-if)#interface vlan 40
 S3(config-if)#ip add 172.16.40.30 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#standby 1 ip 172.16.40.1
 S3(config-if)#standby 1 priority 100
 S3(config-if)#standby 1 preempt
 S3(config-if)#exit
 S3(config)#ip routing
 S3(config)#
```
S2
```
S2(config)#interface vlan 1
 S2(config-if)#ip add 172.16.1.2 255.255.255.0
 S2(config-if)#no shut
 S2(config-if)#exit
 S2(config)#
 S2(config)#ip default-gateway 172.16.1.1
```
Verifiering
-----------
```
S3#sh standby
Vlan1 - Group 1
 **State is Standby**
 Virtual IP address is 172.16.1.1
 Active virtual MAC address is 0000.0c07.ac01
 Local virtual MAC address is 0000.0c07.ac01 (v1 default)
 Hello time 3 sec, hold time 10 sec
 Next hello sent in 1.216 secs
 Preemption enabled
 Active router is 172.16.1.10, priority 150 (expires in 9.600 sec)
 Standby router is local
 Priority 100 (default 100)
 Group name is "hsrp-Vl1-1" (default)
Vlan10 - Group 1
 **State is Active**
 Virtual IP address is 172.16.10.1
 Active virtual MAC address is 0000.0c07.ac01
 Local virtual MAC address is 0000.0c07.ac01 (v1 default)
 Hello time 3 sec, hold time 10 sec
 Next hello sent in 0.208 secs
 Preemption enabled
 Active router is local
 Standby router is 172.16.10.10, priority 100 (expires in 10.112 sec)
 Priority 150 (configured 150)
 Group name is "hsrp-Vl10-1" (default)
Vlan20 - Group 1
 **State is Standby**
 Virtual IP address is 172.16.20.1
 Active virtual MAC address is 0000.0c07.ac01
 Local virtual MAC address is 0000.0c07.ac01 (v1 default)
 Hello time 3 sec, hold time 10 sec
 Next hello sent in 0.560 secs
 Preemption enabled
 Active router is 172.16.20.10, priority 150 (expires in 8.080 sec)
 Standby router is local
 Priority 100 (default 100)
 Group name is "hsrp-Vl20-1" (default)
Vlan30 - Group 1
 **State is Active**
 Virtual IP address is 172.16.30.1
 Active virtual MAC address is 0000.0c07.ac01
 Local virtual MAC address is 0000.0c07.ac01 (v1 default)
 Hello time 3 sec, hold time 10 sec
 Next hello sent in 1.824 secs
 Preemption enabled
 Active router is local
 Standby router is 172.16.30.10, priority 100 (expires in 10.496 sec)
 Priority 150 (configured 150)
 Group name is "hsrp-Vl30-1" (default)
Vlan40 - Group 1
 **State is Standby**
 Virtual IP address is 172.16.40.1
 Active virtual MAC address is 0000.0c07.ac01
 Local virtual MAC address is 0000.0c07.ac01 (v1 default)
 Hello time 3 sec, hold time 10 sec
 Next hello sent in 1.040 secs
 Preemption enabled
 Active router is 172.16.40.10, priority 150 (expires in 10.608 sec)
 Standby router is local
 Priority 100 (default 100)
 Group name is "hsrp-Vl40-1" (default)
S2#ping 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/203/1007 ms
```
Allt ok så långt. Vi kan även testa failover:
```
S1(config)#inte range fa0/1 - 4
S1(config-if-range)#shut
```
En debug visar då följande på S3:
```
S3#
*Mar 1 00:19:36.980: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/3, changed state to down
*Mar 1 00:19:36.988: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/4, changed state to down
*Mar 1 00:19:36.997: %LINEPROTO-5-UPDOWN: Line protocol on Interface Port-channel2, changed state to down
S3#
*Mar 1 00:19:37.978: %LINK-3-UPDOWN: Interface FastEthernet0/3, changed state to down
*Mar 1 00:19:38.012: %LINK-3-UPDOWN: Interface Port-channel2, changed state to down
*Mar 1 00:19:38.012: %LINK-3-UPDOWN: Interface FastEthernet0/4, changed state to down
S3#
*Mar 1 00:19:45.452: HSRP: Vl30 Grp 1 Standby router is unknown, was 172.16.30.10
*Mar 1 00:19:45.452: HSRP: Vl30 Nbr 172.16.30.10 no longer standby for group 1 (Active)
*Mar 1 00:19:45.452: HSRP: Vl30 Nbr 172.16.30.10 Was active or standby - start passive holddown
***Mar 1 00:19:45.872: HSRP: Vl10 Grp 1 Standby router is unknown, was 172.16.10.10**
***Mar 1 00:19:45.872: HSRP: Vl10 Nbr 172.16.10.10 no longer standby for group 1 (Active)**
*Mar 1 00:19:45.872: HSRP: Vl10 Nbr 172.16.10.10 Was active or
S3# standby - start passive holddown
*Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Standby: c/Active timer expired (172.16.1.10)
***Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Active router is local, was 172.16.1.10**
***Mar 1 00:19:45.872: HSRP: Vl1 Nbr 172.16.1.10 no longer active for group 1 (Standby)**
***Mar 1 00:19:45.872: HSRP: Vl1 Nbr 172.16.1.10 Was active or standby - start passive holddown**
*Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Standby router is unknown, was local
***Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Standby -> Act**
**S3#ive**
*Mar 1 00:19:45.872: %HSRP-5-STATECHANGE: Vlan1 Grp 1 state Standby -> Active
*Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Redundancy "hsrp-Vl1-1" state Standby -> Active
*Mar 1 00:19:45.872: HSRP: Vl1 Added 172.16.1.1 to ARP (0000.0c07.ac01)
*Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Activating MAC 0000.0c07.ac01
*Mar 1 00:19:45.872: HSRP: Vl1 Grp 1 Adding 0000.0c07.ac01 to MAC address filter
*Mar 1 00:19:45.872: HSRP: Vl1 IP Redundancy "hsrp-Vl1-1" standby, local -> unknown
*Mar 1 00:19:45.872: HSRP:
S3# Vl1 IP Redundancy "hsrp-Vl1-1" update, Standby -> Active
*Mar 1 00:19:46.023: HSRP: Vl20 Grp 1 Standby: c/Active timer expired (172.16.20.10)
*Mar 1 00:19:46.023: HSRP: Vl20 Grp 1 Active router is local, was 172.16.20.10
*Mar 1 00:19:46.023: HSRP: Vl20 Nbr 172.16.20.10 no longer active for group 1 (Standby)
*Mar 1 00:19:46.023: HSRP: Vl20 Nbr 172.16.20.10 Was active or standby - start passive holddown
*Mar 1 00:19:46.023: HSRP: Vl20 Grp 1 Standby router is unknown, was local
*Mar 1 00:19:46.02
S3#3: HSRP: Vl20 Grp 1 Standby -> Active
***Mar 1 00:19:46.023: %HSRP-5-STATECHANGE: Vlan20 Grp 1 state Standby -> Active**
*Mar 1 00:19:46.023: HSRP: Vl20 Grp 1 Redundancy "hsrp-Vl20-1" state Standby -> Active
*Mar 1 00:19:46.023: HSRP: Vl20 Added 172.16.20.1 to ARP (0000.0c07.ac01)
*Mar 1 00:19:46.023: HSRP: Vl20 Grp 1 Activating MAC 0000.0c07.ac01
*Mar 1 00:19:46.023: HSRP: Vl20 Grp 1 Adding 0000.0c07.ac01 to MAC address filter
*Mar 1 00:19:46.023: HSRP: Vl20 IP Redundancy "hsrp-Vl20-1" standby, lo
S3#cal -> unknown
*Mar 1 00:19:46.023: HSRP: Vl20 IP Redundancy "hsrp-Vl20-1" update, Standby -> Active
*Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Standby: c/Active timer expired (172.16.40.10)
*Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Active router is local, was 172.16.40.10
*Mar 1 00:19:46.392: HSRP: Vl40 Nbr 172.16.40.10 no longer active for group 1 (Standby)
*Mar 1 00:19:46.392: HSRP: Vl40 Nbr 172.16.40.10 Was active or standby - start passive holddown
*Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Standby rout
S3#er is unknown, was local
***Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Standby -> Active**
*Mar 1 00:19:46.392: %HSRP-5-STATECHANGE: Vlan40 Grp 1 state Standby -> Active
*Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Redundancy "hsrp-Vl40-1" state Standby -> Active
*Mar 1 00:19:46.392: HSRP: Vl40 Added 172.16.40.1 to ARP (0000.0c07.ac01)
*Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Activating MAC 0000.0c07.ac01
*Mar 1 00:19:46.392: HSRP: Vl40 Grp 1 Adding 0000.0c07.ac01 to MAC address filter
*Mar 1 00:19:46.392: HSRP:
S3# Vl40 IP Redundancy "hsrp-Vl40-1" standby, local -> unknown
*Mar 1 00:19:46.392: HSRP: Vl40 IP Redundancy "hsrp-Vl40-1" update, Standby -> Active
*Mar 1 00:19:48.875: HSRP: Vl1 IP Redundancy "hsrp-Vl1-1" update, Active -> Active
*Mar 1 00:19:49.043: HSRP: Vl20 IP Redundancy "hsrp-Vl20-1" update, Active -> Active
*Mar 1 00:19:49.412: HSRP: Vl40 IP Redundancy "hsrp-Vl40-1" update, Active -> Active
```
Pingar vi från S2 igen kan vi nu se att S3 har tagit över:
```
S2#ping 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.1, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/3/9 ms
```
Tar vi upp interfacen på S1 så går den återigen Active för Vl1, 20 & 40 pga "standby 1 preempt".,
```
S1#sh standby brief
 P indicates configured to preempt.
 |
Interface Grp Pri P State Active Standby Virtual IP
**Vl1 1 150 P Active local 172.16.1.30 172.16.1.1**
Vl10 1 100 P Standby 172.16.10.30 local 172.16.10.1
**Vl20 1 150 P Active local 172.16.20.30 172.16.20.1**
Vl30 1 100 P Standby 172.16.30.30 local 172.16.30.1
**Vl40 1 150 P Active local 172.16.40.30 172.16.40.1**
```
Klart!
