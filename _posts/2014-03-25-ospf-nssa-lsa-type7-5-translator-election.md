---
title: OSPF - NSSA LSA Type7-5 Translator Election
date: 2014-03-25 10:56
comments: true
categories: [OSPF]
tags: [type 7-5 translation]
---
Forsätter nöta OSPF-labbar från [GNS3 Vault](http://gns3vault.com/OSPF/ospf-nssa-lsa-type-7-to-5-translator-election.html). :) 

![ospfnssa7to5telect](/assets/images/2014/03/ospfnssa7to5telect.png)

GOAL:
-----

*   All IP addresses have been preconfigured for you.
*   Configure OSPF Area 0.
*   Configure OSPF Area 1 as NSSA.
*   Redistribute the loopback0 interface on router Sam into OSPF Area 1.
*   Ensure router Tron is the router performing the translation from LSA Type 7 to Type 5 into area 0

Konfig:
-------

Skippar all grundkonfig då det är väldigt basic. Använde redistribute connected för att få in 4.4.4.4/24-nätet in i OSPF. Kontrollerar vi OSPF-databasen kan vi ses att det just nu är Quorra som utför Type 7->5 translations.

```
Kevin#sh ip ospf database external
OSPF Router with ID (192.168.13.1) (Process ID 1)
Type-5 AS External Link States
Routing Bit Set on this LSA
 LS age: 53
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 4.4.4.0 (External Network Number )
 **Advertising Router: 192.168.34.3**
 LS Seq Number: 80000001
 Checksum: 0x6E08
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 Forward Address: 192.168.34.4
 External Route Tag: 0
```
Hur kommer då detta sig? Om vi kontrollerar OSPF-databasen för både Quorra & Tron ser vi följande: 

Quorra
```
Quorra#sh ip ospf database nssa-external
OSPF Router with ID (192.168.34.3) (Process ID 1)
Type-7 AS External Link States (Area 1)
Routing Bit Set on this LSA
 LS age: 1079
 **Options: (No TOS-capability, Type 7/5 translation, DC)**
 LS Type: AS External Link
 Link State ID: 4.4.4.0 (External Network Number )
 **Advertising Router: 4.4.4.4**
 LS Seq Number: 80000001
 Checksum: 0x6E7C
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.34.4**
 External Route Tag: 0
```
Tron
```
Tron#sh ip ospf database nssa-external
OSPF Router with ID (192.168.24.2) (Process ID 1)
Type-7 AS External Link States (Area 1)
LS age: 1120
 **Options: (No TOS-capability, Type 7/5 translation, DC)**
 LS Type: AS External Link
 Link State ID: 4.4.4.0 (External Network Number )
 **Advertising Router: 4.4.4.4**
 LS Seq Number: 80000001
 Checksum: 0x6E7C
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.34.4**
 External Route Tag: 0
```
Så båda routrarna tar emot 4.4.4.0/24-nätet med information att de ska utföra Type 7 -> 5 translation. Kontrollerar vi istället databasen för Type-5 ser vi att endast Quorra som annonserar detta. Quorra

```
Quorra#sh ip ospf database external
**OSPF Router with ID (192.168.34.3) (Process ID 1)**
Type-5 AS External Link States
LS age: 1252
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 4.4.4.0 (External Network Number )
 **Advertising Router: 192.168.34.3**
 LS Seq Number: 80000001
 Checksum: 0x6E08
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.34.4**
 External Route Tag: 0
```
Tron
```
Tron#sh ip ospf database external
**OSPF Router with ID (192.168.24.2) (Process ID 1)**
Type-5 AS External Link States
Routing Bit Set on this LSA
 LS age: 1316
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 4.4.4.0 (External Network Number )
 **Advertising Router: 192.168.34.3**
 LS Seq Number: 80000001
 Checksum: 0x6E08
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.34.4**
 External Route Tag: 0
```
Varför? Lite efterforskning visade att Ciscos IOS ej har något stöd för att definiera translator direkt i OSPF-processen utan använder sig istället av **högst Router-ID**. När Tron tar emot LSA Type 5-annonseringen från Quorra som har ett högre RID "flushade" den bort sin egen Type-5 LSA och började använda Quorras istället. Vill vi ändra Translator måste vi således ändra Router-ID.
```
Tron(config)#int lo0
Tron(config-if)#ip add 10.10.10.10 255.255.255.255
Tron(config-if)#router ospf 1
Tron(config-router)#router-id 10.10.10.10
Reload or use "clear ip ospf process" command, for this to take effect
Tron(config-router)#do clear ip ospf process
Reset ALL OSPF processes? \[no\]: yes

Quorra(config)#int lo0
Quorra(config-if)#ip add 9.9.9.9 255.255.255.255
Quorra(config-if)#router ospf 1
Quorra(config-router)#router-id 9.9.9.9
Reload or use "clear ip ospf process" command, for this to take effect
Quorra(config-router)#do clear ip ospf process
Reset ALL OSPF processes? \[no\]: yes
```
Kollar vi återigen i Kevin nu kan vi se att Tron har tagit över som Type 7-5 Translator pga ett högre RID. :)

```
Kevin#sh ip ospf database external
OSPF Router with ID (192.168.13.1) (Process ID 1)
Type-5 AS External Link States
Routing Bit Set on this LSA
 LS age: 11
 Options: (No TOS-capability, DC)
 LS Type: AS External Link
 Link State ID: 4.4.4.0 (External Network Number )
 **Advertising Router: 10.10.10.10**
 LS Seq Number: 80000001
 Checksum: 0x4E8E
 Length: 36
 Network Mask: /24
 Metric Type: 2 (Larger than any link state path)
 TOS: 0
 Metric: 20
 **Forward Address: 192.168.34.4**
 External Route Tag: 0
```

Sweet!
