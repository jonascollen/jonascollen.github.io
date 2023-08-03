---
title: OSPF - The Basics II
date: 2013-06-12 23:00
comments: true
categories: [OSPF]
---
Designated Router / Backup Designated Router
--------------------------------------------

Om vi tänker tillbaka till den föregående posten, OSPF - The Basics, så kommer du förhoppningsvis ihåg hur omständigt det var att sätta upp neighbor adjacencys mellan två routrar, och alla paket som skickades fram och tillbaka för att lyckas med detta (Hello/DBD/LSR/LSU osv). Tänk då hur det kommer bli för följande topologi: 

[![ospf-sharednetwork](/assets/images/2013/06/ospf-sharednetwork1.png)](/assets/images/2013/06/ospf-sharednetwork1.png)

Även efter adjacencys är upprättade så kommer en router, så fort den får en uppdatering (antingen om ett eget nät eller från någon annan router) annonsera ut detta till övriga på nätet. Så kommer det fortsätta för varje router - LSA-storm delux! :) 
[![lsa storm](/assets/images/2013/06/ospf-sharednetwork2.png)](/assets/images/2013/06/ospf-sharednetwork2.png)

Det här problemet uppstår endast när vi har ett "Shared network segment" som ovan, för "Point-to-Point" så kan det ju endast finnas en neighbor. För att komma tillrätta med detta problem väljer routrarna istället ut en specifik  "ledare", **Designated Router (DR)****,** som de sätter upp sin adjacency mot och sedan utbyter all routing-information med. Det är denna router som i sin tur är ansvarig för att hålla resterande enheter uppdaterade med routing-information. För att vara på den säkra sidan om denna DR-router skulle gå ner väljs det även ut en **Backup Designated Router (BDR).** Låt oss säga att R6 blir vald till DR och R9 till BDR, neighbor adjacencys skulle då istället se ut såhär: 

[![DR](/assets/images/2013/06/dr.png)](/assets/images/2013/06/dr.png)

