---
title: OSPF - Internal/External Summarization
date: 2013-06-18 18:03
comments: true
categories: [OSPF]
tags: [route summarization]
---
Detta blir ett kortare inlägg om route-summering i OSPF, vilket skiljer sig en hel del från EIGRP's implementering. Vid det här laget ska du väl förhoppningsvis ha järnkoll på de olika LSA-typerna som finns, om inte så är det nog bättre du backar tillbaka och läser tidigare poster om detta [här](http://Jonas Collén.wordpress.com/2013/06/16/ospf-lsa-types/) och [här](http://Jonas Collén.wordpress.com/2013/06/17/ospf-area-types-lsa-type-7/). Det finns vissa regler/saker som är bra att känna till när det kommer till summering innan vi kör igång:

*   En summary-route kommer endast annonseras såvida det finns minst ett aktivt subnät inom specificerat range
*   En summary-route kommer ha costen för det nät med LÄGST cost som faller inom specificerat range.
*   En summary-route kommer pekas till Null0 där vi utför summeringen för att förhindra routingloopar
*   Då OSPF är ett "classless" protokoll kan du använda vilken (giltig) subnätmask som helst

Internal Route Summarization
----------------------------

Då det endast är **LSA Type 3-paket** som informerar andra areas om så kallade "**Interarea Routes (O IA)**" så är det detta paket vi måste modifiera om vi ska kunna använda oss av route summering för just interna nät. Kom ihåg att det **endast är ABR's** som genererar dessa paket, vilket även betyder att det endast är möjligt att utföra route-summering i just **ABR's**. Lås oss använda följande topologi för att testa detta i praktiken. 

[![OSPF Summary topologi](/assets/images/2013/06/ospf-summary-topologi.png)](/assets/images/2013/06/ospf-summary-topologi.png) 

