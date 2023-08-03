---
title: OSPF - Discard Route
date: 2014-03-24 14:56
comments: true
categories: [OSPF]
tags: [discard route]
---
Tänkte roa mig med att göra lite OSPF-labbar från GNS3 Vault, först ut var denna om [Summarization Discard Route](http://gns3vault.com/OSPF/ospf-summarization-discard-route.html). ![ospfsummarizationdiscardroute](/assets/images/2014/03/ospfsummarizationdiscardroute.png)

Goal:
-----

*   All IP addresses have been preconfigured for you.
*   Configure OSPF and use the correct areas.
*   Configure a loopback0 interface on router Spielburg with network address 3.3.3.3 /24.
*   Configure router Shapeir to summarize network 3.3.3.0 /24 to 3.0.0.0 /8.
*   Configure router Shapeir so it doesn't add a null0 entry in its routing table.

Konfig:
-------

Basic config på samtliga enheter, summera nät kan vi som bekant endast göra på ABR. För interna nät används area x range, E1/E2 använder summary-address direkt på ASBR. 

```
Tarna

router ospf 1
 log-adjacency-changes
 network 192.168.12.0 0.0.0.255 area 0

Shapeir

router ospf 1
  log-adjacency-changes 
  area 1 range 3.0.0.0 255.0.0.0
  network 192.168.12.0 0.0.0.255 area 0
  network 192.168.23.0 0.0.0.255 area 1

Spielburg

interface Loopback0
ip address 3.3.3.3 255.255.255.0
router ospf 1
log-adjacency-changes
network 3.3.3.0 0.0.0.255 area 1
network 192.168.23.0 0.0.0.255 area 1
```

Vilket ger följande resultat i Tarna:

```
Tarna#sh ip route | beg Gate
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet0/0
O IA 3.0.0.0/8 \[110/3\] via 192.168.12.2, 00:24:49, FastEthernet0/0
O IA 192.168.23.0/24 \[110/2\] via 192.168.12.2, 00:32:18, FastEthernet0/0
```

Och i Shapeir:

```
Shapeir#sh ip route | beg Gate
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
 3.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O 3.3.3.3/32 \[110/2\] via 192.168.23.3, 00:00:23, FastEthernet0/0
**O 3.0.0.0/8 is a summary, 00:00:23, Null0**
C 192.168.23.0/24 is directly connected, FastEthernet0/0
```

Det var null0-routen vi ej fick ha.. Här körde jag fast rätt rejält, en lösning hade ju kunnat vara att göra en statisk route på Speilberg och sedan redistributa in den i OSPF, men enligt labben skulle konfigändringen göras i Shapeir.. Titeln på labben spoilade dock lite, följande information fanns att läsa på Cisco's OSPF Commands:

discard-route
-------------

To reinstall either an external or internal discard route that was previously removed, use the discard**\-route** command in router configuration mode. To remove either an external or internal discard route, use the **no** form of this command. **discard-route** \[**external** | **internal**\] no **discard-route** \[**external** | **internal**\]

### Syntax Description


external

(Optional) Reinstalls the discard route entry for redistributed summarized routes on an Autonomous System Boundary Router (ASBR).

internal

(Optional) Reinstalls the discard-route entry for summarized internal routes on the Area Border Router (ABR).

Hemligheten ligger i "_**To remove either an external or internal discard route, use the no form of this command.**_" Testade helt enkelt med "no discard-route internal" och vips så var null0-routen borta. :)

```
router ospf 1
no discard-route internal
Shapeir#sh ip route | beg Gate
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
 3.0.0.0/32 is subnetted, 1 subnets
O 3.3.3.3 \[110/2\] via 192.168.23.3, 00:18:12, FastEthernet0/0
C 192.168.23.0/24 is directly connected, FastEthernet0/0
```

Vet inte om jag ser så mycket praktiskt nytta att använda detta dock. :P