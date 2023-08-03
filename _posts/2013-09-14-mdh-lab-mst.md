---
layout: post
title: MDH Lab - MST
date: 2013-09-14 23:23
author: Jonas Collén
comments: true
categories: [Switch]
tags: [mst]
---
Topologi
--------

![lab3-4](/assets/images/2013/09/lab3-41.png)

Objective
---------

*   Observe the behavior of multiple spanning tree (MST)

Background
----------

Four switches have just been installed. The distribution layer switches are Catalyst 3560s, and the accesslayer switches are Catalyst 2960s. There are redundant uplinks between the access layer and distributionlayer. Because of the possibility of bridging loops, spanning tree logically removes any redundant links. In this lab, we will group VLANs using MST so that we can have fewer spanning tree instances running at once to minimize switch CPU load.

Genomförande
------------

Har redan lagt in grundkonfig för trunking & vlan, [se tidigare inlägg](http://roadtoccie.se/2013/09/14/mdh-lab-pvstrapid-pvst/ "MDH Lab – PVST/Rapid-PVST") om du är intresserad om hur det är konfat. Vi börjar med att ta fram en MST-konfig i notepad++ som vi kan använda på alla tre switchar:
```
spanning-tree mst configuration
name GoCCIE
revision 1
instance 1 vlan 10,20,30,40,50
instance 2 vlan 60,70,80,90,100
exit
```
Detta paste'ar vi inte i S1, S2 & S3 och aktiverar sedan MST genom kommandot:

`spanning-tree mode mst
`
Det är nämligen rekommenderat att skapa instansen först på samtliga switchar innan vi aktiverar MST för att inte få "region-inconsistencies".
```
S1#show spanning-tree mst
##### MST1 vlans mapped: 10,20,30,40,50
Bridge address 0024.c33f.9e80 priority 32769 (32768 sysid 1)
Root address 0014.a889.9c80 priority 32769 (32768 sysid 1)
 port Fa0/3 cost 200000 rem hops 19
Interface Role Sts Cost Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 200000 128.3 P2p 
Fa0/2 Desg FWD 200000 128.4 P2p 
Fa0/3 Root FWD 200000 128.5 P2p 
Fa0/4 Altn BLK 200000 128.6 P2p
##### MST2 vlans mapped: 60,70,80,90,100
Bridge address 0024.c33f.9e80 priority 32770 (32768 sysid 2)
Root address 0014.a889.9c80 priority 32770 (32768 sysid 2)
 port Fa0/3 cost 200000 rem hops 19
Interface Role Sts Cost Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 200000 128.3 P2p 
Fa0/2 Desg FWD 200000 128.4 P2p 
Fa0/3 Root FWD 200000 128.5 P2p 
Fa0/4 Altn BLK 200000 128.6 P2p
```
Precis som tidigare kan vi se att det default blir S3 som tar rollen som root-bridge pga sin lägre mac-adress.  Enligt uppgiften ska det dock vara enligt följande:

*   S1 - Primary för instans 1, Secondary för instans 2
*   S3  - Secondary för instans 1, Primary för instans 2

Konfigen påminner väldigt mycket om hur vi gör i PVST:
```
S1(config)#spanning-tree mst 1 root primary
S1(config)#spanning-tree mst 2 root secondary
S3(config)#spanning-tree mst 1 root secondary
S3(config)#spanning-tree mst 2 root primary
```
Svårare än så är det inte. Kommer göra lite mer avancerade topologier senare bara jag blir klar med MDH's labbar där vi kan kolla på ex. "MST Multipel Regions" som kan bli lite halvklurigt. :)