Övriga routrar stannar kvar i "Steg 5" från [OSPF - The Basics](http://www.jonascollen.se/posts/ospf-the-basics/), **2-way state**, mellan varandra och bildar ej en full neighbor adjacency. Om DR-routern går offline kommer BDR-routern att uppgradera sig till DR och en annan router kommer väljas till BDR. Hur väljs då BD & BDR? Följande parametrar bestämmer vilken router som kommer väljas:

*   Routern med den **högsta priorityn** blir **DR**
*   Routern med den **näst högsta priorityn** blir **BDR**
*   Om priorityn är lika (default: 1), så är det istället den med **högst/näst högst Router-ID** som blir **DR/BDR**
*   För att välja en ny DR/BDR måste OSPF-processen alternativt hela routern startas om
*   Routrar som ej väljs till DR/BDR visas som **DROTHER**

För att justera en routers priority används kommandot "**ip ospf priority x**" på det aktuella interfacet, om vi sätter priority till 0 (det lägsta möjliga värdet), stoppar vi routern från att kunna bli varken DR eller BDR (även om det så bara finns två routrar på segmentet). Hur ser då förloppet ut när BD & BDR väljs? Vi kan kontrollera detta genom kommandot **debug ip ospf adj.**

```
*Mar 1 00:04:12.227: OSPF: Interface FastEthernet0/0 going Up
*Mar 1 00:04:12.255: OSPF: 2 Way Communication to 2.2.2.2 on FastEthernet0/0, state 2WAY
*Mar 1 00:04:12.259: OSPF: Backup seen Event before WAIT timer on FastEthernet0/0
*Mar 1 00:04:12.259: OSPF: DR/BDR election on FastEthernet0/0 
*Mar 1 00:04:12.259: OSPF: Elect BDR 1.1.1.1
*Mar 1 00:04:12.259: OSPF: Elect DR 2.2.2.2
*Mar 1 00:04:12.259: OSPF: Elect BDR 1.1.1.1
*Mar 1 00:04:12.263: OSPF: Elect DR 2.2.2.2
*Mar 1 00:04:12.263: DR: 2.2.2.2 (Id) BDR: 1.1.1.1 (Id)
```
Vi kan kontrollera vem som valts till vad genom **sh ip ospf neighbor.**
```
R1#sh ip ospf neighbor
Neighbor ID Pri State Dead Time Address Interface
 2.2.2.2 1 FULL/DR 00:00:35 10.0.0.2 FastEthernet0/0
 3.3.3.3 1 FULL/DROTHER 00:00:39 10.0.0.3 FastEthernet0/0
```
Och ytterligare info kan fås genom **show ip ospf interface fa0/0**.
```
FastEthernet0/0 is up, line protocol is up 
 Internet Address 10.0.0.1/24, Area 0 
 Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 10
 Transmit Delay is 1 sec, **State BDR, Priority 1**
 **Designated Router (ID) 2.2.2.2, Interface address 10.0.0.2**
 **Backup Designated router (ID) 1.1.1.1, Interface address 10.0.0.1**
 Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
 oob-resync timeout 40
 Hello due in 00:00:08
 Supports Link-local Signaling (LLS)
 Cisco NSF helper support enabled
 IETF NSF helper support enabled
 Index 1/1, flood queue length 0
 Next 0x0(0)/0x0(0)
 Last flood scan length is 0, maximum is 1
 Last flood scan time is 0 msec, maximum is 0 msec
 Neighbor Count is 2, Adjacent neighbor count is 2 
 **Adjacent with neighbor 2.2.2.2 (Designated Router)**
 **Adjacent with neighbor 3.3.3.3**
 Suppress hello for 0 neighbor(s)
```
Best practice enligt Cisco är att inte låta en router vara DR/BDR för mer än ett nät, detta är ju dock inte alltid möjligt att följa. För kom ihåg att detta gäller inte en hel area utan DR/BDR bestäms per "**shared network segment**". Se följande exempel (ritade/skrev lite på en screencap från Jeremys OSPF-nuggets): 
[![DR](/assets/images/2013/06/dr.jpg)](/assets/images/2013/06/dr.jpg) 

Observera att det ej väljs någon DR/BDR för Point-to-point nätet, routrarna kommer dock givetvis att bilda full adjacency som vanligt. Rent generellt är det inte speciellt viktigt vilken routern som blir DR/BDR, detta gäller dock inte för Frame-Relay, men mer om det senare!

Linkstate Advertisements
------------------------

Om något inträffar på nätverket som kräver en uppdatering kommer ansvarig router att skicka ett **LSU-paket (innehållandes ett LSA)** för den berörde routen över multicast. I en DR/BDR-miljö skickas det på **multicast**-adressen 224.0.0.6, som endast DR & BDR lyssnar på, för Point-to-Point skickas allt på 224.0.0.5. DR/BDR skickar i sin tur ut samma information till resterande medlemmar via multicast på adressen 224.0.0.5. Som exempel satte jag uppe följande topologi: 

[![ospf lsu update](/assets/images/2013/06/ospf-lsu-update.png)](/assets/images/2013/06/ospf-lsu-update.png) 

R1 är DR och R2 BDR i detta exempel, vad händer när vi lägger till ytterligare ett nät på R3 och annonserar ut detta? Wireshark visar följande: 

[![ospf lsu](/assets/images/2013/06/ospf-lsu.png)](/assets/images/2013/06/ospf-lsu.png) 

[![ospf lsu update debug](/assets/images/2013/06/ospf-lsu-update-debug.png)](/assets/images/2013/06/ospf-lsu-update-debug.png) 

En liten notis som jag inte sett nämnas i något av det material jag läst så skickar BDR LS Ack tillbaka på 224.0.0.5, medans DROTHER's skickar sina på 224.0.0.6, och DR verkar bevisligen inte skicka någon LS Ack tillbaka över huvudtaget. Mottagaren använder sedan följande flödesschema efter de mottagit en LSA: 
[![lsa flow](/assets/images/2013/06/lsaflow.png)](/assets/images/2013/06/lsaflow.png) 

Som synes kontrollerar mottagaren inte bara om route:n redan finns inlagd, utan även om dess sequence nummer är högre än vad den själv har. Detta skulle betyda att informationen är nyare och att routern behöver uppdatera sin egen databas. Skulle det istället vara så att mottagaren själv har ett högre sequence nummer för route:n kommer den istället skicka tillbaka en LSU med sin egen info för att hjälpa avsändaren få en korrekt bild av nätet. Varje genererat LSA-paket har en lifetime på 30 minuter, när detta gått ut så genererar varje router ett nytt paket (samt ökar sequence-number med 1) och skickar ut detta på nätverket. Vi kan kontrollera sequence-numret genom kommandot "sh ip ospf database",
```
R1#sh ip ospf database
OSPF Router with ID (1.1.1.1) (Process ID 1)
Router Link States (Area 0)
Link ID ADV Router Age Seq# Checksum Link count
1.1.1.1 1.1.1.1 1625 0x80000002 0x000DFD 1
2.2.2.2 2.2.2.2 1098 0x80000003 0x005A2C 2
3.3.3.3 3.3.3.3 875 0x80000003 0x002AA8 2
```
Detta skrapar dock endast lite på ytan av LSA och jag kommer återkomma med fler poster på samma ämne var så säker.. ;)
