---
title: OSPF - Fast Hellos
date: 2013-06-24 21:57
comments: true
categories: [CCNP]
tags: [fast hello]
---
Som bekant använder sig OSPF per default av hello-timers på 10 alt. 30 sekunder (deadtimer= 4xhello) beroende på network type. Detta betyder i sin tur att det kan ta upp till 40sek/2 minuter innan OSPF upptäcker att ens neighbor är nere, i en företagsmiljö är det i princip otänkbart, framförallt om de exempelvis använder sig av VoIP etc. Vi har dock möjlighet att justera både hello- & deadtimers som jag tror nämnts i något tidigare inlägg. Standardkonfigurationen av OSPF tillåter oss sätta hello's i följande intervall:
```
R1(config-if)#ip ospf hello-interval ?
 <1-65535> Seconds
```
Och samma sak gäller för deadtimer:
```
R1(config-if)#ip ospf dead-interval ?
 <1-65535> Seconds
 minimal Set to 1 second
```
Observera att dead-interval inte behöver vara 4xHello, utan det är endast en default-setting. Vi kan exempelvis sätta hello-timern till 1 sekund och dead-interval till 2 sekunder. Kom ihåg att detta måste göras på båda routrarna för att adjacency skall skapas. Låt oss testa detta i en enkel topologi. 

[![ospf fast hello](/assets/images/2013/06/ospf-fast-hello.png)](/assets/images/2013/06/ospf-fast-hello.png) R1
```
interface Serial0/0
 ip address 10.0.0.1 255.255.255.252
 ip ospf hello-interval 1
 ip ospf dead-interval 2
```
R2
```
interface Serial0/0
 ip address 10.0.0.2 255.255.255.252
 ip ospf hello-interval 1
 ip ospf dead-interval 2
```
Ger oss följande output av sh ip ospf int s0/0 på R1:
```
R1(config-if)#do sh ip ospf interface s0/0
Serial0/0 is up, line protocol is up 
 Internet Address 10.0.0.1/30, Area 0 
 Process ID 1, Router ID 1.1.1.1, Network Type POINT\_TO\_POINT, Cost: 64
 Transmit Delay is 1 sec, State POINT\_TO\_POINT
 **Timer intervals configured, Hello 1, Dead 2, Wait 2, Retransmit 5**
 oob-resync timeout 40
 Hello due in 00:00:00
 Supports Link-local Signaling (LLS)
 Cisco NSF helper support enabled
 IETF NSF helper support enabled
 Index 1/1, flood queue length 0
 Next 0x0(0)/0x0(0)
 Last flood scan length is 1, maximum is 1
 Last flood scan time is 0 msec, maximum is 0 msec
 Neighbor Count is 1, Adjacent neighbor count is 1 
 Adjacent with neighbor 2.2.2.2
 Suppress hello for 0 neighbor(s)
```
[![wireshark hellotimer](/assets/images/2013/06/wireshark-hellotimer.png)](/assets/images/2013/06/wireshark-hellotimer.png)

Men för vissa kritiska applikationer/VoIP så är kanske inte ens 1 sekund tillräckligt. Det är då vi kan använda oss av OSPF Fast Hellos, detta tillåter oss att justera Hello-timers ner till så lågt som  50ms (högst 333ms/lägst 50ms). Vi konfigurerar detta med följande kommando:
```
interface serial0/0
R1(config-if)#ip ospf dead-interval minimal hello-multiplier ?
 <3-20> Number of Hellos sent within 1 second
R1(config-if)#ip ospf dead-interval minimal hello-multiplier 5
```
Multipliern bestämmer hur många Hello-paket som skall skickas under 1 sekund, i detta fall blir det 5st, 1/5 = 200ms per hello. Då det lägsta värdet vi kunde konfigurera för Hello dock var 1 sek, så kommer OSPF automatiskt ändra biten för varje Hello-paket som skickas till 0 sekunder istället, likaså kommer OSPF att ignorera hello-timern för paketen den tar emot på interfacet.
```
Serial0/0 is up, line protocol is up 
 Internet Address 10.0.0.1/30, Area 0 
 Process ID 1, Router ID 1.1.1.1, Network Type POINT\_TO\_POINT, Cost: 64
 Transmit Delay is 1 sec, State POINT\_TO\_POINT
 **Timer intervals configured, Hello 200 msec, Dead 1, Wait 1, Retransmit 5**
 oob-resync timeout 40
 Hello due in 70 msec
 Supports Link-local Signaling (LLS)
 Cisco NSF helper support enabled
 IETF NSF helper support enabled
 Index 1/1, flood queue length 0
 Next 0x0(0)/0x0(0)
 Last flood scan length is 1, maximum is 1
 Last flood scan time is 0 msec, maximum is 0 msec
 Neighbor Count is 1, Adjacent neighbor count is 1 
 Adjacent with neighbor 2.2.2.2
 Suppress hello for 0 neighbor(s)
```
I wireshark kan vi se att det är lite mer fart på paketen nu: 

[![ospf fast hello wireshark](/assets/images/2013/06/ospf-fast-hello-wireshark.png)](/assets/images/2013/06/ospf-fast-hello-wireshark.png) 

[![fast hello wireshark packet](/assets/images/2013/06/fast-hello-wireshark-packet.png)](/assets/images/2013/06/fast-hello-wireshark-packet.png) 

Observera dock att Deadtimern alltid är 1 sekund och går ej att få lägre, skillnaden på att sätta en högre multiplier är endast att den kommer skicka fler hello-paket under det  intervallet. Att modifiera hello/dead-timers till så här låga värden kräver dock lite eftertänksamhet. Vad händer exempelvis om du vid hög last på länken eller CPUn och får en delay som överstiger deadtimern (1s)? Du kommer tappa neighbor adjacency trots att det inte är något egentligt fel det avbrottet kan bli betydligt längre än de 40 sekunder vi hade innan om du har otur.. Detta avslutar OSPF-inlägget för ett tag framöver tror jag. Next up - Route filtering/redistribution! :)
