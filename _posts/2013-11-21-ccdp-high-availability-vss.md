---
title: CCDP - High Availability & VSS
date: 2013-11-21 14:09
comments: true
categories: [VSS]
tags: [hsrp, vrrp,glbp, mec, vsl]
---
Tänkte skriva ett inlägg om Cisco's VSS, vilket är en Cisco-proprietär lösning för att clustra två (eller fler, implementeras i par) fysiska 6500/4500 switch-chassin till en virtuell switch, vilket vi kommer se senare har en hel del fördelar sett till både prestanda & redundans. VSS finns just nu endast tillgängligt på Ciscos Catalyst 4500 & 6500-serie. ![vss6500](/assets/images/2013/11/vss6500.png) http://www.youtube.com/watch?v=RVwnXMVeKNs

# High Availability

### Layer 2 Looped

![l2-loopad](/assets/images/2013/11/l2-loopad.jpg)

Ovanstående är väl den vanligaste lösningen som åtminstone jag fått lära mig från CCNA/CCNP-spåret när vi vill ha redundans i access-lagret. Vi kör Lager 2 upp till distributions-lagret som i sin tur använder sig av något FHRP som HSRP eller VRRP. Detta leder dock i sin tur att endast en distributions-switch är aktiv gateway (den andra står i standby och tar endast emot ~50% av returtrafiken tack vare lastbalansering), redundanta länkar blockeras dessutom av spanning-tree. Vi använder helt enkelt endast ~50-60% av den tillgängliga prestandan. Genom att använda PVST/MST skulle vi dock åtminstone kunna lastbalansera mellan VLAN:en genom att göra D1 Active/RB för Vlan 20 och D2 Active/RB för Vlan 30 exempelvis, observera att STP-topologin kommer se annorlunda ut för de olika vlanen.

### Layer 2 Loopfree

![l2-loopfri](/assets/images/2013/11/l2-loopfri.jpg)
Om vi istället skulle konvertera etherchanneln mellan distributions-switcharna till en L3-länk kommer STP att släppa blockeringen på upplänkarna då det inte längre finns risk för någon loop, däremot är vi nu tvungna att använda oss av lokala vlan (vilket är en relativt ovanligt lösning fortfarande och kan vara svårt att realisera). Vi kommer fortfarande endast ha en aktiv gateway, men vi kan åtminstone dela upp detta mellan distributions-switcharna för att få lastbalansering per vlan.

### Layer 3 Routed

![l3-routed](/assets/images/2013/11/l3-routed.jpg)
En tredje lösningen är att använda oss av ett helt routat nät, dvs köra Lager 3 hela vägen ner till access-lagret och låta exempelvis OSPF sköta hanteringen av redundanta länkar. Vi slipper även använda STP och det finns nu möjlighet att faktiskt använda all tillgänglig kapacitet då vi inte heller behöver något FHRP. Detta är dock en betydligt dyrare lösning då det kräver access-switchar med MLS-funktioner och är därför inte heller den speciellt vanlig. Cisco's GLBP är även en tänkbar lösning för Layer 2-topologierna då det tillåter oss att ha flera aktiva gateways för ett och samma vlan, men verkar inte finnas implementerat i speciellt många switch-modeller ännu. Mer info om GLBP finns [här](http://blog.ine.com/2008/04/24/glbp-explained/) & [här](http://www.cisco.com/en/US/docs/ios/12_2t/12_2t15/feature/guide/ft_glbp.html)!

# Virtual Switching System

![vss-logical](/assets/images/2013/11/vss-logical.png) 
Då VSS slår samman distributions-switcharna till en logisk enhet ger det oss flera av fördelarna i likhet med den routade modellen. Vi behöver inte oroa oss för Spanning-tree och behöver heller inte använda oss av ett FHRP för Gateway-redundans.  Det ger oss även möjlighet att sätta upp en variant av Etherchannel (MEC) trots att vi terminerar kablarna i två skilda chassin, mer om detta längre ner i inlägget. ![VSS-domain](/assets/images/2013/11/vss-domain.png) VSS använder sig av SSO & NSF för att snabbt kunna svänga över till Standby-routern om ett fel skulle inträffa. Den aktiva switchen sköter alla "Control Plane"-funktioner som exempelvis:

*   Management (SNMP, Telnet, SSH)
*   Layer 2 Protokoll (BPDU, PDUs, LACP, PAgP etc)
*   Layer 3 Protokoll (OSPF, EIGRP etc)
*   Software data

Observera dock att "Data Plane" fortfarande är aktivt på Standby-switchen och används fullt ut till skillnad mot exempelvis HSRP! Både Active & Standby sköter individuellt forwarding lookups för inkommande trafik via "Policy Feature Card (PFC)". Vi kan verifiera vilken switch som är aktiv via kommandot "show switch virtual":

```
vss#show switch virtual
Switch mode: Virtual Switch
Virtual switch domain number: 200 
Local switch number: 1 
Local switch operational role: Virtual Switch Active 
Peer switch number: 2 
Peer switch operational role: Virtual Switch Standby
```

## Virtual-MAC

![vss-mac](/assets/images/2013/11/vss-mac.png)
När vi bootar upp vår virtuella switch kommer den som är Active att sätta sin MAC-adress på samtliga Layer 3-interface, standby-switchen hämtar denna information och använder samma MAC på sina egna interface. Även om ett avbrott inträffar och Standby slår över till Active kommer den fortfarande att behålla denna MAC-adress! Startar vi däremot om båda switcharna och Dist-2 behåller sin roll som active kommer mac-adressen att ändras (till 5678 i ovanstående exempel). Det finns även möjlighet att istället använda en virtuell mac-adress för att försäkra sig om att MAC-adressen alltid kommer vara densamma oavsett vilken av switcharna som blir aktiv efter omstart.  Switchen räknar själv ut adressen genom att kombinera VSS-gruppnumret med en pool av VSS-adresser. Konfigurationen är enligt följande:

```
VSS(config-vs-domain)# switch virtual domain 100
VSS (config-vs-domain)#mac-address use-virtual
Configured Router mac address (0008.e3ff.fd34) is different from operational 
value (0013.5f48.fe40). Change will take effect after the configuration is saved 
and the entire Virtual Switching System (Active and Standby) is reloaded.
VSS(config-vs-domain)#
```
## VSL - Virtual Switch Link

En speciell typ av Etherchannel sätts upp mellan switcharna, VSL, för att ge den aktiva switchen möjlighet att kontrollera hårdvaran i standby-switchen. Den används även för att skicka information om "line card status", Distributed Forwarding Card (DFC) programmering, system management, diagnostik men även vanlig data-trafik när det är nödvändigt. 

![vss-vsh](/assets/images/2013/11/vss-vsh.png)

För att försäkra sig om att VSL-control frames har prioritet över vanlig datatrafik enkapsuleras data i en speciell "Virtual Switch Header (VSH)" på 32 bitar och sätter Priority-biten till 1. All trafik lastbalanseras och använder underliggande Etherchannel-protokollen LACP eller PAgP för att sätta upp kanalen. 6500 Switcharna har förövrigt en hel del mer hashing-scheman att använda till skillnad mot 3650 vi använder i skolan. ;)

