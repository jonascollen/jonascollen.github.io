---
title: Switching - Spanning-Tree
date: 2013-09-08 12:46
comments: true
categories: [Switch]
tags: [stp,bpdu]
---
Då vi saknar exempelvis ett TTL-fält i våra Lager 2-frames behövdes något för att förhindra att frames kan loopa i all oändlighet (broadcast stormar etc) och i slutändan sänka vårat nätverk när vi har redundanta länkar. Spanning-tree löser detta problem genom att helt enkelt stänga av redundanta länkar och på så vis förhindra att en loop kan bildas. 

![spanning-tree](/assets/images/2013/09/spanning-tree.png)
Switcharna i vår topologi kommer först att välja en Root-Bridge oavsett vilken typ av Spanning-tree vi använder oss, vilket blir switchen med lägst Bridge-ID. Detta är inkluderad i "Bridge Protocol Data Unit" (BPDU) och skickas av samtliga switchar.

Bridge Protocol Data Unit
-------------------------

![BDU-frame](/assets/images/2013/09/bdu-frame.png)
![BPDU](/assets/images/2013/09/bpdu.png)

Det finns två typer av BPDU-paket:

*   Configuration BPDU - Skickas endast av Root-Bridge
*   Topology Change Notification (TCN) BPDU - Informerar om att en ändringar skett i topologin, skickas endast ut på Root-Port

Bridge-ID består av:

*   Bridge Priority - 0 till 61440,  kan bara sättas i inkrement av 4096
*   System ID Extension - 0 till 4095, används av PVST/PVST+, anger VLAN-id
*   MAC-Adress

Per default kommer alla switchar ha Bridge Priority satt till 32768, det blir således den switchen med lägst mac-adress (äldst) som väljs till till Root-Bridge. Varje switch kommer dock först sätta sig själv som Root-Bridge då de ännu inte har någon koll på hur topologin ser ut. Switchen gör detta genom att fylla i sin mac-adress under Root-ID & Bridge-ID och skickar ut  BPDU-paketet på alla aktiva länkar. BPDU-Hello timer är default satt till 2 sekunder.
```
VLAN0001
 Spanning tree enabled protocol ieee
 **Root ID Priority 32769**
 **Address **0024.c33f.9e80****
 Cost 12
 Port 56 (Port-channel1)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
**Bridge ID Priority 32769 (priority 32768 sys-id-ext 1)**
 **Address 0024.c33f.9e80**
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
 Aging Time 300
```
När en switch sedan tar emot ett BPDU-paket jämför den Root-ID:t med vad den själv har konfigurerat, är värdet bättre (lägre) byter den Root-bridge. De här bilderna visar förloppet väldigt bra: 
![switches-in-triangle](/assets/images/2013/09/switches-in-triangle.pn) 

Varje switch sätter sig själv som Root-Bridge och bygger ett BPDU-paket med Priority 32768 & dess MAC-adress. 

![switches-send-bpdu](/assets/images/2013/09/switches-send-bpdu.png) 

BPDU-paket skickas ut på alla aktiva interface. 
![stp-root-elected](/assets/images/2013/09/stp-root-elected.png) 

Switch B & C inser att Switch A har ett lägre Root-ID och sätter den som Root-Bridge.  Switcharna kommer sedan försöka räkna ut den bästa vägen till Root-Bridge och stänger av redundanta vägar till samma destination. När Switch A skickar ut sina BPDU-paket kommer den sätta något som kallas "root path cost", STPs metric, till **0**. När Switch B & C **tar emot paketet** adderar de först STP-costen till RPC enligt följande tabell:

*   4M - 250
*   10M - 100
*   100M - 19
*   1G - 4
*   10G - 2

