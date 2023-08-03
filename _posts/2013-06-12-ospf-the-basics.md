---
title: OSPF - The Basics
date: 2013-06-12 16:12
comments: true
categories: [OSPF]
---
OSPF är ett Link-state Routing Protocol, som till skillnad från Distance Vector (EIGRP/RIP) har en karta över hela nätverket (åtminstone sin area, men mer om det senare) i sin topology-table, och varje router räknar själv ut den bästa vägen för att nå sin destination genom Dijkstra's Shortest Path First (**SPF**)-Algoritm. Förhoppningsvis har vi inte redan glömt att nackdelen med just Distance Vector är att de endast vet vad dess grannar berättat. En annan stor fördel gentemot  EIGRP är att OSPF är OpenSource och ej låst till Cisco, det  finns med andra ord möjligheter att sätta upp ett "mixed vendor"-nätverk och få dessa att prata med varandra genom OSPF! OSPF använder sig av triggade updates, och skickar endast ut information om någon förändring inträffar på nätverket, som ex att en länk går ner/ett nät läggs till. Har det dock inte hänt något på 30 minuter skickas en "ls refresh" från varje router med dess routing-information för att vara säker på att alla har samma bild av nätverket och inget har missats. OSPF använder precis som EIGRP tre "tables":

### Neighbor Table

Innehåller information om grannar som skickat Hello-paket till routern (behöver nödvändigtvis ej ha full adjacency mellan varandra). Information som ex. Router-ID, IP-adress och interface sparas här.

```
R1#sh ip ospf neighbor
Neighbor ID Pri State Dead Time Address Interface
3.3.3.3 1 FULL/DR 00:00:30 12.0.0.2 FastEthernet0/1
2.2.2.2 1 FULL/DR 00:00:38 10.0.0.2 FastEthernet0/0
```
### Topology Table

Innehåller samtliga routes inom en area, tillskillnad från ex. EIGRP som endast sparar successor/f.successor-routes.
```
R1#show ip ospf route
OSPF Router with ID (1.1.1.1) (Process ID 1)
Area BACKBONE(0)
Intra-area Route List
* 12.0.0.0/24, Intra, cost 10, area 0, Connected
 via 12.0.0.1, FastEthernet0/1
* 10.0.0.0/24, Intra, cost 10, area 0, Connected
 via 10.0.0.1, FastEthernet0/0
*> 11.0.0.0/24, Intra, cost 20, area 0
 via 12.0.0.2, FastEthernet0/1
 via 10.0.0.2, FastEthernet0/0
*> 192.168.0.0/24, Intra, cost 11, area 0
 via 12.0.0.2, FastEthernet0/1
* 1.1.1.1/32, Intra, cost 1, area 0, Connected
 via 1.1.1.1, Loopback0
*> 2.2.2.2/32, Intra, cost 11, area 0
 via 10.0.0.2, FastEthernet0/0
*> 3.3.3.3/32, Intra, cost 11, area 0
 via 12.0.0.2, FastEthernet0/1
```
### Routing Table

Fungerar precis som vanligt, de routes med lägst metric till en destination sparas här. Observera att även OSPF har lastbalansering då det installerats två vägar till nätet 11.0.0.0/24 när metricen är densamma.
```
1.0.0.0/32 is subnetted, 1 subnets
C 1.1.1.1 is directly connected, Loopback0
 2.0.0.0/32 is subnetted, 1 subnets
O 2.2.2.2 [110/11] via 10.0.0.2, 00:24:35, FastEthernet0/0
 3.0.0.0/32 is subnetted, 1 subnets
O 3.3.3.3 [110/11] via 12.0.0.2, 00:22:30, FastEthernet0/1
 10.0.0.0/24 is subnetted, 1 subnets
C 10.0.0.0 is directly connected, FastEthernet0/0
 11.0.0.0/24 is subnetted, 1 subnets
O 11.0.0.0 [110/20] via 12.0.0.2, 00:22:30, FastEthernet0/1
           [110/20] via 10.0.0.2, 00:24:36, FastEthernet0/0
O 192.168.0.0/24 [110/11] via 12.0.0.2, 00:14:28, FastEthernet0/1
 12.0.0.0/24 is subnetted, 1 subnets
C 12.0.0.0 is directly connected, FastEthernet0/1
```
Metric
------

OSPF använder sig av en metric som kallas "cost". Cost adderas för varje utgående interface och får dess värde från =  **10^8 / Bandwith**, men vi kan förenkla detta genom att istället skriva 100 / Bandwith (i Mbps). Till skillnad från EIGRP så sparar OSPF endast den bästa routen som räknats ut genom SPF, om en förändring sker måste routern därför gå tillbaka och räkna ut en ny väg (om möjligt). Här är en tabell med de vanligaste hastigheterna och dess cost:

*   56k - 1785
*   64k - 1562
*   T1 (1,544) - 65
*   E1 (2,048) - 48
*   Ethernet - 10
*   Fast Ethernet - 1

Som du kanske ser så har OSPF per default ingen möjlighet att se skillnad på länkar över 100Mbit, främst pga cost-utträkningen togs fram under 80-talet.. :) Vi kan dock ändra detta genom kommandot:
```
router ospf x
auto-cost reference-bandwith x (anges i Mbps, 100 är default)
```
Om vi har länkar upp till 10Gb i vårat nätverk skulle vi då sätta reference-bandwith till 10000, detta måste ändras på samtliga routrar!

Areas
-----

OSPF använder sig av ett koncept som kallas Area's, per default kommer du alltid ha en standard-area, "0", detta är även vad vi kallar "**Backbone area**". Genom att använda area's får vi en möjlighet att segmentera nätverket vilket har flera fördelar, exempelvis behöver OSPF endast skicka uppdateringar och använda SPF-algoritmen för att räkna ut den bästa vägen inom sin egen area. Detta minskar belastningen på CPU:er och ger snabbare konvergenstider. Om vi använder oss av en liknelse med GPS, så går det ju betydligt snabbare att räkna ut kortaste vägen mellan två gator inom samma stad än ex. hela sträckan mellan Västerås -  Malmö (Västerås hade varit en egen ospf-area). 

[![OSPF areas](/assets/images/2013/06/ospf-areas.png)](/assets/images/2013/06/ospf-areas.png) 

Det finns vissa regler som gäller för just areas:

*   Varje area måste ha en anslutning till area 0 / Backbone (kan dock vara virtuell, mer om detta senare)
*   Alla routrar inom en area måste ha samma topology-table/samma uppfattning om hur nätet ser ut
*   Uppdateringar skickas endast inom den egna arean
*   Routrar mellan två area's kallas "Area Border Router" (**ABR**)
*   Routrar som är anslutet till ett annat nätverk som inte använder OSPF (ex BGP/RIP/EIGRP) kallas "Autonomous Border Router" (**ASBR**)
*   Nätsummeringar kan endast göras i **ABR** & **ASBR**
*   Neighbor-adjacencys skapas endast inom samma area

Cisco rekommenderar förövrigt att vi har max 50st routrar per area, och varje router bör endast vara neightbor med max. 16st andra. Men efter att ha lyssnat på OSPF - Design från PacketPushers  som jag länkade i ett tidigare inlägg så kan vi konstatera att detta är grovt överdrivet, och det är egentligen inga som helst problem att ha hundratals routrar i en och samma area. KSS - Keep it Simple Stupid! :)

Packet Types
------------

OSPF använder sig ej av TCP för sin kommunikation utan har skapat sitt eget Lager 4-protokoll för att verifiera att allt sänds OK. De paket-typer som finns i OSPF är följande:

*   **Hello-packet**: Används för att sätta upp Neighbor-adjacencys
*   **Database Description-packet (DBD)**: En "lightversion" av routerns linkstate database
*   **Linkstate Request (LSR*)**: Används för att be om mer detaljerad information från en router
*   **Linkstate Update(LSU*)**: Används för att skicka mer detaljerad routing-information
*   **Linkstate Advertisement(LSA)**: En LSU innehåller en/flera LSA's för att minska det antal paket som behöver skickas, varje enskild LSA innehåller information om en specifik route
*   **Linkstack Acknowledgement(LSAck)**: För att bekräfta att DBD/LSR/LSU-paket kommit fram skickas en LSAck tillbaka

_*Vid debugging visas LSR som LS REQ och LSU som LS UPD._

Hello-Packets
-------------

Efter att vi startat OSPF-processen för ett interface kommer routern börja skicka ut Hello-paket. Till skillnad från EIGRP så är det ganska mycket information som dessa innehåller och ett flertal fält måste vara identiska för att en adjacency skall kunna skapas. Hello-paketet består för följande information:

