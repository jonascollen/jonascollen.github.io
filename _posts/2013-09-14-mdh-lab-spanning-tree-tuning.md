---
title: MDH Lab - Spanning-tree Tuning
date: 2013-09-14 16:07
comments: true
categories: [Switch]
tags: [stp]
---
Topologi
--------

![lab3-2](/assets/images/2013/09/lab3-21.png)

Objective
---------

Observe what happens when the default spanning tree behavior is modified.

Background
----------

Four switches have just been installed. The distribution layer switches are Catalyst 3560s, and the accesslayer switches are Catalyst 2960s. There are redundant uplinks between the access layer and distribution layer.

Because of the possibility of bridging loops, spanning tree logically removes any redundant links. In this lab, you will see what happens when the default spanning tree behavior is modified.

Verifiering
-----------

Vi kan väl börja med att ta en titt på hur STP ser ut innan vi börjar modifera något.

S1
```
S1#sh spanning-tree
VLAN0001
 Spanning tree enabled protocol ieee
 Root ID Priority 32769
 Address 0014.a889.9c80
 Cost 19
 Port 5 (FastEthernet0/3)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
Bridge ID Priority 32769 (priority 32768 sys-id-ext 1)
 Address 0024.c33f.9e80
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 300
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 P2p 
Fa0/2 Desg FWD 19 128.4 P2p 
Fa0/3 Root FWD 19 128.5 P2p 
Fa0/4 Altn BLK 19 128.6 P2p
```
S2
```
S2#sh spanning-tree
VLAN0001
 Spanning tree enabled protocol ieee
 Root ID Priority 32769
 Address 0014.a889.9c80
 Cost 19
 Port 3 (FastEthernet0/3)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
Bridge ID Priority 32769 (priority 32768 sys-id-ext 1)
 Address 0025.b4c7.c580
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 300
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Altn BLK 19 128.1 P2p 
Fa0/2 Altn BLK 19 128.2 P2p 
Fa0/3 Root FWD 19 128.3 P2p 
Fa0/4 Altn BLK 19 128.4 P2p
```
S3
```
S3#sh spanning-tree
VLAN0001
 Spanning tree enabled protocol ieee
 Root ID Priority 32769
 Address 0014.a889.9c80
 **This bridge is the root**
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
Bridge ID Priority 32769 (priority 32768 sys-id-ext 1)
 Address 0014.a889.9c80
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 300
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 P2p 
Fa0/2 Desg FWD 19 128.4 P2p 
Fa0/3 Desg FWD 19 128.5 P2p 
Fa0/4 Desg FWD 19 128.6 P2p
```
Som synes är S3 root, varför? Om vi jämför mac-adresserna så ser vi att S3 har lägst, och då priority för alla tre är 32768 vinner därför S3.

S1's root-port är fa0/3, fa0/4 står därför i Alternate/Blocking pga lägre port-id (cost är samma).

S2's root-port är fa0/3, fa0/4 står därför i Alternate/Blocking pga lägre port-id (cost samma). Fa0/1-2 stängs också av då S1 har lägre mac-adress och sätter därför sina portar mot S2 som designated. Vi kan endast ha en designated-port per länk och fa0/1-2 blir därför Alternate.

Genomförande
------------

Vi börjar med att konfa S1 som Primary och S3 som Secondary root.

S1

`S1(config)#spanning-tree vlan 1 root primary
`
S3

`S3(config)#spanning-tree vlan 1 root secondary`