I ovanstående exempel kan vi se att R2 och R4 är ABR's och det är således endast där vi har möjlighet att summera. I area 20 finns det ju ingen anledning men i area 10 finns det helt klart fördelar att göra detta. För vad skulle hända i nuläget om vi säger att nätet 20.0.2.0/24 skulle gå ner? En Linkstate Update skulle genereras från R1 som sedan sprids till alla andra areas i nätet (inkl. R5, NSSA filtrerar endast LSA 5), och samtliga routrar skulle sedan behöva köra SPF-algoritmen igen. Detta är förutom att det är väldigt resurskrävande rätt onödigt, vi vet ju redan om att alla 20.0.x.0-nät ligger i area 10. Om vi istället skulle annonsera ut 20.0.0.0/22 från R2 så spelar det ingen roll om 20.0.2.0/24 skulle gå ner, övriga areas behöver inte ha någon update om detta och slipper således även köra sin SPF-algoritm igen! Innan vi börjar, här är en sh ip route från R1, R3 & R5:
```
R1
Gateway of last resort is not set
20.0.0.0/24 is subnetted, 4 subnets
C 20.0.0.0 is directly connected, Loopback0
C 20.0.1.0 is directly connected, Loopback1
C 20.0.2.0 is directly connected, Loopback2
C 20.0.3.0 is directly connected, Loopback3
C 172.16.0.0/16 is directly connected, FastEthernet0/0
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/20] via 172.16.0.2, 01:25:46, FastEthernet0/0
O IA 10.0.0.128 [110/30] via 172.16.0.2, 01:25:46, FastEthernet0/0
O IA 192.168.0.0/24 [110/40] via 172.16.0.2, 01:25:37, FastEthernet0/0
 30.0.0.0/24 is subnetted, 2 subnets
O E2 30.0.0.0 [110/20] via 172.16.0.2, 01:25:27, FastEthernet0/0
O E2 30.0.1.0 [110/20] via 172.16.0.2, 01:25:27, FastEthernet0/0 
R3
Gateway of last resort is not set
20.0.0.0/32 is subnetted, 4 subnets
O IA 20.0.1.1 [110/21] via 10.0.0.1, 00:18:25, FastEthernet0/1
O IA 20.0.0.1 [110/21] via 10.0.0.1, 00:18:25, FastEthernet0/1
O IA 20.0.3.1 [110/21] via 10.0.0.1, 00:18:25, FastEthernet0/1
O IA 20.0.2.1 [110/21] via 10.0.0.1, 00:18:25, FastEthernet0/1
O IA 172.16.0.0/16 [110/20] via 10.0.0.1, 01:26:03, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
C 10.0.0.0 is directly connected, FastEthernet0/1
C 10.0.0.128 is directly connected, FastEthernet0/0
O IA 192.168.0.0/24 [110/20] via 10.0.0.130, 01:25:54, FastEthernet0/0
 30.0.0.0/24 is subnetted, 2 subnets
O E2 30.0.0.0 [110/20] via 10.0.0.130, 01:25:54, FastEthernet0/0
O E2 30.0.1.0 [110/20] via 10.0.0.130, 01:25:54, FastEthernet0/0
R5
Gateway of last resort is 192.168.0.1 to network 0.0.0.0
20.0.0.0/32 is subnetted, 4 subnets
O IA 20.0.1.1 [110/41] via 192.168.0.1, 00:11:08, FastEthernet0/1
O IA 20.0.0.1 [110/41] via 192.168.0.1, 00:11:08, FastEthernet0/1
O IA 20.0.3.1 [110/41] via 192.168.0.1, 00:11:08, FastEthernet0/1
O IA 20.0.2.1 [110/41] via 192.168.0.1, 00:11:08, FastEthernet0/1
O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 00:11:08, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/30] via 192.168.0.1, 00:11:08, FastEthernet0/1
O IA 10.0.0.128 [110/20] via 192.168.0.1, 00:11:09, FastEthernet0/1
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
O*N2 0.0.0.0/0 [110/1] via 192.168.0.1, 00:11:05, FastEthernet0/1
```
Då det är 20.0.x.0-nätet vi vill summera så är det bevisligen ABR för area 10 vi ska jobba med - R2 i detta fall.
```
router ospf 1
 area 10 range 20.0.0.0 255.255.252.0
```
Observera att vi nu måste använda **subnätmask** istället för wildcard, way to keep it simple cisco.. ;P Kollar vi nu sh ip route i R2 kan vi se att den summerade routen nu pekar mot Null0.
```
 20.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
O 20.0.1.1/32 [110/11] via 172.16.0.1, 00:00:22, FastEthernet0/0
**O 20.0.0.0/22 is a summary, 00:00:22, Null0**
O 20.0.0.1/32 [110/11] via 172.16.0.1, 00:00:22, FastEthernet0/0
O 20.0.3.1/32 [110/11] via 172.16.0.1, 00:00:22, FastEthernet0/0
O 20.0.2.1/32 [110/11] via 172.16.0.1, 00:00:22, FastEthernet0/0
C 172.16.0.0/16 is directly connected, FastEthernet0/0
 10.0.0.0/25 is subnetted, 2 subnets
C 10.0.0.0 is directly connected, FastEthernet0/1
O 10.0.0.128 [110/20] via 10.0.0.2, 00:00:23, FastEthernet0/1
O IA 192.168.0.0/24 [110/30] via 10.0.0.2, 00:00:23, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
O E2 30.0.0.0 [110/20] via 10.0.0.2, 00:00:23, FastEthernet0/1
O E2 30.0.1.0 [110/20] via 10.0.0.2, 00:00:30, FastEthernet0/1
```
Om du är osäker varför paketen inte "blackhole"-routas trots detta -  repetera "longest match" från CCNA. Låt oss se hur det ser ut för R5:
```
Gateway of last resort is 192.168.0.1 to network 0.0.0.0
**20.0.0.0/22 is subnetted, 1 subnets**
**O IA 20.0.0.0 [110/41] via 192.168.0.1, 00:00:03, FastEthernet0/1**
O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 01:29:12, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/30] via 192.168.0.1, 01:29:12, FastEthernet0/1
O IA 10.0.0.128 [110/20] via 192.168.0.1, 01:29:12, FastEthernet0/1
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
O*N2 0.0.0.0/0 [110/1] via 192.168.0.1, 01:29:08, FastEthernet0/1
```
Stabilt! Denna förändring gäller ju självklart även för area 0. I Wireshark kan vi se följande två paket skickas: 

[![OSPF Summary LSA](/assets/images/2013/06/ospf-summary-lsa.png)](/assets/images/2013/06/ospf-summary-lsa.png) 

Detta är väl precis vad vi förväntat oss, ett LSA Type 3-paket som annonserar 20.0.0.0/22 med cost 31. Dock känner ju R5 fortfarande till även de specifika näten, OSPF löser detta genom följande paket som skickas samtidigt. 

[![OSPF Summary LSA 2](/assets/images/2013/06/ospf-summary-lsa-2.png)](/assets/images/2013/06/ospf-summary-lsa-2.png) 

