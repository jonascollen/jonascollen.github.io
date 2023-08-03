---
title: OSPF - LSA Types
date: 2013-06-16 19:25
comments: true
categories: [OSPF]
tags: [lsa]
---
OSPF fyller som bekant sin Linkstate Database (LSDB) via LSA-paket den får från andra routrar. Något vi ej nämnt tidigare är dock att det finns ett flertal olika typer av LSA-paket som alla har olika användningsområden och det är dessa vi kommer gå igenom nu. LSA Type 6 & 8 täcks dock inte av CCNP-materialet och kommer därför ej att förklaras mer ingående.

*   LSA Type 1 - Router LSA
*   LSA Type 2 - Network LSA
*   LSA Type 3 - Summary LSA
*   LSA Type 4 - Summary ASBR LSA
*   LSA Type 5 - Autonomous system external LSA
*   LSA Type 6 - Multicast OSPF LSA
*   LSA Type 7 - External LSA
*   LSA Type 8 - External attribute LSA for BGP

LSA Type 1 - Router LSA
-----------------------

Skickas av samtliga routrar och innehåller information om routerns alla **directly connected**\-nät (som vi lagt till i ospf-processen). Denna LSA-typ skickas dock **endast inom den egna arean.**

*   IP-Prefix inkl subnätmask
*   Link type

