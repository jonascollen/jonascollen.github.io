---
title: MDH Lab - Private-VLANs & VACL
date: 2013-09-17 16:28
comments: true
categories: [Private-Vlan,VACL]
---
# Topologi

![pvlan](/assets/images/2013/09/pvlan.png)

Objectives
----------

*   Secure the server farm using private VLANs.
*   Secure the staff VLAN from the student VLAN.
*   Secure the staff VLAN when temporary staff personnel are used.

Background
----------

In this lab, you will configure the network to protect the VLANs using router ACLs, VLAN ACLs, and private VLANs. First, you will secure the new server farm by using private VLANs so that broadcasts on one server VLAN are not heard by the other server VLAN. Service providers use private VLANs to separate different customers’ traffic while utilizing the same parent VLAN for all server traffic. The private VLANs provide traffic isolation between devices, even though they might exist on the same VLAN. You will then secure the staff VLAN from the student VLAN by using a RACL, which prevents traffic from the student VLAN from reaching the staff VLAN. This allows the student traffic to utilize the network and Internet services while keeping the students from accessing any of the staff resources. Lastly, you will configure a VACL that allows a host on the staff network to be set up to use the VLAN for access but keeps the host isolated from the rest of the staff machines. This machine is used by temporary staff employees.

Genomförande
------------

## L2 basic konfig

Då vi ska använda oss av Private-VLANs i den här labben måste vi konfigurera våra switchar till VTP mode Transparent. S1

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
 S1(config)#vtp mode transparent
 Setting device to VTP TRANSPARENT mode.
S1(config)#vlan 100,150,200
 S1(config-vlan)#exit
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
 S3(config)#vtp mode transparent
 Setting device to VTP Transparent mode for VLANS.
 S3(config)#vlan 100,150,200
 S3(config-vlan)#exit
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
S2(config)#vtp mode transparent
Setting device to VTP Transparent mode for VLANS.
S2(config)#vlan 100,150,200
S2(config-vlan)#exit
```
## HSRP

S1
```
S1(config)#interface vlan 1
 S1(config-if)#ip add 172.16.1.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.1.1
 S1(config-if)#standby 1 preempt
 S1(config-if)#standby 1 priority 100
 S1(config-if)#
 S1(config-if)#interface vlan 100
 S1(config-if)#ip add 172.16.100.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.100.1
 S1(config-if)#standby 1 preempt
 S1(config-if)#standby 1 priority 150
 S1(config-if)#
 S1(config-if)#interface vlan 150
 S1(config-if)#ip add 172.16.150.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.150.1
 S1(config-if)#standby 1 preempt
 S1(config-if)#standby 1 priority 150
 S1(config-if)#
 S1(config-if)#interface vlan 200
 S1(config-if)#ip add 172.16.200.10 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#standby 1 ip 172.16.200.1
 S1(config-if)#standby 1 preempt
 S1(config-if)#standby 1 priority 100
 S1(config-if)#
```
S3
```
S3(config)#interface vlan 1
 S3(config-if)#ip add 172.16.1.30 255.255.255.0
 S3(config-if)#no shut
 S3(config-if)#standby 1 ip 172.16.1.1
 S3(config-if)#standby 1 preempt
 S3(config-if)#standby 1 priority 150
 S3(config-if)#
 S3(config-if)#interface vlan 100
 S3(config-if)#ip add 172.16.100.30 255.255.255.0
 S3(config-if)#standby 1 ip 172.16.100.1
 S3(config-if)#standby 1 preempt
 S3(config-if)#standby 1 priority 100
 S3(config-if)#
 S3(config-if)#interface vlan 150
 S3(config-if)#ip add 172.16.150.30 255.255.255.0
 S3(config-if)#standby 1 ip 172.16.150.1
 S3(config-if)#standby 1 preempt
 S3(config-if)#standby 1 priority 100
 S3(config-if)#
 S3(config-if)#interface vlan 200
 S3(config-if)#ip add 172.16.200.30 255.255.255.0
 S3(config-if)#standby 1 ip 172.16.200.1
 S3(config-if)#standby 1 preempt
 S3(config-if)#standby 1 priority 150
 S3(config-if)#
