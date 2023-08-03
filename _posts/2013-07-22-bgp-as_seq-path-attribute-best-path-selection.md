---
title: BGP - AS_SEQ Path Attribute & Best Path Selection
date: 2013-07-22 23:47
comments: true
categories: [BGP]
tags: [as_seq, path selection]
---
AS_SEQ Path
------------

Per default använder sig BGP av AS_Path attributet "AS_SEQ" (_Autonomous System Sequence_) för att bestämma vilken route som är bäst, men även för att förhindra eventuella routing loopar. AS_Path listar vilka AS som trafiken kommer gå via för att nå sin destination, BGP kommer då att välja den route som går via **lägst** antal AS. Detta kan komiskt nog liknas vid RIP som använder hopcount för att räkna ut den bästa vägen. Låt oss använda följande topologi för att testa detta lite mer ingående: 
![bgp AS-path](/assets/images/2013/07/bgp-as-path.png) 

Vi tar och kollar vidare på nätet 192.168.0.0/24. I R5 kan vi se följande:
```
R5#sh ip route | beg Gat
 Gateway of last resort is not set
C 200.0.0.0/24 is directly connected, Loopback1
 5.0.0.0/24 is subnetted, 1 subnets
 C 5.5.5.0 is directly connected, Loopback0
 C 200.0.1.0/24 is directly connected, Loopback2
 6.0.0.0/24 is subnetted, 1 subnets
 B 6.6.6.0 [20/0] via 172.16.51.1, 01:03:52
 C 200.0.2.0/24 is directly connected, Loopback3
 172.16.0.0/24 is subnetted, 1 subnets
 C 172.16.51.0 is directly connected, FastEthernet0/0
 C 200.0.3.0/24 is directly connected, Loopback4
 **B 192.168.0.0/24 [20/0] via 172.16.51.1, 00:16:32**
 B 200.0.0.0/22 [200/0] via 0.0.0.0, 00:52:34, Nu
R5#sh ip bgp
 BGP table version is 13, local router ID is 5.5.5.5
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight **Path**
 *> 5.5.5.0/24 0.0.0.0 0 32768 i
 *> 6.6.6.0/24 172.16.51.1 0 500 200 i
 ***> 192.168.0.0 172.16.51.1 0 500 200 50 i**
 s> 200.0.0.0 0.0.0.0 0 32768 i
 *> 200.0.0.0/22 0.0.0.0 32768 i
 s> 200.0.1.0 0.0.0.0 0 32768 i
 s> 200.0.2.0 0.0.0.0 0 32768 i
 s> 200.0.3.0 0.0.0.0 0 32768 i
```
Observera next-hop adressen, R1 annonserar sin egen adress då det är ett eBGP-förhållande mellan R1 <-> R5.  Vi kan också se att för R5 skall nå destinationen 192.168.0.0 kommer den behöva gå via Path: 500 -> 200 -> 50. Detta känns ju dock ej som den kortaste vägen när trafiken går via AS 200 också? Gör vi samma sak på R4 ser det ju dock bättre ut:
```
R4#sh ip bgp
 BGP table version is 35, local router ID is 4.4.4.4
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
 *>i5.5.5.0/24 1.1.1.1 0 100 0 100 i
 * 6.6.6.0/24 172.16.47.7 0 50 200 i
 *>i 3.3.3.3 0 100 0 200 i
 ***> 192.168.0.0 172.16.47.7 0 0 50 i**
 r>i200.0.0.0/22 1.1.1.1 0 100 0 100 i
```
Varför väljer då R5 att ta omvägen?
```
R1#sh ip bgp 192.168.0.0
 BGP routing table entry for 192.168.0.0/24, version 4
 Paths: (2 available, best #1, table Default-IP-Routing-Table)
 Flag: 0x820
 Advertised to update-groups:
 1
 200 50
 3.3.3.3 (metric 129) from 3.3.3.3 (3.3.3.3)
 Origin IGP, metric 0, localpref 100, valid, internal, best
 50
 **172.16.47.7 (inaccessible) from 4.4.4.4 (4.4.4.4)**
 Origin IGP, metric 0, localpref 100, valid, internal
```
Det visade sig att jag glömt annonsera nätet 172.16.47.0/24  i R4 till OSPF-processen, BGP använder endast routes den kan nå. Efter jag fixat detta får vi nu förväntat resultat på R5:
```
R5#sh ip bgp
 BGP table version is 22, local router ID is 5.5.5.5
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
 *> 5.5.5.0/24 0.0.0.0 0 32768 i
 *> 6.6.6.0/24 172.16.51.1 0 500 200 i
 ***> 192.168.0.0 172.16.51.1 0 500 50 i**
 s> 200.0.0.0 0.0.0.0 0 32768 i
 *> 200.0.0.0/22 0.0.0.0 32768 i
 s> 200.0.1.0 0.0.0.0 0 32768 i
 s> 200.0.2.0 0.0.0.0 0 32768 i
 s> 200.0.3.0 0.0.0.0 0 32768 i
```
Om vi även kollar R3's tabell kan vi se att den har två routes till 192.168.0.0/24-nätet, men att den väljer den med kortast AS_PATH.
```
R3#sh ip bgp
 BGP table version is 38, local router ID is 3.3.3.3
 Status codes: s suppressed, d damped, h history, * **valid, > best, i - internal**,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
 *>i5.5.5.0/24 1.1.1.1 0 100 0 100 i
 *> 6.6.6.0/24 172.16.36.6 0 0 200 i
 *** 192.168.0.0 172.16.36.6 0 200 50 i**
 ***>i 172.16.47.7 0 100 0 50 i**
 r>i200.0.0.0/22 1.1.1.1 0 100 0 100 i
Routing entry for 192.168.0.0/24
 Known via "bgp 500", distance 200, metric 0
 Tag 50, type internal
 Last update from 172.16.47.7 00:24:41 ago
 Routing Descriptor Blocks:
 * 172.16.47.7, from 4.4.4.4, 00:24:41 ago
 Route metric is 0, traffic share count is 1
 AS Hops 1
 Route tag 50
```
En viktig skillnad mellan iBGP och eBGP är förövrigt att iBGP ej lägger till sitt AS i AS_PATH. Varför? BGP's "loop prevention" innebär att routern alltid inspekterar AS_PATH för nya routes, och om den finner att sitt eget AS-nummer redan finns med så ignoreras routen (en bättre väg måste bevisligen redan finnas). Hur vi löser loop prevention i iBGP har jag redan nämnt i det tidigare inlägget  "[BGP - Internal BGP & Transitarea](http://www.jonascollen.se/posts/bgp-internal-bgp-transitarea/)" (full mesh).

Best Path Selection
-------------------

Som vi sett i ovanstående exempel, lämnar vi BGP med default-inställningar kommer den välja routen med kortast AS-PATH. Det ingår dock betydligt fler variabler där routern går i turordning tills den hittar den bästa kandidaten.

1.  Largest Weight
2.  Highest Local Preference
3.  Locally originated
4.  Shortest AS-Path
5.  Lowest origin type (i < e < ?)
6.  Lowest MED (metric)
7.  eBGP > iBGP
8.   Lowest IGP metric to neighbor (max. match check)
9.  Older route
10.  Lowest Router-ID

Det är med andra ord lite mer än vad vi är vana med från OSPF/EIGRP. ;) Det är somliga av dessa värden vi senare kommer att modifiera för att ha möjlighet att påverka vilken väg vi önskar att trafiken skall ta.