För att göra det hela lite krångligare så finns det fyra olika varianter på "**Link type**", ej att blanda ihop med [Network Types](http://www.jonascollen.se/posts/ospf-networktypes/) som vi gick igenom i gårdagens post, de flesta är dock som tur är rätt självklara.. **Link type 1** - **Point-to-Point** används för just Point-to-Point interface och innehåller Router-ID till neighborn. **Link type 2 - Link to transit network** används för **multiaccess** nätverk som Ethernet när routern har lyckats skapa **adjacency** mot åtminstone en annan router (DR/BDR), alternativt att denna router är DR och bildat adjacency mot en DROTHER. Link ID innehåller här Router-ID:t för DR-routern. Om ingen neighbor adjacency har lyckats skapats för ett multiaccess-nätverk annonseras det istället som ett **Link type 3 - Link to stub network**. Detta används även om vi annonserar nät som ligger på ett loopback-interface. **Link type 4 - Virtual Link** används när vi skapat en virtuell länk mellan areas, mer om detta senare! 
[![OSPF LSA1 linktypes](/assets/images/2013/06/ospf-lsa1-linktypes.png)](/assets/images/2013/06/ospf-lsa1-linktypes.png) 

Låt oss använda följande topologi för att kolla hur dessa paket används och ser ut i praktiken. 
[![ospf LSA topology](/assets/images/2013/06/ospf-lsa-topology.png)](/assets/images/2013/06/ospf-lsa-topology.png)

```
R2#sh ip ospf neighbor
Neighbor ID Pri State Dead Time Address Interface
2.2.2.2 1 FULL/DR 00:00:38 10.0.0.2 FastEthernet0/0
3.3.3.3 1 FULL/BDR 00:00:32 10.0.0.3 FastEthernet0/0
```

Här är ett exempel på ett LSU-paket från R2 som annonserar sitt Loopback-nät på 2.2.2.2/32 samt Broadcast-nät på 10.0.0.0/24 till DR & BDR. 
[![ospf lsa type 1 router lsa](/assets/images/2013/06/ospf-lsa-type-1-router-lsa1.png)](/assets/images/2013/06/ospf-lsa-type-1-router-lsa1.png) 

Observera att för Link Type 2 så är det ip-adressen för DR som sätts under "Transit ID".

LSA Type 2 - Network LSA
------------------------

Används endast i multiaccess-nätverk där vi använder oss av DR & BDR. LSA Type 2 skickas av **DR-routern** och annonserar att det är just den routern som är DR. Det innehåller även en lista över samtliga routrar som är anslutna till segmentet inklusive IP-prefix och subnätmask. **Detta skickas precis som LSA Type 1 endast inom den egna arean**. Här har vi ett LSA-paket från R2 (DR) som skickas till R1 efter att de skapat en adjacency. 
[![LSA type 2](/assets/images/2013/06/lsa-type-2.png)](/assets/images/2013/06/lsa-type-2.png) 

Notera att under "Attached Router" så är det inte ip-adressen till routrarna som specificeras utan dess **Router-ID**.

LSA Type 3 - Summary LSA
------------------------

Då LSA Type 1 endast annonseras inom den egna arean behöver vi ett sätt att sprida routing-information när vi använder oss av mer än bara area 0/backbone. Det är då ABR (Area Border Routers) kommer in i bilden, de tar Type 1 LSA-paketen och gör om dessa till ett LSA Type 3 vilket den sedan skickar ut till area 0 för vidare spridning. Om vi tar bygger ut vår tidigare topologi till följande så kan vi se hur detta går steg för steg. 

[![Multiarea LSA](/assets/images/2013/06/multiarea-lsa.png)](/assets/images/2013/06/multiarea-lsa.png) 

I ovanstående exempel kommer R1 att bli ABR mellan area 10 och area 0. R4 kommer skicka LSA Type 1 och informera om alla "directly connected"-nät (Lo0 44.44.44.44/32 & 192.168.0.0/24).  Då det är ett Point-to-Point nät så väljs som bekant ingen DR/BDR  och det skickas därför ej några LSA Type 2. Kom ihåg att LSA Type 1 endast skickas inom arean (10), för att R2 & R3 ska kunna lära sig dessa behöver R1 som i detta fall blir **ABR** annonsera detta. R1 åstadkommer detta genom att göra om LSA Type 1-paketet från R4 till ett LSA Type 3 och annonserar sedan ut detta i area 0 till R2 & R3. Likaså kommer den ta LSA Type 1-paketen som skickas inom area 0 och skicka ut dessa som LSA Type 3 ut på area 10. Type 3 LSA-routes visas som OSPF InterArea-route (O IA) i routing-tabellen för R3:

```
1.0.0.0/32 is subnetted, 1 subnets
O 1.1.1.1 \[110/11\] via 10.0.0.1, 01:26:22, FastEthernet0/0
 2.0.0.0/32 is subnetted, 1 subnets
O 2.2.2.2 \[110/11\] via 10.0.0.2, 01:27:41, FastEthernet0/0
 3.0.0.0/32 is subnetted, 1 subnets
C 3.3.3.3 is directly connected, Loopback0
 10.0.0.0/24 is subnetted, 1 subnets
C 10.0.0.0 is directly connected, FastEthernet0/0
**O IA 192.168.0.0/24 \[110/74\] via 10.0.0.2, 00:07:31, FastEthernet0/0**
 **44.0.0.0/32 is subnetted, 1 subnets**
**O IA 44.44.44.44 \[110/75\] via 10.0.0.2, 00:05:30, FastEthernet0/0**
```

Kollar vi i Wireshark ser LSA-paketet ut enligt följande (skickat från ABR R1 till area 10 som multicast på 224.0.0.5): 
[![LSA Type 3](/assets/images/2013/06/lsa-type-3.png)](/assets/images/2013/06/lsa-type-3.png) 

En show ip ospf database på R3 visar följande:
```
R3#sh ip ospf database
OSPF Router with ID (3.3.3.3) (Process ID 1)
Router Link States (Area 0) **<- LSA Type 1**
Link ID ADV Router Age Seq# Checksum Link count
1.1.1.1 1.1.1.1 1745 0x80000005 0x00F001 2
2.2.2.2 2.2.2.2 866 0x80000006 0x0027BB 2
3.3.3.3 3.3.3.3 1634 0x80000006 0x0017BF 2
Net Link States (Area 0) **<- LSA Type 2**
Link ID ADV Router Age Seq# Checksum
10.0.0.2 2.2.2.2 1778 0x80000004 0x000CFA
Summary Net Link States (Area 0) **<- LSA Type 3**
Link ID ADV Router Age Seq# Checksum
44.44.44.44 2.2.2.2 739 0x80000001 0x00E959
192.168.0.0 2.2.2.2 862 0x80000001 0x001E6D
```

Namnet Summary LSA är lite vilseledande då route's **inte** summeras automatiskt.

LSA Type 4 - Summary ASBR LSA
-----------------------------

När vi redistributar routing-information in till OSPF, exempelvis från RIP eller statiska routes, så kommer den ansvariga routern att ändra en bit i sin LSA Type 1 och identifiera sig som en **ASBR** (Autonomous System Border Router). Detta innebär dock samma problem igen, Type 1 skickas endast inom den egna arean och vi behöver ju sprida ut denna informationen till övriga areas. När en ABR upptäcker denna ändring i LSA Type 1-paketet kommer den skapa ett **LSA Type 4** för att skicka vidare ut till övriga areas. I detta paket pekas ASBR-routen ut så att övriga areas ska veta vem det är de ska prata med, advertising router kommer ju dock bli ABR, men den kommer ange Router-ID för ASBR under "**Link State ID**". Vi testar detta genom att skapa en statisk route på R4 för **172.16.0.0/16** och redistributar denna information till area 10. R1 ser att **R4** nu utger sig för att vara en **ASBR** och skapar därför ett LSA Type 4-paket och skickar ut på area 0 via multicast. 

[![LSA type 4](/assets/images/2013/06/lsa-type-4.png)](/assets/images/2013/06/lsa-type-4.png) 

Observera att Advertising router blir som sagt R1, men Link Sate ID pekar till R4 som destination. Paketet innehåller heller **ingen** information om nätet 172.16.0.0/16!

LSA Type 5 - Autonomous system external LSA
-------------------------------------------

LSA Type 4 används bevisligen endast för att peka ut vilken/vlka router(s) som är ASBR. Vad händer då med nätet som nu redistributas in till OSPF av R4? Kollar vi i wireshark kan vi se följande LSA-paket skickas ut från ASBR/R4: 

[![LSA type 5](/assets/images/2013/06/lsa-type-5.png)](/assets/images/2013/06/lsa-type-5.png) 

Dessa route's skickas **EJ** ut som vanliga LSA Type 1 då det inte är directly connected (**även** fast vi i detta fall gjort en statisk route som pekar till null0), istället märks dessa routes som "**LSA Type 5 - External**". Dessa paket har ej samma begränsningar som Type 1 & 2 utan skickas automatiskt vidare av ABR's till andra areas. Vi kan bekräfta detta genom att kolla vad R1 annonserar ut till area 0: 

[![LSA type 5 area0](/assets/images/2013/06/lsa-type-5-area0.png)](/assets/images/2013/06/lsa-type-5-area0.png)

Kollar vi i R3 routing table kan vi se att dessa route's markeras som O E1 eller O E2 (mer om detta senare) - Ospf External Type 1 / Type 2.
```
1.0.0.0/32 is subnetted, 1 subnets
O 1.1.1.1 \[110/11\] via 10.0.0.1, 02:08:01, FastEthernet0/0
 2.0.0.0/32 is subnetted, 1 subnets
O 2.2.2.2 \[110/11\] via 10.0.0.2, 02:09:20, FastEthernet0/0
 3.0.0.0/32 is subnetted, 1 subnets
C 3.3.3.3 is directly connected, Loopback0
**O E2 172.16.0.0/16 \[110/20\] via 10.0.0.2, 00:16:54, FastEthernet0/0**
 10.0.0.0/24 is subnetted, 1 subnets
C 10.0.0.0 is directly connected, FastEthernet0/0
O IA 192.168.0.0/24 \[110/74\] via 10.0.0.2, 00:49:13, FastEthernet0/0
 44.0.0.0/32 is subnetted, 1 subnets
O IA 44.44.44.44 \[110/75\] via 10.0.0.2, 00:47:10, FastEthernet0/0
```
Och kontrollerar vi databasen igen ser vi att den byggts på lite sen sist:
```
OSPF Router with ID (3.3.3.3) (Process ID 1)
Router Link States (Area 0) **<-** **Type 1**
Link ID ADV Router Age Seq# Checksum Link count
1.1.1.1 1.1.1.1 1870 0x80000006 0x00EE02 2
2.2.2.2 2.2.2.2 1195 0x80000007 0x0025BC 2
3.3.3.3 3.3.3.3 1801 0x80000007 0x0015C0 2
Net Link States (Area 0) **<- Type 2**
Link ID ADV Router Age Seq# Checksum
10.0.0.2 2.2.2.2 1974 0x80000005 0x000AFB
Summary Net Link States (Area 0) **<- Type 3**
Link ID ADV Router Age Seq# Checksum
44.44.44.44 2.2.2.2 948 0x80000002 0x00E75A
192.168.0.0 2.2.2.2 1195 0x80000002 0x001C6E
Summary ASB Link States (Area 0) **<- Type 4**
Link ID ADV Router Age Seq# Checksum
44.44.44.44 2.2.2.2 1097 0x80000001 0x00D171
Type-5 AS External Link States **<- Type 5**
Link ID ADV Router Age Seq# Checksum Tag
172.16.0.0 44.44.44.44 1105 0x80000001 0x0035FD 0
```
LSA Type 7 - External LSA
-------------------------

Denna LSA är lite speciell, för att förstå behovet av denna behöver vi först lära oss de olika Area Types som finns inom OSPF, det får bli nästa post!
