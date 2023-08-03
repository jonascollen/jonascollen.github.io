---
title: IPv6 - Link-local Multicast
date: 2013-08-11 00:41
comments: true
categories: [IPv6, RIPng]
---
Då broadcast inte längre används i IPv6 som vi nämnt i tidigare [inlägg](http://roadtoccie.se/2013/08/10/ipv6-the-basics/ "IPv6 – The Basics") kan det väl vara lämpligt med en introduktion till multicast. https://www.youtube.com/watch?v=lGjJOhv6sck Rekommenderar ovanstående video till att börja med, förklarar konceptet väldigt bra. Jag har använt mig av samma topologi för denna labb. 

![ipv6-simple-topology](/assets/images/2013/08/ipv6-simple-topology.png) 
Multicast ger oss möjligheten att skicka ett och samma paket till flera mottagare, men tas endast emot av enheter som är intresserade av paketet (genom att de gått med i vår multicast-grupp). Betydligt mer effektivt än broadcast där alla enheter måste processa paketet oavsett om de är intresserade av informationen eller ej. Multicast adresser börjar som bekant alltid med FF00::/8, men det finns flera olika typer av multicast-trafik/grupper. För Multicast-adresser som börjar med FF02 som vi ska kolla på idag betyder att de är **Link-local**, dvs de forwardas EJ utanför vårt L2-segment.

*   FF02::1 - All-nodes address, används för att nå alla enheter på L2-segmentet
*   FF02::2 - All-routers address, används för att nå alla routrar på L2-segmentet
*   FF02::5 - All-OSPF routers, används för att nå alla OSPF-routrar L2-segmentet (samma funktion som IPv4s 224.0.0.5)
*   FF02::6 - All-OSPF DR/BDR routers, används för att nå alla OSFP DR/BDR-routrar på L2-segmentet (samma funktion som IPv4s 224.0.0.6)
*   FF02::9 - All-RIPng routers, används för att nå alla RIP-routrar på L2-segmentet
*   FF02::A - All-EIGRP routers, används för att nå alla EIGRP-routrar på L2-segmentet
*   FF02::1::FFxx:xxxx - Solicited-node address, resolvar en Link-local IPv6-adress till Link-Layer address (IPv6s version av ARP!)

Det sistnämnda kräver väl lite mer förklaring.. En IPv6 Unicast-adress är antingen:

*   Global (2000::/3 - 3ffff::/3)
*   Link-Local  (FE80::/8)

För varje IPv6 Unicast-adress vi har konfigurerad så måste enhet även gå med i den associerade Solicited-node multicast-gruppen för den specifika adressen. Multicast-gruppen Solicited Node börjar alltid med adressen FF02::1:FFxx:xxxx, där vi för de sista 24 bitarna använder motsvarande sista 24-bitar från vår Global eller Link-Local adress. Detta ger oss möjlighet att kommunicera med enheter trots att vi inte ännu kan deras mac-adress, det är med andra ord IPv6 variant av ARP-lookup (broadcast) som vi använder i IPv4. Funktionen kallas NDP (Neighbor Discovery Protocol) och gör en hel del annat skoj förutom IPv6 -> Mac-adress resolving, men det får bli en egen post vid ett senare tillfälle. Om vi tar följande exempel från en Global adress: 2001:db8:6783:1111::1/64 För att göra det enklare skriver vi ut alla  "leading 0's" vi använt. 2001:0db8:6783:1111:0000:0000:0000:0001/64, och så tar vi de sista 24-bitarna: 2001:0db8:6783:1111:0000:0000:00**00:0001**/64 00:0001 Solicited Node multicast-adressen blir då: **FF02::1:FF00:0001** Vi tar och testar detta i R1;

```
conf t
interface fa0/0
no ipv6 add
ipv6 enable
R1#show ipv6 int fa0/0
FastEthernet0/0 is up, line protocol is up
 IPv6 is enabled, link-local address is **FE80::C000:22FF:FE40:0** 
 No Virtual link-local address(es):
 No global unicast address is configured
 **Joined group address(es):**
 FF02::1
 FF02::2
 **FF02::1:FF40:0**
```
Vi kan se att routern automatiskt gått med i tre multicast-grupper.

*   FF02::1 - All-nodes address
*   FF02::1:FF40:0 - Solicited-node address

Om vi även aktiverar IPv6-routing genom kommandot ipv6 unicast-routing kan vi se att routern går med i ytterligare en grupp:
```
**Joined group address(es):**
 FF02::1
 **FF02::2**
 FF02::1:FF40:0
```
*   FF02::2 - All-routers address

Routern vet nu att den är en router(:D), och måste således gå med i All-routers multicast-gruppen. Låt oss aktivera EIGRP, OSPF & RIP på routern och se vad som händer.
```
conf t
 int fa0/0
 ipv6 rip RIP-LAB enable
 ipv6 ospf 1 area 0
 ipv6 eigrp 10
 ipv6 router ospf 1
 router-id 1.1.1.1
 ipv6 router eigrp
 router-id 1.1.1.1
```
Detta ger oss följande resultat (vi _behöver lägga till router-id för EIGRP & OSPF då vi inte har någon IPv6-adress konfigurerad på routern)_:
```
R1#sh ipv6 int fa0/0
FastEthernet0/0 is up, line protocol is up
 IPv6 is enabled, link-local address is FE80::C000:22FF:FE40:0 
 No Virtual link-local address(es):
 No global unicast address is configured
 Joined group address(es):
 FF02::1 <- All-nodes
 FF02::2 <- All-routers
 FF02::5 <- All OSPF-routers
 FF02::6 <- All OSPF BD/BDR-routers
 FF02::9 <- All RIPng-routers
 FF02::A <- All EIGRP-router s
 FF02::1:FF40:0 <- Solicited node-address
```
Snyggt! Hur går det då till lite mer specifikt när en host först bootar upp och endast känner till sin default-gateways L3 IPv6-adress (R1), exempelvis 2001:db8:6783:1::1? Hosten kommer skicka en "ICMPv6 Neighbor-Solicitation" förfrågan till Solicited-Node Multicast-adressen på L2-segmentet för just 2001:db8:6783:1::1/64, vilket blir? FF02::1:FF00:1 R1 som är den enda som lyssnar till denna multicast-grupp kommer svara Host med ett ICMPv6 Neighbor-Advertisement innehållandes dess MAC-adress. Om du kommer ihåg posten från tidigare idag så såg vi följande wireshark-dump på just ett "Neighbor Solicitation"-paket: 
[![IPv6-neighborsolicitation](/assets/images/2013/08/ipv6-neighborsolicitation.png)](/assets/images/2013/08/ipv6-neighborsolicitation.png) 

Paketet är adresserat till multicast-gruppen FF02::1:FFCD:1111 för att kontrollera om det finns någon enhet med adressen "FE80::200:ABFF:FECD:1111)". Varje L3 multicast-adress har förövrigt en motsvarande L2-adress. L2-adressen börjar alltid med 3333 (16bitar), och tar sedan de sista 32 bitarna från vår multicast-grupp. Multicast-gruppen FF02::1 blir då: 3333:0000:0001 -> 33:33:00:00:00:01. FF02::5 blir: **3333:0000:0005 -> 33:33:00:00:00:05** Vi kan verifiera detta genom att kolla hur ett OSPF-hello paket ser ut: 
[![ipv6-ospfhello](/assets/images/2013/08/ipv6-ospfhello.png)](/assets/images/2013/08/ipv6-ospfhello.png)

Destinationsadresserna är:

*   L2 - 33:33:00:00:00:05
*   L3 - FF02::5

Det blir lite krångligare när vi ska konvertera Solicited-node adressen till L2 då vi endast använt 24-bitar till att skapa vår L3-adress, men vi behöver som bekant 32-bitar. Vi löser det på ungefär samma sätt som EUI-64 adresser skapas, genom att lägga till FF efter de första 16 bitarna. L2-adressen för FF02::1:FF00:0001 blir då: **3333:FF00:0001 -> 33:33:FF:00:01**
