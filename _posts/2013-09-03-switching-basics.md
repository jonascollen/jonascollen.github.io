---
title: Switching - Basics
date: 2013-09-03 21:57
comments: true
categories: [Switch]
tags: [vtp,dtp,isl,dot1q]
---
Enterprise Composite Model
--------------------------

Cisco jobbar för att vi ska gå ifrån den hierarkiska modellen vi lärde oss i CCNA: 
![hier-topology](/assets/images/2013/09/hier-topology.gif) 

Och istället gå över till den betydligt mer heltäckande Enterprise Composite Model som ser ut enligt följande: 

[![comp-top1](/assets/images/2013/09/comp-top1.gif)
![comp-top2](/assets/images/2013/09/comp-top2.gif) 

Och lite riktlinjer att följa:

*   Endast riktigt stora företag har ett Core-lager, vanligaste implementeringen är istället att ha en "Collapsed Core" tillsammans med Distributions-lagret
*   Separera Server-lanet från övrig verksamhet
*   Separera Voice-trafik från övrig verksamhet
*   Designa nätet i "block" gällande redundans/routing/vlan, detta för att förenkla QoS, nätsummering, förbättra säkerheten och få bättre L2-segmentering . Försök håll en fördelning på 80(extern)/20(intern) trafik inom vlanen
*   Implementera multicast-support i hela nätet
*   Separat management-nät
*   Håll VLAN:en inom sitt respektive block och använd istället L3-routing i access/distributions-lagret mellan byggnader/avdelningar

Trunking
--------

För att upprätta en fungerande trunk-länk bör följande matcha:

*   Native VLAN
*   Encapsulering (ISL/Dot1q)
*   DTP-Mode

Konfigureras genom:
```
switchport trunk encapsulation _n_
switchport mode trunk
switchport nonegotiate
switchport trunk allowed vlan _n_
```
Native VLAN används främst för VoIP-implementationer idag. 
![native-vlan](/assets/images/2013/09/native-vlan.jpg)

### Inter-Switch Link (ISL)

*   Cisco Proprietärt
*   Encapsulerar hela orginal-framen och lägger till en ny 26 bytes header & 4 byte CRC
*   30 byte overhead
*   Supportar ej Native-VLAN/untagged
*   Är på väg att fasas ut och är inget som används i nyare implementationer

![ISL](/assets/images/2013/09/isl.jpg)
Endast 4 byte av den nya ISL-headern används idag, "VLAN ID" & BPDU, resterande fält räknas nu som "skräp-data".

### 802.1Q (dot1q)

*   Open/Industry Standard
*   Lägger endast till en 4 bytes VLAN-Tag till den befintliga L2-framen

![dot1q](/assets/images/2013/09/dot1q.png) 

**Tag protocol** - Informerar om att det är en 802.1Q-tag, har alltid värdet 0x8100.
**User Priority/Tag Control Information** - 3 bitar att använda till CoS-märkning (Class of Service, QoS). 
**Canonical Format Indicator** - 1 bit för att informera om det är en Ethernet- eller Token Ring-frame. 
**VLAN ID** - Informerar om vilket VLAN framen hör till Tack vare den betydligt lägre overheaden samt stödet för Quality of Service är det enkelt att se varför dot1q är förstahandsvalet när det kommer till trunking. Om vi dock inte manuellt sätter dot1q utan istället använder oss av "`switchport trunk encapsulation negotiate`" kommer switcharna att defaulta till ISL-encapsulering, hejja cisco! :)

Dynamic Trunking Protocol (DTP)
-------------------------------

Kopplar vi ihop två cisco-switchar med varandra kommer de med största sannolikhet upprätta en trunk-länk automatiskt tack vare DTP. DTP låter oss dynamiskt sätta upp trunk-länkar och har vi har följande konfigureringsmöjligheter för en switchport:

*   Access
*   Trunk
*   Dynamic auto
*   Dynamic desirable (Cisco factory default)
*   No-negotiate
*   Tunnel (dot1q-tunnel, mer om detta senare)