Det är endast Root-Bridge som genererar BPDU-paket, Switch B & C vidarebefordrar bara paketen, dock med den uppdaterade "Root path costen" samt att de sätter sin egen mac-address under "Bridge-ID". Switch C kommer få ett BPDU-paket från Switch B som ser ut enligt följande: **Root-ID:** AAA **Bridge-ID:** BBB **Root-path-cost:** 19 

Switch B får följande från Switch C: **Root-ID:** AAA **Bridge-ID:** CCC **Root-path-cost:** 19 De jämför paketet med vad den fått från Switch A: **Root-ID:** AAA / AAA  / AAA **Bridge-ID:** AAA / BBB / CCC **Root-path-cost: 0** / 19 / 19 Både switch A & B kommer därför välja Fa0/0 som bästa vägen till Root-bridge pga **lägst** RPC. Denna port kallas "**Root-port**" och en switch kan endast ha en aktiv root-port. Om det skulle vara samma RPC över två länkar kommer switchen välja interfacet med lägst Interface-ID (Fa0/0>Fa0/1 etc). Spanning-Tree

*   Första versionen av STP, 802.1d
*   Kallas även "Common Spanning Tree" (CST)
*   Har endast en STP-topologi för samtliga VLAN

STP använder tre port-states:

*   **Root-Port** - Den port med lägst RPC till Root-Bridge
*   **Designated port** - Forwardar trafik, kan endast finnas en per länk
*   **Blocked/Non-designated port** - redundant port som stängts av

Root-Bridge har självfallet ingen Root-port utan sätter istället samtliga interface till "Designated Port" (D-P). Vi har redan räknat ut vilken port som blir Root-port för Switch B & C. Då vi endast kan ha en Designated Port per länk måste Switch B & C komma överens om vem som får sätta sin port som D-G för fa0/1, vinnaren blir återigen den med lägst Bridge-ID (mac-adress), dvs Switch B. Skulle vi ha två redundanta länkar mellan Switch B & C kommer ju även Bridge-ID vara identiskt, STP använder då istället lägst interface-ID. Det enda som återstår för Switch C blir att sätta sitt interface som Blocked/Non-Designated port. En blockerad port droppar alla inkommande frames men lyssnar fortfarande på BPDU-paket.  

Postade en bild på ytterligare ett exempel i ett tidigare inlägg för en lite mer avancerad topologi: 
![2013-08-31 21.56.49](/assets/images/2013/08/2013-08-31-21-56-49.jpg) 
I detta fall har jag satt SW E som root-bridge. Förhoppningsvis kan du nu räkna ut vad SW D kommer göra med sin länk upp till SW A & B. :) Vi bör alltid själva bestämma vilken switch som ska vara root-bridge (vi riskerar annars att det blir någon gammal skräp-switch tack vare lägst mac-adress) genom kommandot:

`spanning-tree vlan 1 root primary`

Detta sänker automatiskt priorityn med 8192 mindre än nuvarande root-bridge,
```
VLAN0001
 Spanning tree enabled protocol ieee
 **Root ID Priority 32769**
 Address 0014.a889.9c80
 This bridge is the root
S3(config)#spanning-tree vlan 1 root primary
S3(config)#do sh spanning-tree
VLAN0001
 Spanning tree enabled protocol ieee
 **Root ID Priority 24577**
 Address 0014.a889.9c80
 This bridge is the root
```
Vi kan även konfigurera en backup för att ta över vid ett evenuellt avbrott:

`spanning-tree vlan 1 root secondary`

Detta adderar 4096 till nuvarande root-bridge id:
```
VLAN0001
 Spanning tree enabled protocol ieee
 **Root ID Priority 24577**
 Address 0014.a889.9c80
 Cost 12
 Port 56 (Port-channel1)
 Hello Time 2 sec Max Age 20 sec Forward Delay 15 sec
**Bridge ID Priority 28673 (priority 28672 sys-id-ext 1)**
 **Address 0024.c33f.9e80**
```
Förutsatt att vi lämnat övriga switchar med default-värden är detta en bra lösning. Men för att vara på den säkra sidan kan vi även sätta ett manuellt priority-värde med:

`spanning-tree vlan 1 priority _n_ (0 -61440)`

På så vis kan vi även höjja priorityn för en switch vi absolut inte vill ska kunna bli Root-bridge.

### STP Portstates

En stor nackdel med STP är dess konvergenstid (upp till 50 sekunder) när en port från från down/blocking till forwarding. 
![stp-states](/assets/images/2013/09/stp-states.gif)
När vi tar upp ett nytt interface kommer följande alltid ske: 
1. Porten ställs i **Blocking**, switchen lyssnar på BPDU-frames för att verifera att porten är en giltig root- eller designated port. 
2. Porten ställs i **Listening**, använder Forward delay-timer (default 15 sek) där den skickar/tar emot BPDU-frames för att lära sig om Root-Bridge etc. 
3. Porten ställs i **Learning**, använder Forward delay-timer (default 15 sek) för att lära sig tillhörande mac-adresser. 
4. Porten ställs i **Forwarding** och vi kan nu skicka/ta emot data.
```
Switch(config-if)#no shut
00:04:21: set portid: VLAN0001 Fa0/1: new port id 8003
**00:04:21: STP: VLAN0001 Fa0/1 -> listening**
*Mar 1 00:04:22.135: %LINK-3-UPDOWN: Interface FastEthernet0/1, changed state to up
*Mar 1 00:04:23.142: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up
**00:04:36: STP: VLAN0001 Fa0/1 -> learning**
00:04:51: STP: VLAN0001 sent Topology Change Notice on Fa0/3
**00:04:51: STP: VLAN0001 Fa0/1 -> forwarding**
```
Om en blockerad port slutar ta emot BPDU-paket måste den först vänta ut dess "max-age" timer (20sek) innan den den kan få över till Listening. Detta gäller även för en Root-port, har vi inte mottaget ett BPDU-paket på 20 sekunder räknar switchen med att vägen till root-bridge är nere och kommer istället försöka hitta en ny root-port.  Max-age är tiden switchen cachar BPDU-information. I värsta fall tar det med andra ord 20 + 15 + 15 = 50 sekunder innan vi kan skicka data över en länk.

### STP Timers & Tuning

Då det endast är **Root-Bridge** som generar BPDU-paketen för STP/PVST/PVST+ är det endast där vi bör modifiera STP-timers. Hello-timer (2 sekunder default)

```
spanning-tree vlan x hello-time _n_

Forward-Delay (15 sekunder default)

spanning-tree vlan x forward-time _n_

Max-Age (20 sekunder default)

spanning-tree vlan x max-age _n_
```

Switchen kan även ta fram optimala värden åt oss beroende på hur många switch-hopp det finns i vår topologi via kommandot: spanning-tree vlan x root primary diameter _n_
```
S1(config)#spanning-tree vlan 1 root primary diameter 3

VLAN0001
 Spanning tree enabled protocol ieee
 Root ID Priority 20481
 Address 0024.c33f.9e80
 This bridge is the root
 **Hello Time 2 sec Max Age 12 sec Forward Delay 9 sec**
```
Vi kan även modifiera RPC för ett interface via: if)#spanning-tree vlan x cost _n_ (1-200000000) Och port-priority med:

`spanning-tree vlan x port-priority _n_ (0-240, increments of 16)`

### STP Topology Change

