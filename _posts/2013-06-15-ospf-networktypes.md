---
title: OSPF - Network Types
date: 2013-06-15 21:11
comments: true
categories: [OSPF]
---
Fortsättning på tidigare inlägg om [DR & BDR](http://Jonas Collén.wordpress.com/2013/06/12/ospf-the-basics-ii/). OSPF har ett flertal olika "**Network Types**" specificerade, och det är utefter detta som ospf-processen bestämmer om den behöver välja en **DR/BDR** under **2-way state** i skapandet av en **Neighbor Adjacency**. Dessa är:

*   Non-Broadcast (NMBA)
*   Point-To-Multipoint
*   Point-To-Multipoint Non-Broadcast
*   Broadcast
*   Point-To-Point

Kom ihåg att DR/BDR endast behövs för multiaccess-nät. Detta innebär att för **Point-to-Point** så **skippas** hela DR/BDR-processen över och **full** adjacency sätts upp. Skillnaden går att se genom "sh ip ospf neighbor" där vi har en Point-to-Point länk på Fa0/1 och ett vanligt multiaccess (ethernet) på fa0/0. Neighbor ID Pri State Dead Time Address Interface 2.2.2.2 1 FULL/DR 00:00:34 10.0.0.2 FastEthernet0/0 3.3.3.3 1 FULL/DROTHER 00:00:30 10.0.0.3 FastEthernet0/0 **192.168.0.2 0 FULL/ - 00:00:33 192.168.0.2 FastEthernet0/1*** _*Detta exempel är kanske lite förvirrande, per default är det alltid broadcast som network-type för Ethernetinterface, men för detta exempel ändrade jag detta genom "ip ospf network point-to-point"._ Samma sak gäller för **Point-to-Multipoint** & **PtM Non-Broadcast** räknas OSPF som flera inviduella **Point-to-Point** och väljer därför **EJ** någon DR/BDR. Så vi kan konstatera att **DR/BDR används endast för Broadcast & Non-Broadcast (_NMBA -_** _**Non-Broadcast MultiAccess**_**)**.  Skönt nog har Cisco gjort det enkelt för oss och dessa nätverkstyper konfigureras automatiskt åt oss. Vi kan kontrollera vilken network type ett interface har genom kommandot "**sh ip ospf interface**": **Ethernet:** 

FastEthernet0/0 is up, line protocol is up
 Internet Address 10.0.0.1/24, Area 0
 Process ID 1, Router ID 1.1.1.1, Network Type **BROADCAST**, Cost: 10

**Point-to-Point:**

Serial0/0 is up, line protocol is up 
 Internet Address 192.168.0.2/30, Area 1 
 Process ID 1, Router ID 192.168.0.2, Network Type **POINT_TO_POINT**, Cost: 10

**Point-to-MultiPoint**

Serial0/0.1 is up, line protocol is up 
 Internet Address 172.16.0.1/16, Area 1 
 Process ID 1, Router ID 192.168.0.2, Network Type **POINT_TO_MULTIPOINT**, Cost: 64

**Non-Broadcast:**

Serial0/0 is up, line protocol is up
Internet Address 172.16.0.1/16, Area 1 
 Process ID 1, Router ID 192.168.0.2, Network Type **NON_BROADCAST**, Cost: 64

Det finns som tidigare nämnt möjlighet att manuellt justera vilken network-type ett interface skall ha, detta kan vara användbart i framförallt labb-miljöer. Du ändrar detta via kommandot "**ip ospf network x**",

R4(config-if)#ip ospf network ?
 broadcast Specify OSPF broadcast multi-access network
 non-broadcast Specify OSPF NBMA network
 point-to-multipoint Specify OSPF point-to-multipoint network
 point-to-point Specify OSPF point-to-point network

DR/BDR i Frame-Relay
--------------------

Ett problem som uppstår i en Frame-Relay miljö som använder sig av Hub & Spoke är som bekant att all trafik mellan "spokes" måste gå via huvudroutern. Och som du förhoppningsvis inte sovit under dina CCNA-studier känner du redan till att routrar ej forwardar Broadcast - detta innefattar även Multicast. Om vi tar följande topologi för att utveckla: [![Framerelay DR](/assets/images/2013/06/framerelay-dr.png)](/assets/images/2013/06/framerelay-dr.png) Vad händer om Spoke 1 skulle bli DR eller BDR? Spoke2 kommer försöka sätta upp en full adjacency till Spoke1, vilket kommer misslyckas då all multicast fastnar i Hub och ej forwardas. Det är därför ett **KRAV** att Hub konfigureras till **DR** och att vi **EJ t****illåter varken Spoke1 eller Spoke2 att bli BDR, detta gäller för både Broadcast*- & NMBA i Hub&Spoke-nät**. Har du läst den tidigare posten ang. BD/BDR så har du förhoppningsvis redan koll på hur vi löser detta (hint: genom ip ospf priority x). *Kom ihåg att vi kan konfigurera Frame-Relay att tillåta en form av pseudo-broadcast genom antingen Inverse-ARP eller manuellt mappa dlci och ip-adress och sedan använda broadcast i statement, ex;

R1(config)#interface serial0 
R1(config-if)#no frame-relay inverse-arp 
R1(config-if)#frame map ip 200.1.1.2 122 broadcast 
R1(config-if)#frame map ip 200.1.1.3 123 broadcast

Neighbor adjacencys för Non-broadcast
-------------------------------------

Speciellt för Non-broadcast och Point-to-Multipoint Non-broadcast är ju som vi hör på namnet att de **EJ tillåter broadcast** (vilket även innebär att multicast stoppas). OSPF har dock möjlighet att helt använda sig av unicast istället, det enda vi behöver göra i detta fall är att manuellt ange våra neighbors. Väldigt enkelt!

R1(config)#router ospf 1
R1(config-router)#neighbor 10.0.0.2 
R1(config-router)#neighbor 10.0.0.3

Försöker vi dock göra detta på ett nätverk som tillåter broadcast får vi följande meddelande:

*Mar  1 01:43:08.991: %OSPF-4-CFG_NBR_INVAL_NET_TYPE: Can not use configured neighbor: neighbor command is allowed only on NBMA and point-to-multipoint networks

Här är förövrigt en bra "fusklapp" att använda. 
[![networktypes](/assets/images/2013/06/networktypes.png)](/assets/images/2013/06/networktypes.png)
