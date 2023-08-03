---
title: Redistribution - Labs
date: 2013-07-04 20:29
comments: true
categories: [EIGRP, OSPF, RIP, Route Manipulation]
tags: [route tagging]
---
Har tillbringat eftermiddagen med att göra GNS3 Vaults Redistribute-labbar. [RIP to EIGRP redistribution](http://gns3vault.com/Redistribution/rip-to-eigrp-redistribution.html) [RIP to OSPF redistribution](http://gns3vault.com/Redistribution/rip-to-ospf-redistribution.html) Dessa två var väldigt enkla, inget speciellt att tänka på.. Men denna var rätt intressant:

One-Way Redistribution
----------------------

[![Oneway](/assets/images/2013/07/oneway.png)](/assets/images/2013/07/oneway.png)

*   All IP addresses have been preconfigured for you.
*   Configure OSPF on router Ace and Aggie and only advertise network 192.168.12.0 /24.
*   Configure EIGRP AS 1 on router Ace, Aggie and Abu. Only advertise network 192.168.13.0 /24 and 192.168.23.0 /24.
*   Redistribute the loopback0 interface in EIGRP AS 1 on router Abu.
*   Redistribute EIGRP information into OSPF on router Ace.
*   Do a traceroute from router Aggi or Ace to network 3.3.3.0 /24. You notice that you are not using the most optimal path...fix this problem so router Aggie uses the most optimal path.

Det var den sista punkten som ställde till det. Kontrollerar vi routing table på Aggie kan vi se följande:
```
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
D 192.168.13.0/24 [90/30720] via 192.168.23.3, 00:03:09, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
**O E2 3.3.3.0 [110/500] via 192.168.12.1, 00:00:08, FastEthernet1/0**
C 192.168.23.0/24 is directly connected, FastEthernet0/0
```
Aggie väljer helt enkelt att gå via Ace för att komma till 3.3.3.3/24 istället för att gå direkt ner till Abu via fa0/0. En show topology table för EIGRP visar dock att vi ju faktiskt känner till "den bättre vägen" men trots detta väljer att gå via Ace.
```
Aggie#sh ip eigrp topology 
IP-EIGRP Topology Table for AS(1)/ID(192.168.23.2)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
 r - reply Status, s - sia Status
**P 3.3.3.0/24, 0 successors, FD is Inaccessible**
 **via 192.168.23.3 (1709312/1706752), FastEthernet0/0**
P 192.168.13.0/24, 1 successors, FD is 30720
 via 192.168.23.3 (30720/28160), FastEthernet0/0
P 192.168.23.0/24, 1 successors, FD is 28160
 via Connected, FastEthernet0/0
```
Varför? EIGRP External-routes har en AD av 170, OSPF's E1/E2 har 110! Lösningen är helt enkelt att modifiera Aggie's AD, exempelvis genom att höja OSPF's external över 170.
```
router ospf 1
distance ospf external 200
```
Kollar vi återigen routing table kan vi nu se följande:
```
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
D 192.168.13.0/24 [90/30720] via 192.168.23.3, 00:10:07, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
**D EX 3.3.3.0 [170/1709312] via 192.168.23.3, 00:00:01, FastEthernet0/0**
C 192.168.23.0/24 is directly connected, FastEthernet0/0
```
Labben hittar du [här](http://gns3vault.com/Redistribution/multipoint-one-way-redistribution.html).

Multipoint One-way Redistribution
---------------------------------

[![Oneway](/assets/images/2013/07/oneway.png)](/assets/images/2013/07/oneway.png) 
Uppgiften är egentligen densamma som vi hade tidigare, skillnaden är att vi nu ska redistributa EIGRP-informationen på både Ace & Aggie in till OSPF, tidigare gjorde vi detta endast på Ace. Vilka problem kan detta leda till? Routing table för Ace ser bra ut:
```
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
C 192.168.13.0/24 is directly connected, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
D EX 3.3.3.0 [170/1709312] via 192.168.13.3, 00:32:00, FastEthernet0/0
D 192.168.23.0/24 [90/30720] via 192.168.13.3, 00:34:03, FastEthernet0/0
```
Men för Aggie är det samma problem som innan:
```
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
D 192.168.13.0/24 [90/30720] via 192.168.23.3, 00:33:31, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
O E2 3.3.3.0 [110/500] via 192.168.12.1, 00:00:01, FastEthernet1/0
C 192.168.23.0/24 is directly connected, FastEthernet0/0
```
Vi gör återigen samma lösning på Aggie;
```
router ospf 1
distance ospf external 200
```
Resultatet blir:
```
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
D 192.168.13.0/24 [90/30720] via 192.168.23.3, 00:38:07, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
D EX 3.3.3.0 [170/1709312] via 192.168.23.3, 00:00:02, FastEthernet0/0
C 192.168.23.0/24 is directly connected, FastEthernet0/0
```
Men om vi nu kontrollerar routing-tabel för Ace så har något hänt..
```
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet1/0
C 192.168.13.0/24 is directly connected, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
**O E2 3.3.3.0 [110/500] via 192.168.12.2, 00:00:08, FastEthernet1/0**
D 192.168.23.0/24 [90/30720] via 192.168.13.3, 00:38:14, FastEthernet0/0
```

Vafan! Kom ihåg att vi endast kan gäller lokalt för routern vi ändrar på, Ace anser därför nu att den har en bättre väg till 3.3.3.0/24-nätet via Aggie istället. Vi är med andra ord tvungna att lägga in "**distance ospf external 200**" även på Ace. Efter detta är allt frid och fröjd igen.. :) Även om det är väldigt enkelt just nu så är det ju bara att sätta sig in i vilka märkliga problem & routing-loopar detta kan leda till i större topologier. Labben hittar du [här](http://gns3vault.com/Redistribution/multipoint-one-way-redistribution-v15-239.html).

RIP to EIGRP Two-Way Redistribution & Route Tagging
---------------------------------------------------

[![routetagg redist](/assets/images/2013/07/routetagg-redist.png)](/assets/images/2013/07/routetagg-redist.png)

*   All IP addresses have been preconfigured for you.
*   RIP and EIGRP have been preconfigred for you on the corresponding routers.
*   Enable two-way redistribution between RIP and EIGRP on router Lassie and Willy.
*   You notice that router Willy or Lassie are sending traffic to 4.4.4.4 towards router Flipper, make sure you get rid of this sub-optimal routing.
*   Use route tagging to accomplish this.

Precis som labben säger tror Willy att bästa vägen för att nå 4.4.4.4 är via Flipper(!) efter vi lagt in grundkonfig.
```
Gateway of last resort is not set
R 192.168.12.0/24 [120/1] via 192.168.13.1, 00:00:08, FastEthernet0/0
 1.0.0.0/24 is subnetted, 1 subnets
R 1.1.1.0 [120/1] via 192.168.13.1, 00:00:08, FastEthernet0/0
C 192.168.13.0/24 is directly connected, FastEthernet0/0
 4.0.0.0/24 is subnetted, 1 subnets
**R 4.4.4.0 [120/6] via 192.168.13.1, 00:00:08, FastEthernet0/0**
D 192.168.24.0/24 [90/30720] via 192.168.34.4, 00:03:40, FastEthernet1/0
C 192.168.34.0/24 is directly connected, FastEthernet1/0
```

Varför? Jo precis som tidigare, RIP har en AD på 120, EIGRP External har 170, Willy kommer med andra ord alltid föredra att gå via Flipper istället för direkt ner till Skippy. Inte helt optimalt direkt.. Den här gången fick vi dock inte modifiera AD-värdet utan ska istället använda oss av route tagging, nice! Om vi börjar med RIP -> EIGRP, så behövs först en access-lista som definierar vilka routes vi vill tagga (RIP):
```
ip access-list standard RIP
 permit 1.1.1.1
 permit 192.168.12.0 0.0.0.255
 permit 192.168.13.0 0.0.0.255
```
Vi bygger sedan en route-map som märker dessa med tag 10, och blockerar routes märkte med tag 5 (fixar vi senare):
```
route-map RIP-INTO-EIGRP deny 5
 match tag 5

route-map RIP-INTO-EIGRP permit 10
 match ip address RIP
 set tag 10
```
Och samma sak för EIGRP -> RIP:
```
ip access-list standard EIGRP
 permit 4.4.4.4
 permit 192.168.24.0 0.0.0.255
 permit 192.168.34.0 0.0.0.255
route-map EIGRP-INTO-RIP deny 5
 match tag 10

route-map EIGRP-INTO-RIP permit 10
 match ip address EIGRP
 set tag 5
```
En show ip route för Lassie & Willie ser nu ut enligt följande:

Willy
```
Gateway of last resort is not set
R 192.168.12.0/24 [120/1] via 192.168.13.1, 00:00:20, FastEthernet0/0
 1.0.0.0/24 is subnetted, 1 subnets
R 1.1.1.0 [120/1] via 192.168.13.1, 00:00:20, FastEthernet0/0
C 192.168.13.0/24 is directly connected, FastEthernet0/0
 4.0.0.0/24 is subnetted, 1 subnets
**D EX 4.4.4.0 [170/156160] via 192.168.34.4, 00:09:53, FastEthernet1/0**
D 192.168.24.0/24 [90/30720] via 192.168.34.4, 00:09:53, FastEthernet1/0
C 192.168.34.0/24 is directly connected, FastEthernet1/0
Lassie
Gateway of last resort is not set
C 192.168.12.0/24 is directly connected, FastEthernet0/0
 1.0.0.0/24 is subnetted, 1 subnets
R 1.1.1.0 [120/1] via 192.168.12.1, 00:00:06, FastEthernet0/0
R 192.168.13.0/24 [120/1] via 192.168.12.1, 00:00:06, FastEthernet0/0
 4.0.0.0/24 is subnetted, 1 subnets
**D EX 4.4.4.0 [170/156160] via 192.168.24.4, 00:04:00, FastEthernet1/0**
C 192.168.24.0/24 is directly connected, FastEthernet1/0
D 192.168.34.0/24 [90/30720] via 192.168.24.4, 00:04:00, FastEthernet1/0
```
Sådär ja! Vill du också göra denna labb så finns den [här](http://gns3vault.com/Redistribution/rip-eigrp-redistribution-route-tagging.html). Ska brygga mig en stor kanna kaffe och sätta mig med advanced distribution-labben gjord av [Petr Lapukhov](http://blog.ine.com/author/petr/) (CCIEx4 :D). Den är dock främst riktad mot de som studerar inför CCIE-Lab men rekommenderades även för de som verkligen ville gå in på djupet under sina ccnp-studier, och det vill vi ju givetvis! Det får dock bli en egen post om jag nu lyckas. ;)