```
Verifiering
```
S1#sh standby brief
 P indicates configured to preempt.
 |
 Interface Grp Pri P State Active Standby Virtual IP
 Vl1 1 100 P Standby 172.16.1.30 local 172.16.1.1
 Vl100 1 150 P Active local 172.16.100.30 172.16.100.1
 Vl150 1 150 P Active local 172.16.150.30 172.16.150.1
 Vl200 1 100 P Standby 172.16.200.30 local 172.16.200.1
```
### Private-VLANs

PVLANs provide layer 2-isolation between ports within the same broadcast domain. There are three types of PVLAN ports:_

*   _Promiscuous— A promiscuous port can communicate with all interfaces, including the isolated and community ports within a PVLAN._
*   _Isolated— An isolated port has complete Layer 2 separation from the other ports within the same PVLAN, but not from the promiscuous ports. PVLANs block all traffic to isolated ports except traffic from promiscuous ports. Traffic from isolated port is forwarded only to promiscuous ports._
*   _Community— Community ports communicate among themselves and with their promiscuous ports. These interfaces are separated at Layer 2 from all other interfaces in other communities or isolated ports within their PVLAN._

Mer info om just PVLANs och design-rekommendationer finns att läsa [här](http://www.cisco.com/en/US/products/hw/switches/ps700/products_tech_note09186a008013565f.shtml)! Vi börjar med att konfigurera upp våra secondary-vlans (Isolated- & Community-PVLAN) i S1 & S3.

```
S1(config)#vlan 151
S1(config-vlan)#private-vlan isolated
S1(config-vlan)#exit
S1(config)#!Community
S1(config)#vlan 152
S1(config-vlan)#private-vlan community
S1(config-vlan)#exit
S1(config)#

S3(config)#vlan 151
S3(config-vlan)#private-vlan isolated
S3(config-vlan)#exit
S3(config)#!Community
S3(config)#vlan 152
S3(config-vlan)#private-vlan community
S3(config-vlan)#exit
```
Vi behöver sedan associera våra PVLAN med "parent"-vlanet 150, "primary:n".
```
S1(config)#vlan 150
S1(config-vlan)#private-vlan primary
S1(config-vlan)#private-vlan association 151,152
S1(config-vlan)#exit

S3(config)#vlan 150
S3(config-vlan)#private-vlan primary
S3(config-vlan)#private-vlan association 151,152
S3(config-vlan)#exit
```
Då vi i detta exemplet använder SVI's för att routa trafik direkt i switchen behöver vi även knyta våra Secondary-vlan till primaryn (150).
```
S1(config)#interface vlan 150
S1(config-if)#private-vlan mapping 151,152
S1(config-if)#
*Mar 1 01:35:40.794: %PV-6-PV_MSG: Created a private vlan mapping, Primary 150, Secondary 151
*Mar 1 01:35:40.794: %PV-6-PV_MSG: Created a private vlan mapping, Primary 150, Secondary 152