```
vss(config)#port-channel load-balance ? 
dst-ip Dst IP Addr 
dst-mac Dst Mac Addr 
dst-mixed-ip-port Dst IP Addr and TCP/UDP Port 
dst-port Dst TCP/UDP Port 
mpls Load Balancing for MPLS packets 
src-dst-ip Src XOR Dst IP Addr 
src-dst-mac Src XOR Dst Mac Addr 
src-dst-mixed-ip-port Src XOR Dst IP Addr and TCP/UDP Port 
src-dst-port Src XOR Dst TCP/UDP Port 
src-ip Src IP Addr 
src-mac Src Mac Addr 
src-mixed-ip-port Src IP Addr and TCP/UDP Port 
src-port Src TCP/UDP Port
```
Har vi dessutom ett Supervisor Engine 2T-kort kan vi använda lastbalansera baserad på VLAN-information:
```
VSS2T(config)# port-channel load-balance ?
dst-ip Dst IP Addr 
dst-mac Dst Mac Addr 
dst-mixed-ip-port Dst IP Addr and TCP/UDP Port
dst-port Dst TCP/UDP Port
mpls Load Balancing for MPLS packets
src-dst-ip Src XOR Dst IP Addr 
src-dst-mac Src XOR Dst Mac Addr 
src-dst-mixed-ip-port Src XOR Dst IP Addr and TCP/UDP Port
src-dst-port Src XOR Dst TCP/UDP Port
src-ip Src IP Addr 
src-mac Src Mac Addr 
src-mixed-ip-port Src IP Addr and TCP/UDP Port
src-port Src TCP/UDP Port
vlan-dst-ip Vlan, Dst IP Addr 
vlan-dst-mixed-ip-port Vlan, Dst IP Addr and TCP/UDP Port
vlan-src-dst-ip Vlan, Src XOR Dst IP Addr 
vlan-src-dst-mixed-ip-port Vlan, Src XOR Dst IP Addr and TCP/UDP Port
vlan-src-ip Vlan, Src IP Addr 
vlan-src-mixed-ip-port Vlan, Src IP Addr and TCP/UDP Port
```
Fantastiskt nog har Cisco även implementerat en ny funktion för att faktiskt verifiera vilken väg trafiken tar, något som var oerhört omständigt tidigare, är väl bara att hoppas att detta även kommer i nyare IOS-versioner för de "enklare" switcharna också sen. Det har även implementeras något som kallas "Adaptive Load-balacing", vilket gör att switchen inte längre behöver behöver resetta sina portar om ett interface tas bort/läggs till i etherchanneln (avbrott som annars varade ~200-300ms, vilket på en 10Gb-länk kan vara en hel del förlorad data).

```
vss#sh etherchannel load-balance hash-result ?
interface Port-channel interface
ip IP address
ipv6 IPv6 
l4port Layer 4 port number 
mac Mac address 
mixed Mixed mode: IP address and Layer 4 port number 
mpls MPLS 
vss#sh etherchannel load-balance hash-result interface port-channel 120 ip 192.168.220.10 192.168.10.10 
Computed RBH: 0x4 
**Would select Gi1/2/1 of Po120**
```

![vss-vslboot](/assets/images/2013/11/vss-vslboot.png)
När vi aktiverat VSS på switcharna används RRP för att bestämma vilken av switcharna som ska ta rollen som Active. Vi kan på förhand bestämma vilken vi vill ha som aktiv genom att konfigurera priority på båda switcharna. Det finns även möjlighet till att konfigurera preemption, men detta avråder man ifrån att använda i Arch-boken då det leder till längre konvergenstid. VSL Etherchanneln måste bestå av minst två 10Gb-länkar (max 8) och ska helst termineras i två skilda linjekort för maximal redundans.

## Multichassis EtherChannel (MEC)

![vss-mec](/assets/images/2013/11/vss-mec.png)
En av de största fördelarna med VSS förutom den ökade redundansen är möjligheten att sätta upp en variant på Etherchannel trots att den fysiska kopplingen går till två skilda chassin. Lösningen kallas Multichassis Etherchannel (MEC) och kan konfigureras som både L2 eller L3. I princip fungerar MEC precis som en helt vanlig Etherchannel men något som däremot skiljer sig är hur lastbalanseringen utförs. 