![DTP](/assets/images/2013/09/dtp.png)
Det är rekommenderat att alltid konfigurera trunk-länkar till No-negotiate trunk, och access till just no-negotiate access. Vi kan verifiera status med "show dtp [interface]".
```
switchport mode _n_
switchport nonegotiate
```
Vlan Trunking Protocol (VTP)
----------------------------

Ger oss möjlighet att centralt skapa/modifiera/ta bort vlan på ett stort antal switchar.

*   Annonserar endast vlan mellan 1-1005
*   Switchar använder revisions-nummer för att hålla koll på om det är ny/gammal information (börjar alltid på 0)
*   Vi kan resetta revisions-numret via att byta vtp domän eller sätta switchen i VTP Transparent-mode temporärt
*   Talar ej om vilka interface som hör till respektive VLAN
*   Skickas endast över trunk-länkar som multicast

Kopplar vi in en gammal/begagnad switch med ett högre revisions-nummer till ett befintligt LAN kommer vlan.dat skrivas över och vi blir av med alla vår vlan-inställningar i samtliga VTP Servers & Clients. VTP finns i tre lägen:

*   Server (default) - Tillåter att vi gör ändringar i vlan-konfigurationen.,skickar/tar emot VTP-paket
*   Client - Tillåter EJ att vi gör några ändringar i vlan-konfigurationen, skickar/tar emot VTP-paket
*   Transparent - Lyssnar ej på VTP-paket den får från andra switchar, vi tillåts göra ändringar men detta sparas endast lokalt på switchen. Vidarebefodrar däremot VTP-paket i version 2 & 3.

För att kunna konfigurera "extended-range vlans" (1006-4095) krävs det att switchen är i Transparent-mode för -VTP version 1 & 2, version 3 har fullt stöd även i Server- & Client-mode. När vi konfigurerar switchen till VTP Transparent-mode sparas vlan-ändringar i running-config! VTP-advertisements skickas av switchar i Client-mode vid uppstart där de ber om en uppdatering, samt från switchar i Server-mode var 300:e sekund.

*   Summary Advertisement - Skickas av VTP Server var 300:e sekund samt varje gång en förändring i VLAN-databasen sker. Inkluderar information om VTP version, domän, revisionsnummer, tidsstämpel, md5 kryptering, hash code och antalet "Subset advetisements" som följer.
*   Subset Advertisement - Skickas av VTP Servern när en VLAN-förändring sker. Specificerar vad exakt som hänt med ett vlan (lagt till/tagit bort etc). Innehåller följande information: Status för VLANet, VLAN Type, MTU, Lenght of Vlan name, vlan number, SAID, Vlan name. Det skickas ett Subset advertisement per VLAN.

Konfigureras enligt:
```
vtp mode _n_
vtp domain _n_
vtp password _n_
vtp version _n_
```

Det finns även möjlighet till VLAN-pruning (vtp pruning) som begränsar automatiskt vilka vlan som tillåts över en trunk-port för att begränsa broadcast-domäner. Detta fungerar dock endast där vi använder VTP Server-mode. Rekommenderar att läsa följande artikel! Allt du behöver veta om VTP till CCNP/CCIE [här](http://www.cisco.com/en/US/docs/switches/lan/catalyst3560/software/release/12.2_52_se/configuration/guide/swvtp.html).

VLANs
-----

*   12 bit field
*   Vlan 0 & 4095 är reserverade
*   "Normal VLANs" sträcker sig mellan Vlan 1 - 1005
*   1002 & 1004 är reserverade för FDDI
*   1003 & 1005 är reserverade för Token Ring
*   "Extended VLANs" sträcker sig mellan Vlan 1006 - 4094

Switchen har ett separat CAM-table för varje VLAN. Får se om jag även tar mig tid att skriva ett kortare inlägg om STP/RSTP och CAM/TCAM/CEF senare, mycket av det känns dock väldigt basic så blir i slutändan bara slöseri med tid för egen del. Kommer hellre igång med det lite mer avancerade snabbare istället.. ;)