Om en förändring sker i vår topologi, dvs att ett interface går upp/ner, kommer den aktuella switchen att skicka ett "BPDU  Topology Change Notification"-paket (BPDU TCN) ut på sin root-port. Detta vidarebefordras sedan av övriga switchar tills den når root-bridge.
```
Switch(config-if)#shut
**00:04:11: STP: VLAN0001 sent Topology Change Notice on Fa0/3**
*Mar 1 00:04:13.335: %LINK-5-CHANGED: Interface FastEthernet0/1, changed state to administratively down
*Mar 1 00:04:14.342: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to down
Switch(config-if)#no shut
......
**00:04:51: STP: VLAN0001 sent Topology Change Notice on Fa0/3**
00:04:51: STP: VLAN0001 Fa0/1 -> forwarding
```
Packetet ack:as av root-bridge som sedan sätter TCN-biten (Topology Change Notification) i nästa BPDU Configuration-frame. Detta säger åt underliggande switchar att minska sin "Bridge-table aging time" från default 300 sek till dess Forward-Delay value (15sek). Detta flushar ut gamla mac-adresser mycket snabbare vilket i sin tur ger switcharna en uppdaterad bild av hur den eventuellt nya topologin ser ut. Problemet är dock att detta skickas ut oavsett vilket interface som går upp/ner, dvs även host-portar har denna effekt på nätverket. Har vi ett nät med 500 användare är det inte ofta vår aging-time kommer vara över 15 sekunder. För att lösa detta infördes "PortFast", vilket jag kommer berätta mer om i ett senare inlägg. En av fördelarna med PortFast är dock att switchen EJ skickar ut någon TCN-frame om en förändring sker på interfacet.

PortFast
--------

Används för "edge ports", dvs edge portar till hosts/voip-telefoner/routrar där vi endast har en inkopplad enhet (och som inte använder STP). PortFast sätter Forward Delay till 0 vilket gör att porten direkt kan gå från Down/Blocking till Forwarding. Som vi nämnde ovan skickas det inte heller några BPDU TCNs vid förändringar på interfacet. Däremot så lyssnar porten så att det inte kommer in några BPDU-paket, om så är fallet kommer porten omedelbart tappa PortFast-funktionen och istället använda "vanliga" STP-timers.
```
interface FastEthernet0/1
 spanning-tree portfast
```
Vi kan verifiera vilka interface som har PortFast konfigurerat via show spanning-tree.
```
Interface Role Sts Cost Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Fa0/1 Desg FWD 19 128.3 **P2p Edge** 
Fa0/2 Desg FWD 19 128.4 P2p 
Fa0/3 Root FWD 19 128.5 P2p 
Fa0/4 Altn BLK 19 128.6 P2p 
Fa0/6 Desg FWD 19 128.8 P2p 
Fa0/24 Desg FWD 19 128.26 P2p
```
En debug visar hur mycket snabbare det nu går att få upp interfacet igen, observera även att det inte skickas något TCN-paket.
```
Switch(config-if)#no shut
00:11:20: set portid: VLAN0001 Fa0/1: new port id 8003
00:11:20: STP: VLAN0001 Fa0/1 ->jump to forwarding from blocking
*Mar 1 00:11:20.467: %LINK-3-UPDOWN: Interface FastEthernet0/1, changed state to up
*Mar 1 00:11:21.473: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up
Switch(config-if)#
```
Per-VLAN Spanning Tree (PVST)
-----------------------------

![pvst](/assets/images/2013/09/pvst.png)
En Cisco proprietär lösning som skapar en Spanning-tree instans per VLAN, vi kan således ha en Root-Bridge för Vlan 10 och en för Vlan 20 etc. Ger oss möjlighet att skicka trafik för vlan 20 över en länk som vlan 10 blockerat och vice versa, vi tappar på så vis inte  så mycket av den potentiella prestandan. Nackdelen är att den kräver att vi använder ISL som trunking-protokoll.

### Per-VLAN Spanning-Tree+ (PVST+)

En förbättrad version av föregångaren som tillåter oss använda både 802.1q & ISL som trunking-protokoll. Här finns lite mer intressant information att läsa om vilka mer skillnader det finns: [http://cciethebeginning.wordpress.com/2008/10/15/differences-between-pvst-and-pvst/](http://cciethebeginning.wordpress.com/2008/10/15/differences-between-pvst-and-pvst/)