Vi kan se att R2's root-port nu ändrats till Fa0/1:
```
S2#sh spanning-tree
VLAN0001
 Spanning tree enabled protocol ieee
 Root ID Priority 24577
 **Address 0024.c33f.9e80 <- S1**
 Cost 19
 Port 1 (FastEthernet0/1)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
Bridge ID Priority 32769 (priority 32768 sys-id-ext 1)
 Address 0025.b4c7.c580
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 15
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
**Fa0/1 Root FWD 19 128.1 P2p** 
Fa0/2 Altn BLK 19 128.2 P2p 
Fa0/3 Altn BLK 19 128.3 P2p 
Fa0/4 Altn BLK 19 128.4 P2p
```
S3 som tidigare hade alla portar som designated (root-bridge) har nu satt Fa0/3 som root-port och fa0/4 som alternate.
```
S3(config)#do sh sp
VLAN0001
 Spanning tree enabled protocol ieee
 Root ID Priority 24577
 Address 0024.c33f.9e80
 Cost 19
 Port 5 (FastEthernet0/3)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
Bridge ID Priority 28673 (priority 28672 sys-id-ext 1)
 Address 0014.a889.9c80
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 300
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 P2p 
Fa0/2 Desg FWD 19 128.4 P2p 
Fa0/3 Root FWD 19 128.5 P2p 
Fa0/4 Altn BLK 19 128.6 P2p
```
Hur gör vi då om vi vill att STP ska föredra att gå över Fa0/4 istället för Fa0/3?

Första alternativet är att ändra port-priority (default 128), kom ihåg att lägst är bäst:
```
S3(config)#int fa0/4
S3(config-if)#spanning-tree port-priority ?
<0-240> port priority in increments of 16
S3(config-if)#spanning-tree port-priority 112
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 P2p 
Fa0/2 Desg FWD 19 128.4 P2p 
Fa0/3 Root FWD 19 128.5 P2p 
Fa0/4 Altn BLK 19 112.6 P2p
```
När vi verifierar med sh spanning-tree märks det ingen förändring? Vi har lägre priority men porten står fortfarande som ALT/Blocking. Detta hade jag helt glömt bort, root-port bestäms efter vad "upstream-neighbor" advertisar som port-id (port-priority+ interface-id). Ändrar vi istället samma konfig på S1 så kan vi direkt se skillnaden:
```
S1(config-if)#spanning-tree port-priority 112
S3#sh spanning-tree
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 P2p 
Fa0/2 Desg FWD 19 128.4 P2p 
Fa0/3 Altn BLK 19 128.5 P2p 
**Fa0/4 Root FWD 19 128.6 P2p**
```
Hittade följande förklaring på nätet som förklarade koncecptet bra:

> The root bridge originates the bpdus, switch D receives bpdus on ports 5/1 and 5/2 , it compares the two bpdus received, both have the same bridge id so the port cost is checked. Port cost is dependent on the bw of the interface, in this case both have the same bandwidth so the senders port id is checked.
> 
> The senders port-id consists of the senders port priority and the port number of the sending interface.The bpdu with the lowest port-id will be preferred so the interface which received the best bpdu will be root and the other interface with a lower priority bpdu is blocked.
> 
> So in order to manipulate which port is forwarding or blocking on your local switch, you must configure the remote switch ports so that they will modify bpdu parameters on transmission.

Vi kan även ändra spanning-tree cost för en länk, vilket som synes är 19 just nu på samtliga länkar mellan S1-S2-S3 (default för 100M).

Om vi vill ändra S2's root-port från Fa0/1 till Fa0/2 behöver vi bara öka costen på Fa0/1 (alternativt sänka den på Fa0/2):
```
S2(config)#inte fa0/1
S2(config-if)#spanning-tree cost ?
 <1-200000000> port path cost
S2(config-if)#spanning-tree cost 200
S2#sh spanning-tree
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Altn BLK 200 128.1 P2p 
**Fa0/2 Root LIS 19 128.2 P2p** 
Fa0/3 Altn BLK 19 128.3 P2p 
Fa0/4 Altn BLK 19 128.4 P2p
```
Observera att för cost så kan vi ändra direkt på switchen, vi behöver inte göra ändringen på neighbor. Detta då switchen alltid lägger till sin egen "Root Path Cost" när den mottager ett BPDU. S1 kommer skicka BPDU's med RPC = 0, för fa0/1 blir det då 0+200, och fa0/2 0+19 -> fa0/2 blir root-port pga lägre cost.