*   **Router-ID**: Varje OSPF-Router behöver ett unikt router-id, mer om detta senare)
*   **Hello / Dead interval***: Här bestäms intervallet som Hello-paket kommer att skickas samt hur länge vi väntar innan en neighbor anses som offline (samma som EIGRP's Holdtimer), dead interval är per default "Hellotimer x 4"
*   **Neighbors**: Innehåller information om routerns övriga neighbors
*   **Area ID***: Vilket area routern befinner sig i
*   **Router Priority**: Används för att bestämma vem som blir Designated Router (**DR**) / Backup Designated Router (**BDR**), mer om detta senare
*   **DR & BDR IP-adress**: IP-adressen till routerns DR / BDR
*   **Authentication Password***: Används för lösenordsskyddade routinguppdateringar, antingen i cleartext eller md5
*   **Stub area flag*:**  Förutom olika area-nummer har OSPF även olika typer av areas, mer om detta senare

Punkterna markerade med "*" kräver att båda routrarna har identiskt information för respektive fält. Hello-paket skickas per default ut var 10:e sekund på Broadcast/P2P-länkar, och var 30:e sekund på NMBA. Då dead interval var "4 x hello-timer" innebär detta att det sedan kommer ta 40 sekunder innan en router anser att sin granne är död på ex. ett ethernet-nätverk, och hela **180 sekunder** på ett frame-relay nät! Observera att AS-numret aldrig skickas med, till skillnad från EIGRP är OSPF's AS endast lokalt, du kan därför blanda AS-nummer mellan routrarna och de kommer ändå få adjacency om resterande fält är ok. Varför man nu skulle vilja göra detta??.. :) En debug så ser vi hur paketen ser ut i CLI:
```
R1#debug ip ospf packet
OSPF packet debugging is on
R1#
*Mar 1 04:00:27.539: OSPF: rcv. v:2 t:1 l:48 rid:3.3.3.3
 aid:0.0.0.0 chk:CF93 aut:0 auk: from FastEthernet0/1
```
*   **V:2** - OSPF-version (IPv6 använder OSPFv3)
*   **T:1** - Typ av OSPF-paket, 1 = Hello-paket
*   **L:48** - Paketlängd i bytes
*   **RID** - Router-ID, i detta fall 3.3.3.3
*   **AID** - Area-ID, i detta fall 0
*   **CHK:CF93** - Checksum för felhantering
*   **AUT:0** - Autentisering, 0 = ingen (1 är cleartext, 2 md5)
*   **AUK:** - Används endast om autentisering är aktiverat

Neighbor Adjacency
------------------

Processen att sätta upp en Neighbor Adjacency i OSPF är tyvärr inte på långa vägar lika enkelt som EIGRP där vi endast behöver utbyta varsitt hello-paket, utan här är det betydligt mer omständigt. Efter att du startat OSPF-processen på R1 sker följande (vi förutsätter att det redan är aktiverat på R3): 
[![ospf neighbors](/assets/images/2013/06/ospf-neighbors.png)](/assets/images/2013/06/ospf-neighbors.png) 

R1 IP: 10.0.0.1 
R2 IP: 10.0.0.2

### Steg 1

R1 kommer först att välja ett Router-ID (ingår i Hello-paketet), och detta kan göras på tre olika sätt:

*   Routern väljer den högsta IP-adressen från något av sina aktiva interface (behöver ej köra OSPF)
*   Om ett loopback-interface finns konfigurerat har detta högre prioritet än andra interface oavsett IP-adress
*   Om du manuellt skriver in ett Router-ID så har detta prioritet över både Loopback och andra interface, oavsett adress

Det går sedan endast att byta Router-ID vid **omstart** av OSPF-processen alternativt hela routern. Kommer återkomma till detta när vi börjar prata om DR / BDR senare. I detta fall kommer Router-ID att bli **1.1.1.1**.

### Steg 2

R1 lägger till det interface du specificerat under "network x.x.x.x x.x.x.x area x" i "Link-state Database" (**LSDB**). Ex:
```
router ospf 1
network 10.0.0.0 0.0.0.3 area 0
```
### Steg 3 - "Down State"

R1 börjar skicka ut Hello-paket på Fa0/1-interfacet innehållande den information vi skrev om under "Hello-packets" via **multicast** på adressen **224.0.0.5**. Länken betraktas nu vara i ett "**Down State**" tills den själv mottar ett Hello-paket.

### Steg 4 - "Init State"

R1 har nu förutom att börja skicka ut sina egna Hello-paket även mottagit ett från sin neighbor R3, detta skickades som ett **unicast** direkt till R1 på 10.0.0.1. Länken ändras från att vara i ett "**Down State**" till "**Init State**", R1 & R3 jämför nu så att alla fält markerade med * under Hello-packet stämmer överens med den information de själv besitter. Är allt ok går de vidare till Steg 5, om inte så backar de länken till "**Down State**"/steg 3 igen.

### Steg 5 - "2-way State"

R1 & R3 kontrollerar nu om den redan finns angiven under "Neighbors"-fältet i det mottagna Hello-paketet, om så är fallet bortser den från Hello-paketet och nollställer istället **Dead interval**. Finns de inte med betraktas neighborn som ny och läggs istället till i "**Neighbor Table**", de går sedan vidare till Steg 6.

### Steg 6 - "Exstart State"

Länken ändras från ett "**2-way State**" till ett "**Exstart State**". De två routrarna kommer nu överens om vem som skall vara Master & Slave, detta bestäms efter:

*   Priority (default "1")
*   Om Priority är samma används istället Router-ID, där högst router-id = Master

Master/Slave-förhållandet bestämmer endast vilken av routrarna som skall börja skicka sin routing-information till den andra. I detta exempel hade R3 blivit Master, och R1 Slave. Efter de kommit överens sker följande:

*   Master skickar en "_lightversion**"**_ av sin routing-databas innehållande vilka routes den känner till, detta kallas "Database Description-Packet" (**DBD**), Jeremy från CBT kallade även detta "Cliffnotes of it's linkstate database"
*   Slave skickar en Ack på Mastern's DBD-paket och sänder sedan sitt egna DBD-paket
*   Master skickar en Ack på Slave's DBD-paket

### **Steg 7 - "Loading State"**

Både Master & Slave går nu över den information de fått i DBD-paketet och jämför med sin egen **linkstate database**. För de nät de inte redan känner till sker följande:

*   Slave skickar ett Linkstate Request (**LSR**) och ber om mer information för dessa nät
*   Master skickar en LSAck och svarar med en Linkstate Update (**LSU**), som i sin tur innehåller en Linkstate Advertisement (**LSA)** för varje route Slave behövde information om
*   Slave svarar med en LSAck
*   Master skickar ett Linkstate Request (**LSR**) och ber om mer information
*   Slave skickar en LSAck och svarar med en  Linkstate Update (**LSU**), som i sin tur innehåller en Linkstate Advertisement (**LSA)** för varje route Master behövde information om
*   Master svarar med en LSAck

### Steg 8 - "Full State"

R1 & R3 är nu synkade med varandra och länken ändras från "**Loading State**" till "**Full State**". Det är nu dags för SPF att kavla upp ärmarna och räkna ut vilka routes som skall läggas till i **Routing Table**.

### Debug

Det är ju alltid bättre att se hur det faktiskt fungerar i verkligen än att jag sitter och skriver såhär. Här följer en debug från R1 som visar hela händelseförloppet från Steg 1 till Steg 8 (har dock rensat bort info om BD/BDR).
```
*Mar 1 04:17:22.914: OSPF: Interface FastEthernet0/1 going Up
*Mar 1 04:17:22.954: OSPF: 2 Way Communication to 3.3.3.3 on FastEthernet0/1, state 2WAY **<- Steg 5**
*Mar 1 04:17:22.958: OSPF: Backup seen Event before WAIT timer on FastEthernet0/1
*Mar 1 04:17:22.962: OSPF: Send DBD to 3.3.3.3 on FastEthernet0/1 seq 0x1DEF opt 0x52 flag 0x7 len 32
R1(config-router)#
*Mar 1 04:17:22.998: OSPF: Rcv DBD from 3.3.3.3 on FastEthernet0/1 seq 0xBD2 opt 0x52 flag 0x7 len 32 mtu 1500 state EXSTART **<- Steg 6**
*Mar 1 04:17:22.998: OSPF: NBR Negotiation Done. We are the **SLAVE** 
*Mar 1 04:17:23.002: OSPF: Send DBD to 3.3.3.3 on FastEthernet0/1 seq 0xBD2 opt 0x52 flag 0x0 len 32
*Mar 1 04:17:23.038: OSPF: Rcv DBD from 3.3.3.3 on FastEthernet0/1 seq 0xBD3 opt 0x52 flag 0x3 len 92 mtu 1500 state EXCHANGE
*Mar 1 04:17:23.038: OSPF: Send DBD to 3.3.3.3 on FastEthernet0/1 seq 0xBD3 opt 0x52 flag 0x0 len 32
*Mar 1 04:17:23.070: OSPF: Rcv DBD from 3.3.3.3 on FastEthernet0/1 seq 0xBD4 opt 0x52 flag 0x1 len 32 mtu 1500 state EXCHANGE
*Mar 1 04:17:23.070: OSPF: Exchange Done with 3.3.3.3 on FastEthernet0/1
*Mar 1 04:17:23.070: OSPF: Send LS REQ to 3.3.3.3 length 36 LSA count 3 **<- Steg 7**
*Mar 1 04:17:23.074: OSPF: Send DBD to 3.3.3.3 on FastEthernet0/1 seq 0xBD4 opt 0x52 flag 0x0 len 32
*Mar 1 04:17:23.106: OSPF: Rcv LS UPD from 3.3.3.3 on FastEthernet0/1 length 132 LSA count 3
*Mar 1 04:17:23.106: OSPF: Synchronized with 3.3.3.3 on FastEthernet0/1, state FULL
*Mar 1 04:17:23.106: %OSPF-5-ADJCHG: Process 1, Nbr 3.3.3.3 on FastEthernet0/1 from LOADING to FULL, Loading Done **<- Steg 8**
```
Och nu är det äntligen klart! Inte riktigt lika enkelt som EIGRP kanske.. ;) Det finns dock undantag till ovanstående, men det återkommer jag till i nästa post om BD/BDR.
