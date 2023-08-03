---
title: Switching - Etherchannel
date: 2013-09-07 21:30
comments: true
categories: [Switch]
tags: [etherchannel]
---
Etherchannel ger oss möjlighet att aggregera redundanta länkar, och vi kan på så vis ta vara på all potentiell bandbredd istället för att länkarna ska blockeras av Spanning-tree. Nedanstående video från Keith Barker förklarar konceptet väldigt bra: http://www.youtube.com/watch?v=CTcToRNonB8

*   Interfacen som används måste ligga i samma VLAN alt. vara trunk-portar
*   Kan ej använda SPAN
*   Ändringar i det logiska port-channel interfacet påverkar alla portar i gruppen (channel-group)
*   Ändringar på enskilt interface återspeglas ej till gruppen, alla interface i gruppen måste ha identisk konfig!
*   Rekommenderat att konfigurera Speed & Duplex manuellt på samtliga länkar
*   Supportar länkhastigheter från 100M upp till 10 Gigabit
*   Etherchanneln kan användas som en L2 access-port, trunk, tunnel eller L3-interface

Etherchannel har stöd för att aggregera 2 till maximalt 8 aktiva länkar!

Protokoll
---------

![etherchannelprotocol](/assets/images/2013/09/etherchannelprotocol.jpg) 
![negotiation_modes](/assets/images/2013/09/negotiation_modes.png) 

Som synes måste vi konfigurera Active eller Desirable på åtminstone en av switcharna för att det ska bildas en Etherchannel om vi använder oss av protokollen LACP eller PAgP.  Vi kan även "hårdkonfa" en Etherchannel utan att använda PAgP eller LACP via kommandot "channel-group x mode on". När vi konfigurerar Etherchannel rekommenderar Brian Dennis (CCIEx4) från INE att man alltid först stänger ner berörda interface för att undvika en del buggar som kan uppstå, detta är dock inget som nämns i Cisco materialet.

### Port Aggregation Protocol (PAgP)

*   Cisco proprietärt
*   Använder channel-modes: desirable & auto

![netlab](/assets/images/2013/09/netlab.png)

S1
```
S1(config)#int range fa0/3 - 4
S1(config-if-range)#shut
S1(config-if-range)#channel-protocol pagp
S1(config-if-range)#channel-group 1 mode desirable
S1(config-if-range)#no shut
**Creating a port-channel interface Port-channel 1**
*Mar 1 00:14:54.435: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/3, changed state to up
*Mar 1 00:14:54.879: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/4, changed state to up
```
Observera dock att port-channel interfacet ej kom upp! Då vi konfade interfacet till desirable kommer S1 nu börja skicka PAgP-paket till S3, men den kommer inte skapa Etherchanneln förrän den fått svar. S3
```
S3(config)#int range fa0/3 - 4
S3(config-if-range)#shut
S3(config-if-range)#channel-protocol pagp
S3(config-if-range)#channel-group 1 mode auto
S3(config-if-range)#no shut
**Creating a port-channel interface Port-channel 1**
*Mar 1 00:15:46.167: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/4, changed state to up
*Mar 1 00:15:46.285: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/3, changed state to up
***Mar 1 00:15:52.039: %LINK-3-UPDOWN: Interface Port-channel1, changed state to up**
***Mar 1 00:15:53.046: %LINEPROTO-5-UPDOWN: Line protocol on Interface Port-channel1, changed state to up**
```
S3 börjar nu svara på S1's förfrågan om att skapa en Etherchannel och interfacet går därför upp både på S1 & S3, vilket tog cirka 7 sekunder i min topologi med 2st 3560's.
```
 1#show etherchannel summary
 Flags: D - down P - bundled in port-channel
 I - stand-alone s - suspended
 H - Hot-standby (LACP only)
 R - Layer3 S - Layer2
 U - in use f - failed to allocate aggregator
 M - not in use, minimum links not met
 u - unsuitable for bundling
 w - waiting to be aggregated
 d - default port

Number of channel-groups in use: 1
Number of aggregators: 1
Group Port-channel Protocol Ports
------+-------------+-----------+-----------------------------------------------
1 Po1(SU) **PAgP** Fa0/3(P) Fa0/4(P)
```
Unikt för PAgP är även funktionen "**non-silent**":
```
S3(config-if)#channel-group 1 mode desirable ?
 non-silent Start negotiation only after data packets received
S3(config-if)#channel-group 1 mode auto ?
 non-silent Start negotiation only after data packets received
```
Cisco ger följande rekommendationer för hur vi ska konfigurera detta:

