---
title: EIGRP - Frame Relay
date: 2013-06-10 15:59
comments: true
categories: [EIGRP]
tags: [frame-relay]
---
EIGRP över Frame-Relay är lite speciellt, något jag redan nämnt i tidigare poster är att det ej är möjligt att skicka broadcast. Detta ställer ju till stora problem för just EIGRP med även andra routingprotokoll som OSPF & RIPv2 då de använder sig av multicast. Det finns dock två olika lösningar för detta:

Pseudobroadcast
---------------

Frame-Relay har en inbyggd funktion där den emulerar broadcast (dessa skickas istället som unicast) över en länk. Detta konfigureras automatiskt om vi använder oss av "_Inverse ARP_" för att sätta upp mappningen med DLCI-nummer mellan två siter. Det går dock även att göra detta manuellt, först stänger vi av Inverse Arp genom "_no frame-relay inverse-arp_", och skapar sedan mappningen manuellt via "_frame-relay map ip x.x.x.x  (destinations-adress) x (lokal dlci) **broadcast**_".

Statisk Neighbor-mappning
-------------------------

Det finns även möjligheten att statiskt ange varje neighbor, eigrp skickar då istället unicast endast. Detta gör dock att automatisk neighbor adjacency inte längre fungerar utan man är tvungen att manuellt skriva in varje neighbor vart eftersom de läggs till i nätet. Detta konfigureras under "_router eigrp x"_, ex: "_neighbor x.x.x.x serial0/0.1_".

Split-Horizon över multipoint
-----------------------------

Split-Horizon är per default inaktiverat på fysiska interface, men aktiverat på subinterface. Detta leder till stora problem med routing-uppdateringar om vi kör en Multipoint-access över ett subinterface.. I "Hub-n-spoke"-exemplet nedan så får R3 aldrig reda på näten som R2 har och vice-versa på grund av just Split-Horizon. 
[![hubnspoke](/assets/images/2013/06/hubnspoke.jpg)](/assets/images/2013/06/hubnspoke.jpg) 

Vi måste därför komma ihåg att ALLTID stänga av split-horizon i dessa fall. Detta görs på subinterfacet med kommandot _"no ip split-horizon eigrp x_" där x är AS-numret. Den enda fördelen med att använda sig av multipoint istället för point-to-point är egentligen att man sparar in på antal ip-adresser som behövs. Det kan även vara användbart vid riktigt stora nät där det skulle bli alldeles för stort administrativt arbete att skapa point-to-point-länkar för varje site.

Justera EIGRPs bandbreddsutnyttjande
------------------------------------

Som nämndes i gårdagens post finns det möjlighet att styra hur mycket bandbredd EIGRP får använda sig av på ett interface. Detta är default 50% av den konfigurerade bandwith:en, och om det är ett multipoint-förhållande så delar den värdet med antal neighbors. Om vi tar nedanstående topologi ser vi att det kan leda till problem om vi inte i efterhand justerar dessa värden. 
[![frame-relay hubnspoke](/assets/images/2013/06/framerelay5.jpg)](/assets/images/2013/06/framerelay5.jpg) 

Om vi ovanstående exempel säger följande:

*   PVC1 mellan BB & R4 har en CIR på 512k
*   PVC2 mellan BB & R2 har en CIR på 128k
*   PVC3 mellan BB & R3 har en CIR på 128k
*   PVC4 mellan BB & R5 har en CIR på 64k

Vad händer då om R4 skickar en update till R5? 50% av 512k är 256k (vi delar inte per antal neighbors då det endast är Backbone som använder sig av multipoint-interface), länken för R5 kommer med andra ord att bli överlastad med endast EIGRP-trafik. Enklaste lösningen är egentligen att migrera nätet från multipoint och istället skapa point-to-point för varje site. Men det går som sagt även att justera bandbreddsutnyttjandet enligt följande: Ta PVC med lägst bandbredd, i detta fall PVC4 på 64k och multiplicera med antal pvc's får vi = 256k, detta sätter vi som bandwith på Backbone-routern. Backbone-routern kommer då dela 256k / 4 (antal neighbors) och limitera EIGRP-trafiken till 64k per PVC - Problemet löst! Detta ska väl täcka Frame-Relay för EIGRP och jag börjar känna mig klar med detta protokoll! Men innan jag hoppar vidare till OSPF tänkte jag sätta mig och göra följande labbar från gns3vault:

*   Frame-Relay Basucs
*   EIGRP over Frame-Relay with Subinterfaces
*   EIGRP over Frame-Relay with Multipoint-interface
*   EIGRP Multipoint bandwith Pacing
*   EIGRP Point-to-point bandwith Pacing
*   EIGRP Hybrid Bandwith Pacing

Ska dock först skriva en mer detaljerad post om EIGRP's Query-paket som kan ställa till den hel del problem och hur man kommer runt detta.