Observera metric:en - vilket är det maximala cost-värdet en länk kan ha, den räknas därför som **Unreachable** och tas bort från routing tabellen. Se likheten med RIP exempelvis, där advertisas istället hopcount till 16.

External Route Summarization
----------------------------

[![OSPF Summary topologi](/assets/images/2013/06/ospf-summary-topologi.png)](/assets/images/2013/06/ospf-summary-topologi.png) 

Om vi använder oss av samma topologi som tidigare kan vi se att R5 agerar ASBR och annonserar 30.0.x.0/24 som **LSA Type 7 (nssa)**, vilket sen **ABR (R4) gör om till LSA Type 5** och sprider till övriga areas. En show ip route i R1 visar följande:
```
Gateway of last resort is not set
20.0.0.0/24 is subnetted, 4 subnets
C 20.0.0.0 is directly connected, Loopback0
C 20.0.1.0 is directly connected, Loopback1
C 20.0.2.0 is directly connected, Loopback2
C 20.0.3.0 is directly connected, Loopback3
C 172.16.0.0/16 is directly connected, FastEthernet0/0
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/20] via 172.16.0.2, 01:25:46, FastEthernet0/0
O IA 10.0.0.128 [110/30] via 172.16.0.2, 01:25:46, FastEthernet0/0
O IA 192.168.0.0/24 [110/40] via 172.16.0.2, 01:25:37, FastEthernet0/0
 30.0.0.0/24 is subnetted, 2 subnets
**O E2 30.0.0.0 [110/20] via 172.16.0.2, 01:25:27, FastEthernet0/0**
**O E2 30.0.1.0 [110/20] via 172.16.0.2, 01:25:27, FastEthernet0/0**
```
Om vi vill använda oss av route summering för externa routes (LSA Type 5/7) - så utför vi till skillnad från Internal Routes detta direkt på **ASBR (gäller även när vi använder oss av NSSA/ T NSSA)**. Kommandot skiljer sig dock lite:
```
router ospf 1
summary-address 30.0.0.0 255.255.254.0
```
I R5 skapas en Null0 summary-route precis som när vi gjorde detta för interna routes i föregående exempel:
```
20.0.0.0/22 is subnetted, 1 subnets
O IA 20.0.0.0 [110/41] via 192.168.0.1, 00:31:25, FastEthernet0/1
O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 02:15:42, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/30] via 192.168.0.1, 02:15:42, FastEthernet0/1
O IA 10.0.0.128 [110/20] via 192.168.0.1, 02:15:42, FastEthernet0/1
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
S 30.0.0.0/24 is directly connected, Null0
**O 30.0.0.0/23 is a summary, 00:15:07, Null0**
S 30.0.1.0/24 is directly connected, Null0
```
Och för R1 har vi nu endast en route till 30.0.0.0/24 & 30.0.1.0/24:
```
Gateway of last resort is not set
20.0.0.0/24 is subnetted, 4 subnets
C 20.0.0.0 is directly connected, Loopback0
C 20.0.1.0 is directly connected, Loopback1
C 20.0.2.0 is directly connected, Loopback2
C 20.0.3.0 is directly connected, Loopback3
C 172.16.0.0/16 is directly connected, FastEthernet0/0
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/20] via 172.16.0.2, 03:25:44, FastEthernet0/0
O IA 10.0.0.128 [110/30] via 172.16.0.2, 03:25:44, FastEthernet0/0
O IA 192.168.0.0/24 [110/40] via 172.16.0.2, 03:25:35, FastEthernet0/0
 **30.0.0.0/23 is subnetted, 1 subnets**
**O E2 30.0.0.0 [110/20] via 172.16.0.2, 00:09:55, FastEthernet0/0**
```
I Wireshark ser vi samma mönster som tidigare: 
[![LSA Type 7 summary](/assets/images/2013/06/lsa-type-7-summary.png)](/assets/images/2013/06/lsa-type-7-summary.png)

Först en route som pekar till 30.0.0.0/23 med cost 20, och sedan ett paket med maxad metric för de två inviduella routes'en som gör dessa "unreachable": 
[![LSA Type 7 summary 2](/assets/images/2013/06/lsa-type-7-summary-2.png)](/assets/images/2013/06/lsa-type-7-summary-2.png) 

Inte speciellt svårt, gäller bara att hålla isär att **LSA Type 3 summeras i ABR - LSA Type 5 /7 summeras i ASBR** (och att kommandona skiljer sig lite åt för att utföra detta). Det var allt jag hade om summering!
