---
title: OSPF - Forwarding Address Suppression Type-7
date: 2014-03-24 17:02
comments: true
categories: [OSPF]
tags: [forwarding address suppression, type-7]
---
En till OSPF-labb från [GNS3 Vault](http://gns3vault.com/OSPF/ospf-suppress-forward-address.html). ![ospfsuppressforwardaddress](/assets/images/2014/03/ospfsuppressforwardaddress.png)

Goal:
-----

*   All IP addresses have been preconfigured for you.
*   Configure OSPF and use the correct areas. Ensure Area 1 is a NSSA.
*   Configure RIP between router Charlie and Evelyn.
*   Create a loopback0 interface on router Evelyn with IP address 1.1.1.1 /24 and advertise it in RIP.
*   Redistribute between RIP and OSPF.
*   Configure a prefix-list on router Jake which filters network 192.168.13.0 /24.
*   Ensure you can still reach network 1.1.1.0 /24 from all routers without removing the prefix-list. You are only allowed to use OSPF commands.

Konfig:
-------

Simpel grundkonfig på samtliga enheter, kom ihåg att konfigurera nssa på både Alan & Charlie. 

```
Berta


router ospf 1
 network 192.168.24.0 0.0.0.255 area 2

Jake

router ospf 1
 network 192.168.12.0 0.0.0.255 area 0
 network 192.168.24.0 0.0.0.255 area 2

Alan

router ospf 1
 area 1 nssa
 network 192.168.12.0 0.0.0.255 area 0
 network 192.168.13.0 0.0.0.255 area 1

Charlie

router ospf 1
 area 1 nssa
 redistribute rip metric 20 subnets
 network 192.168.13.0 0.0.0.255 area 1

router rip
 version 2
 redistribute ospf 1 metric 3
 network 192.168.35.0

Evelyn

interface Loopback0
 ip address 1.1.1.1 255.255.255.0

router rip
 version 2
 network 1.0.0.0
 network 192.168.35.0
```

Steg 2 var att filtrera bort 192.168.13.0/24 med en prefix-lista på Jake. Jake

```
ip prefix-list JAKE seq 5 deny 192.168.13.0/24
ip prefix-list JAKE seq 10 permit 0.0.0.0/0 le 32

router ospf 1
 distribute-list prefix JAKE in FastEthernet0/0
```

Tanken är nu att vi fortfarande ska kunna nå exempelvis 1.1.1.0/24 från Berta.

```
Berta#sh ip route | beg Gate
Gateway of last resort is not set
O IA 192.168.12.0/24 \[110/2\] via 192.168.24.2, 00:21:19, FastEthernet0/0
C 192.168.24.0/24 is directly connected, FastEthernet0/0
```

Nope..  Samma problem i Jake:

```
Jake#sh ip route | beg Gate
 Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet0/0
 C 192.168.24.0/24 is directly connected, FastEthernet1/0
```

I Evelyn ser det dock bra ut.

```
Evelyn#sh ip route | beg Gate
Gateway of last resort is not set
R 192.168.12.0/24 \[120/3\] via 192.168.35.3, 00:00:11, Serial0/0
 1.0.0.0/24 is subnetted, 1 subnets
C 1.1.1.0 is directly connected, Loopback0
R 192.168.13.0/24 \[120/3\] via 192.168.35.3, 00:00:11, Serial0/0
R 192.168.24.0/24 \[120/3\] via 192.168.35.3, 00:00:11, Serial0/0
C 192.168.35.0/24 is directly connected, Serial0/0
```

Så vad är då felet? Om vi kollar OSPF-databasen kan vi se att Jake fortfarande får info om 1.1.1.0/24 & 192.168.35.0/24 via Type-5 External LSAs men att "**forward adress**" är 192.168.13.3, men då vi inte har någon route dit blir den ogiltig och installeras ej i FIB.
```
Type-5 AS External Link States
Link ID ADV Router Age Seq# Checksum Tag
1.0.0.0 **192.168.13.1** 966 0x80000001 0x00713D 0
192.168.35.0 **192.168.13.1** 1021 0x80000001 0x004AD8 0
Jake#sh ip ospf database external
OSPF Router with ID (192.168.24.2) (Process ID 1)
Type-5 AS External Link States

LS age: 981
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 1.0.0.0 (External Network Number )
 Advertising Router: 192.168.13.1
 LS Seq Number: 80000001
 Checksum: 0x713D
 Length: 36
 Network Mask: /8
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.13.3**
 External Route Tag: 0

LS age: 1036
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 192.168.35.0 (External Network Number )
 Advertising Router: 192.168.13.1
 LS Seq Number: 80000001
 Checksum: 0x4AD8
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.13.3**
 External Route Tag: 0
```

Hur löser vi då detta? [OSPF Forwarding Address Suppression in Translated Type-5 LSA](http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-sy/iro-15-sy-book/iro-for-add-sup.html) :D

> The OSPF Forwarding Address Suppression in Translated Type-5 LSAs feature causes a not-so-stubby area (NSSA) area border router (ABR) to translate Type-7 link state advertisements (LSAs) to Type-5 LSAs, but use the address 0.0.0.0 for the forwarding address instead of that specified in the Type-7 LSA. This feature causes routers that are configured not to advertise forwarding addresses into the backbone to direct forwarded traffic to the translating NSSA ABRs.

I vår topologi är Charlie ASBR och Alan ABR för vårat NSSA, det blir således i Alan vi ska konfigurera detta.' 

Alan

`area 1 nssa translate type7 suppress-fa`

Kollar vi Jakes database igen ser det nu betydligt bättre ut!

```
Type-5 AS External Link States
Link ID ADV Router Age Seq# Checksum Tag
1.0.0.0 192.168.13.1 53 0x80000002 0x006BBB 0
192.168.35.0 192.168.13.1 53 0x80000002 0x004457 0
Jake#sh ip ospf database external
OSPF Router with ID (192.168.24.2) (Process ID 1)
Type-5 AS External Link States

**Routing Bit Set on this LSA**
 LS age: 67
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 1.0.0.0 (External Network Number )
 Advertising Router: 192.168.13.1
 LS Seq Number: 80000002
 Checksum: 0x6BBB
 Length: 36
 Network Mask: /8
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 0.0.0.0**
 External Route Tag: 0

**Routing Bit Set on this LSA**
 LS age: 67
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 192.168.35.0 (External Network Number )
 Advertising Router: 192.168.13.1
 LS Seq Number: 80000002
 Checksum: 0x4457
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 0.0.0.0**
 External Route Tag: 0
```

Och routing-tabellen för Berta:

```
Berta#sh ip route | beg Gate
Gateway of last resort is not set
O IA 192.168.12.0/24 \[110/2\] via 192.168.24.2, 00:08:56, FastEthernet0/0
O E2 1.0.0.0/8 \[110/20\] via 192.168.24.2, 00:08:56, FastEthernet0/0
C 192.168.24.0/24 is directly connected, FastEthernet0/0
O E2 192.168.35.0/24 \[110/20\] via 192.168.24.2, 00:08:56, FastEthernet0/0
Berta#ping 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 72/129/204 ms
```

Sweet! Artikeln jag länkade till är väldigt läsvärd för att få en lite djupare förståelse om hur detta bör användas.