*   Use the non-silent keyword when you connect to a device that transmits bridge protocol data units (BPDUs) or other traffic. Use this keyword with the auto or desirable mode. PAgP non-silent adds an extra level of link state detection because it listens for BPDUs or other traffic in order to determine if the link functions properly. This adds a form of UniDirectional Link Detection (UDLD) capability that is not available when you use the default silent PAgP mode.
*   Use the silent keyword when you connect to a silent partner (which is a device that does not generate BPDUs or other traffic). An example of a silent partner is a traffic generator that does not transmit packets. Use the silent keyword with auto or desirable mode. If you do not specify silent or non-silent, silent is assumed.
*   The silent mode does not disable the PAgP ability to detect unidirectional links. However, when you configure a channel, non-silent prevents a unidirectional port from even joining the link.
*   A PAgP configuration (the set port channel {desirable | auto} command) is safer than a non-PAgP configuration (the set port channel on command). A PAgP configuration provides protection for unidirectional links and also avoids misconfigurations that can arise when there are ports channeling on one side of the link and not on the other side.

[http://www.cisco.com/en/US/tech/tk389/tk213/technologies_tech_note09186a00800949c2.shtml#silent](http://www.cisco.com/en/US/tech/tk389/tk213/technologies_tech_note09186a00800949c2.shtml#silent)

### Link Aggregation Control Protocol (LACP)

*   Öppen/Industri-standard, 802.3ad
*   Använder channel-modes: active & passive

LACP ger oss även möjligheten att lägga till totalt 16st interface i en "channel-group". Etherchannel kommer fortfarande endast ha 8st aktiva interface, men skulle en av dessa länkar gå ner aktiveras istället någon av de andra åtta som står i standby-läge! Vilken port som blir aktiv/standby bestäms efter "port priority-number" - lägst nummer = högst prio.  Består av 2 byte priority & 2 byte port-name, per default väljs de med lägst interface-nummer. Kan konfigureras via:
```
S3(config)#inte fa0/3
 S3(config-if)#lacp port-priority ?
 <0-65535> Priority value
```
Supportar 100M FastEthernet till 10 Gigabit - dvs vi kan få upp till (8x10Gb)x2=160Gb. S1
```
 S1(config)#int range fa0/3 - 4
 S1(config-if-range)#shut
 S1(config-if-range)#channel-protocol lacp
 S1(config-if-range)#channel-group 1 mode **active**
 S1(config-if-range)#no shut
```
S3
```
 S3(config)#int range fa0/3 - 4
 S3(config-if-range)#shut
 S3(config-if-range)#channel-protocol lacp
 S3(config-if-range)#channel-group 1 mode **passive**
 S3(config-if-range)#no shut
 *Mar 1 00:30:56.205: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/3, changed state to up
 *Mar 1 00:30:57.086: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/4, changed state to up
 *Mar 1 00:30:57.195: %LINK-3-UPDOWN: Interface Port-channel1, changed state to up
 *Mar 1 00:30:58.202: %LINEPROTO-5-UPDOWN: Line protocol on Interface Port-channel1, changed state to up
```
### Statisk
```
 S1(config)#int range fa0/3 - 4
 S1(config-if-range)#shut
 S1(config-if-range)#channel-group 1 mode on
 S1(config-if-range)#no shut
 S3(config)#int range fa0/3 - 4
 S3(config-if-range)#shut
 S3(config-if-range)#channel-group 1 mode on
 S3(config-if-range)#no shut
 S1#sh etherchannel summary
 Flags: D - down P - bundled in port-channel
 I - stand-alone s - suspended
 H - Hot-standby (LACP only)
 R - Layer3 S - Layer2
 U - in use f - failed to allocate aggregator
 M - not in use, minimum links not met
 u - unsuitable for bundling
 w - waiting to be aggregated
 d - default port
 Number of channel-groups in use: 1
 Number of aggregators: 1
 Group Port-channel Protocol Ports
 ------+-------------+-----------+-----------------------------------------------
 1 Po1(SU) **-** Fa0/3(P) Fa0/4(P)
```
Load-Balancing
--------------

Etherchannel ger oss även möjlighet att lastbalansera över länkarna, vilket förövrigt är per frame och inte per bit. Detta skiljer sig dock lite från vad vi är vana vid från andra protokoll som du kommer se längre ner. Vilket interface som skall användas bestäms från resultatet av en hashing-algoritm baserad på den load-balancing method vi använder oss av. 
![etherchannel-lb](/assets/images/2013/09/etherchannel-lb.png) 

Vilken som används per default är beroende av modell på switchen, 3560:n jag använder har stöd för följande metoder:

*   dst-ip Dst IP Addr
*   dst-mac Dst Mac Addr
*   src-dst-ip Src XOR Dst IP Addr
*   src-dst-mac Src XOR Dst Mac Addr
*   src-ip Src IP Addr
*   src-mac Src Mac Addr

Men använder src-mac per default, vi kan verifiera detta med kommandot "**show etherchannel load-balance**".
```
S3#show etherchannel load-balance 
EtherChannel Load-Balancing Configuration:
 **src-mac**
EtherChannel Load-Balancing Addresses Used Per-Protocol:
Non-IP: Source MAC address
 IPv4: Source MAC address
 IPv6: Source MAC address
```
Algoritmen fungerar som så att den tar "3 least significant bits", och använder vi endast en parameter (src-mac) som i detta fall används görs inget mer. 
![netlab](/assets/images/2013/09/netlab.png)
För enkelhetens skull, säg att vi lastbalanserar mellan S1 & S3 över två fysiska länkar, och vi har två datorer med mac-adresserna aaaa.0000.0000 & bbbb.1111.1111 som är kopplade till S1. Vi Konverterar mac-adressen till binärt och tar de sista 3 bitarna: aaaa.0000.0000 -> 3 Least significant bits = 000 bbbb.1111.1111 -> 3 Least significant bits = 001 S1 kommer även ge ett hash-värde för varje interface i channel-groupen (max 8): Interface 1= 0 Interface 2 = 1 ... Interface 8 = 7 Om resultatet av hash-algoritmen blir 0 kommer trafiken skickas på Interface 1 (Fa0/3), om den blir 1 kommer trafiken gå över Interface 2 (Fa0/4). Detta leder som du kanske förstår till en del problem. Skulle trafiken gå via en router kommer source mac alltid vara densamma, således kommer endast ett fysiskt interface att användas oavsett hur många vi har i vår etherchannel! Av denna anledning är det viktigt att tänka till vilken typ av trafik vi har och anpassa algoritmen så vi får en så stor spridning som möjligt. Tyvärr är det endast de riktigt dyra switcharna (Catalyst 6xxx) som klarar att lastbalansera baserat på Lager 4-information. Vi konfigurerar detta med kommandot:

`port-channel load-balance _n_`

Om vi anger två parametrar för hashing-algoritmen, exempelvis src-mac & dest-mac, tas "3 least significant bits" från båda adresserna och switchen utför sedan istället en XOR-operation. ![xor](/assets/images/2013/09/xor.png)

Layer 3
-------

När vi använder en MLS har vi även möjligheten att konfigurera vårat etherchannel till ett Lager 3-interface. Tillvägagångssättet är lite bakvänt men fortfarande väldigt basic. S1
```
S1(config)#int range fa0/3 - 4
 S1(config-if-range)#shut
 S1(config-if-range)#exit
S1(config)#interface port-channel 5
 S1(config-if)#no switchport
 S1(config-if)#ip add 10.0.0.1 255.255.255.0
S1(config-if)#int range fa0/3 - 4
 S1(config-if-range)#no switchport
 S1(config-if-range)#channel-group 5 mode desirable
 S1(config-if-range)#channel-protocol pagp
 S1(config-if-range)#no shut
```
S3
```
S3(config)#int range fa0/3 - 4
 S3(config-if-range)#shut
 S3(config-if-range)#exit
S3(config)#interface port-channel 5
 S3(config-if)#no switchport
 S3(config-if)#ip add 10.0.0.2 255.255.255.0
S3(config-if)#int range fa0/3 - 4
 S3(config-if-range)#no switchport
 S3(config-if-range)#channel-group 5 mode auto
 S3(config-if-range)#channel-protocol pagp
 S3(config-if-range)#no shut
```
Verifiering
```
S3(config-if-range)#do ping 10.0.0.1
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 10.0.0.1, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 1/4/9 ms
 ```
 