S3(config)#int vlan 150
S3(config-if)#private-vlan mapping 151,152
S3(config-if)#
*Mar 1 01:35:37.170: %PV-6-PV_MSG: Created a private vlan mapping, Primary 150, Secondary 151
*Mar 1 01:35:37.170: %PV-6-PV_MSG: Created a private vlan mapping, Primary 150, Secondary 152
```
Verifiera med "show vlan private-vlan":
```
S1#sh vlan private-vlan
Primary Secondary Type Ports
------- --------- ----------------- ------------------------------------------
150 151 isolated 
150 152 community
```
Nu återstår det endast att koppla interfacen till respektive secondary-vlan. Enligt uppgiften ska fördelningen se ut enligt följande:

*   Fa0/5-10 - Isolated
*   Fa0/11-15 - Community

S1
```
S1(config)#int range fa0/5 - 10
S1(config-if-range)#description Isolated-port
S1(config-if-range)#switchport mode private-vlan host
S1(config-if-range)#switchport private-vlan host-association 150 151
S1(config-if-range)#
S1(config-if-range)#int range fa0/11 - 15
S1(config-if-range)#description Community-port
S1(config-if-range)#switchport mode private-vlan host
S1(config-if-range)#switchport private-vlan host-association 150 152
```
S3
```
S3(config)#int range fa0/5 - 10
S3(config-if-range)#description Isolated-port
S3(config-if-range)#switchport mode private-vlan host
S3(config-if-range)#switchport private-vlan host-association 150 151
S3(config-if-range)#
S3(config-if-range)#int range fa0/11 - 15
S3(config-if-range)#description Community-port
S3(config-if-range)#switchport mode private-vlan host
S3(config-if-range)#switchport private-vlan host-association 150 152
```
Verifiering
```
S3#show vlan private-vlan
Primary Secondary Type Ports
------- --------- ----------------- ------------------------------------------
150 151 isolated Fa0/5, Fa0/6, Fa0/7, Fa0/8, Fa0/9, Fa0/10
150 152 community Fa0/11, Fa0/12, Fa0/13, Fa0/14, Fa0/15
```
## RACL

Vi skulle även skydda Vlan 200 (172.16.200.0/24) från Vlan 100 (172.16.100.0/24), vilket vi gör enkelt med en vanlig ACL.
```
S1(config)#ip access-list extended RACL
S1(config-ext-nacl)#deny ip 172.16.100.0 0.0.0.255 172.16.200.0 0.0.0.255
S1(config-ext-nacl)#permit ip any any
S1(config-ext-nacl)#exit
S1(config)#interface vlan 200
S1(config-if)#ip access-group RACL in
```
S3
```
S3(config)#ip access-list extended RACL
S3(config-ext-nacl)#deny ip 172.16.100.0 0.0.0.255 172.16.200.0 0.0.0.255
S3(config-ext-nacl)#permit ip any any
S3(config-ext-nacl)#exit
S3(config)#interface vlan 200
S3(config-if)#ip access-group RACL in
S3(config-if)#
```
### VACL

Vlan-ACL är ett nytt koncept i CCNP, mer info finns att läsa [här](http://www.cisco.com/en/US/products/hw/switches/ps700/products_tech_note09186a008013565f.shtml)! Istället för att knyta det till ett interface används det istället direkt på VLAN:et, själva konfigureringen påminner väldigt mycket om route-maps som vi använt oss mycket av i tidigare inlägg om ex. [route-filtering](http://roadtoccie.se/2013/06/26/route-filtering-the-basics/). Enligt specifikationen skulle vi blockera hosten 172.16.100.150 från att nå någon annan på vlan 100. Vi skapar först en ACL för att ha något att använda i match-statement.
```
S1(config)#ip access-list extended VACL-BLOCK
S1(config-ext-nacl)#permit ip host 172.16.100.150 172.16.100.0 0.0.0.255
S1(config-ext-nacl)#exit

S3(config)#ip access-list extended VACL-BLOCK
S3(config-ext-nacl)#permit ip host 172.16.100.150 172.16.100.0 0.0.0.255
S3(config-ext-nacl)#exit
```
Vi bygger sedan vår VLAN Access-map, glöm inte att utan en match all/permit så blockerar vi all trafik precis som en vanlig ACL.
```
S1(config)#vlan access-map AM-VACL-BLOCK 10
S1(config-access-map)#match ip addr VACL-BLOCK
S1(config-access-map)#action drop
S1(config-access-map)#exit
S1(config)#vlan access-map AM-VACL-BLOCK 20
S1(config-access-map)#action forward
S1(config-access-map)#exit
S1(config)#

S3(config)#vlan access-map AM-VACL-BLOCK 10
S3(config-access-map)#match ip addr VACL-BLOCK
S3(config-access-map)#action drop
S3(config-access-map)#exit
S3(config)#vlan access-map AM-VACL-BLOCK 20
S3(config-access-map)#action forward
S3(config-access-map)#exit
S3(config)#
```
Sen återstår det bara att knyta detta till vlan:et.

```
S1(config)#vlan filter AM-VACL-BLOCK vlan-list 100
S3(config)#vlan filter AM-VACL-BLOCK vlan-list 100
```
Tyvärr har vi inga host att testa med så vi får helt enkelt räkna med att allt är ok! ;)