Istället för att endast använda en hashing-algoritm för att bestämma vilken port som skall användas prioriterar MEC alltid lokala interface över interface den behöver gå via VSL-länken för att nå. För trafik som måste floodas ut på VLANet (broadcast, multicast eller unknown unicast) skickas en kopia av paketet över VSL-länken, men från den [dokumentation jag hittat](http://www.cisco.com/en/US/prod/collateral/switches/ps5718/ps9336/white_paper_c11_429338.pdf) ignoreras tydligen detta paket av mottagaren då den förutsätter att paketet redan skickats ut på LAN:et vilket låter lite märkligt, så hur väl detta stämmer låter jag vara osagt... 

Inkommande "Control Plane"-trafik som STP, PAgP eller VTP-data kommer däremot att vidarebefordras över VSL-länken till Active-switchen då det bara finns en aktiv RP. MEC har förövrigt stöd för både PAgP & LACP och alla förhandlingar sköts av Active-switchen. Om samtliga lokala interface på något av chassina skulle gå ner konverteras MEC till en vanlig Etherchannel och trafiken fortsätter skickas över VSL-länken istället utan något längre avbrott.

# Redundans

Standby-switchen använder följande funktioner för att upptäcka eventuella fel som uppstår på primären:

*   VSL Protocol
*   Cisco Generic Online Diagnostics (GOLD) failure event
*   CDL-based hardware assistance
*   Full VSL-link down

![vss-failover](/assets/images/2013/11/vss-failover.png)
Om standby upptäcker ett avbrott initieras en SSO switchover och den tar över rollen som Active vilket enligt Cisco ska ta under sekunden att göra.  Om däremot det endast är en av VSL-länkarna som går ner fortsätter switcharna precis som vanligt, däremot kommer trafik som skickas över återstående VSL-länkar att få en höjd delay på ~50-100ms. Om däremot samtliga VSL-länkar går ner uppstår det en hel del problem! 

![vss-dualactive](/assets/images/2013/11/vss-dualactive.png)
Standby upptäcker att den inte längre har kontakt med Active och initierar därför en SSO failover och går själv över till Active-state. Men switchen som redan var i Active kommer däremot endast tro att den tappat kontakt med Standby och fortsätter som vanligt. Vi har nu två switchar som båda är i Active och som bl.a. använder samma IP- & MAC-adresser. Detta leder ju i sin tur till en hel del problem som ip-konflikter och BPDU's som skickas med samma Bridge-ID m.m. Det är därför väldigt viktigt med hög redundans för just VSL-länkarna och Cisco's rekommendationer med att vi terminerar anslutningarna i olika linjekort.  Cisco har även implementerat ytterligare några säkerhetsfunktioner för att upptäcka detta så snabbt som möjligt:

*   Enchanced PAgP
*   Layer 3 BFD
*   Fast Hello

![vss-pagp](/assets/images/2013/11/vss-pagp.png)
Enhanced PAgP skickar vid avbrott på VSL-länken direkt ut ett PAgP-meddelande med TLV-fältet innehållandes dess VSS Active ID. Om de upptäcker en mismatch kommer den ena switchen direkt ställa sig i Recovery-mode istället. Vi aktiverar detta via:

```
vss#conf t 
Enter configuration commands, one per line. End with CNTL/Z. 
vss(config)#switch virtual domain 10 
vss(config-vs-domain)#dual-active detection pagp 
vss(config-vs-domain)#dual-active trust channel-group 20 
vss(config-vs-domain)#
```

![vss-fasthellos](/assets/images/2013/11/vss-fasthellos.png)
Fast Hello's kräver en dedikerad L2-länk mellan switcharna som endast används för att skicka just Fast Hellos, konfigen är dock väldigt simpel.

```
vss(config)# interface fastethernet 1/2/40
vss(config-if)# dual-active fast hello
WARNING: Interface FastEthernet1/2/40 placed in restricted config mode. All 
extraneous configs removed!
vss(config)# switch virtual domain 10
vss(config-vs-domain)# dual-active detection fast hello
vss(config-vs-domain)# exit
```
BFD spar vi till en annan gång då det känns värdigt ett helt eget inlägg. :
