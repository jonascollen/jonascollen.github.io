---
title: IPv6 - Static, 6to4 & ISATAP Tunnels
date: 2013-08-20 12:48
comments: true
categories: [IPv6]
tags: [6to4,ISATAP,GRE]
---
Har haft lite för mycket att göra senaste dagarna för att skriva en post om detta, nöjer mig istället med en video som förklarar konceptet väldigt bra och enklare konfig-exempel: [https://www.youtube.com/watch?v=JY7INWIcqvk](https://www.youtube.com/watch?v=JY7INWIcqvk) Topologi: ![ipv6-tunnels](/assets/images/2013/08/ipv6-tunnels1.png)

Static MCT
----------

*   Point-to-point
*   RFC 4213
*   "Manually-Configured-Tunnels"
*   20 bytes overhead
*   Genererar Link-local adress för tunneln med ett FE80::/96 prefix, varav sista 32 bitarna tas från IPv4 source-adressen

Exempel: IPv4 10.9.9.1 -> Hex 0A09:0901 -> IPv6 Link-local FE80::A09:901

### R5
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2001:DB8:CC1E:100::1/64
 ipv6 eigrp 1
interface Tunnel0
 ipv6 address 2001:DB8:CC1E:500::1/64
 ipv6 eigrp 1
 tunnel source Loopback0
 tunnel destination 6.6.6.6
 tunnel mode ipv6ip
ipv6 router eigrp 1
 no shutdown
```
### R6
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2001:DB8:CC1E:200::1/64
 ipv6 eigrp 1
interface Tunnel0
 ipv6 address 2001:DB8:CC1E:500::2/64
 ipv6 eigrp 1
 tunnel source Loopback0
 tunnel destination 5.5.5.5
 tunnel mode ipv6ip
ipv6 router eigrp 1
 no shutdown
```
### Verifiering
```
R6#sh ipv6 interface tunnel0
Tunnel0 is up, line protocol is up
 IPv6 is enabled, link-local address is **FE80::606:606**
IPv4 6.6.6.6 -> Hex 0606:0606 -> FE80::606:606

R6#sh int tu0
Tunnel0 is up, line protocol is up
Hardware is Tunnel
MTU 1514 bytes, BW 9 Kbit/sec, DLY 500000 usec,
reliability 255/255, txload 1/255, rxload 1/255
Encapsulation TUNNEL, loopback not set
Keepalive not set
Tunnel source 6.6.6.6 (Loopback0), destination 5.5.5.5
**Tunnel protocol/transport IPv6/IP**
  Tunnel TTL 255
  Fast tunneling enabled
  Tunnel transmit bandwidth 8000 (kbps)
  Tunnel receive bandwidth 8000 (kbps)
  Last input 00:00:03, output 00:00:02, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue: 0/0 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     280 packets input, 33744 bytes, 0 no buffer
     Received 0 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
     273 packets output, 27464 bytes, 0 underruns
     0 output errors, 0 collisions, 0 interface resets
     0 unknown protocol drops
     0 output buffer failures, 0 output buffers swapped out

R5#ping ipv6 2001:db8:cc1e:200::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:CC1E:200::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 140/499/780 ms
R5#sh ipv6 eigrp neighbor
IPv6-EIGRP neighbors for process 1
H Address Interface Hold Uptime SRTT RTO Q Seq
 (sec) (ms) Cnt Num
0 Link-local address: Tu0 13 00:05:16 417 5000 0 3
 FE80::606:606
```
Static GRE
----------

*   Point-to-Point
*   RFC 2784
*   Lägger till ytterligare 4 bytes till overhead (24b, Max MTU 1476)
*   Tillåter att vi använder ett flertal protokoll över tunneln (inte bara IPv6)
*   Default-encapsulering i Ciscos IOS
*   Genererar en link-local adress på samma sätt som ett serial-interface (Tar Eth-if + EUI-64)
```
R5#sh ipv6 int tunnel0
Tunnel0 is up, line protocol is up
 IPv6 is enabled, link-local address is FE80::C004:BFF:FEB4:0
R5#sh int fa0/0
FastEthernet0/0 is up, line protocol is up 
 Hardware is Gt96k FE, address is **c204.0bb4.0000** (bia c204.0bb4.0000)
```
Det enda som skiljer sig från att konfigurera en MCT-tunnel är kommandot:
```
no tunnel mode ipv6ip
**tunnel mode gre ip**
```
I övrigt är all konfig identisk.

6to4
----

*   Point-To-Multipoint
*   RFC 3056
*   Stödjer ej IGPs utan kräver användandet av BGP eller Statiska routes
*   2002::/16 reserverat för intern användning
*   Genererar Link-local adress för tunneln med ett FE80::/96 prefix, varav sista 32 bitarna tas från IPv4 source-adressen på samma sätt som vid MCT
*   Rekommenderar användandet av /48-prefix per site, /64-prefix per subnät
*   Använd IPv4-adressen för 2 & 3 "quartet" / 17-48 bits;

Exempel: 2002 (prefix) : AABB : CCDD (ipv4-add) : subnet :: /64 IPv4 10.9.9.1 ->Hex 0A09:0901 6to4 adress - > 2002:0A09:0901::/48 Site-links -> 2002:0A09:0901:**0**::/64 Site 1 -> 2002:0A09:0901:**1**::/64 Site 2 -> 2002:0A09:0901:**2**::/64 Site 3 -> 2002:0A09:0901:**3**::/64 
[!][ipv6-6to4tunnel](/assets/images/2013/08/ipv6-6to4tunnel.png)
IPv4 är redan konfigurerat enligt ovanstående bild med full BGP-peering mellan samtliga AS. R5 annonserar 5.5.5.0/24 prefixet, R6 annonserar 6.6.6.0/24 och R7 annonserar 7.7.7.0/24.
```
R5#sh ip bgp
BGP table version is 9, local router ID is 200.0.3.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
*> 5.5.5.0/24 0.0.0.0 0 32768 i
*> 6.6.6.0/24 172.16.51.1 0 500 200 i
*> 7.7.7.0/24 172.16.51.1 0 500 50 i
```
### R5

IPv4 5.5.5.5 -> Hex 0505:0505 -> 2002:505:505::/48
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2002:505:505::5/64

interface Tunnel0
 ipv6 unnumbered Loopback0
 tunnel source Loopback0
 tunnel mode ipv6ip 6to4
ipv6 route 2002::/16 Tunnel0
```
### R6

IPv4 6.6.6.6-> Hex 0606:0606 -> 2002:606:606::/48
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2002:606:606::6/64

interface Tunnel0
 ipv6 unnumbered Loopback0
 tunnel source Loopback0
 tunnel mode ipv6ip 6to4
ipv6 route 2002::/16 Tunnel0
```
### R7

IPv4 7.7.7.7 -> Hex 0707:0707 -> 2002:707:707::/48
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2002:707:707::7/64

interface Tunnel0
 ipv6 unnumbered Loopback0
 tunnel source Loopback0
 tunnel mode ipv6ip 6to4
ipv6 route 2002::/16 Tunnel0
```
### Verifiering
```
R5(tcl)#foreach ip { 
+>(tcl)#2002:505:505::5 
+>(tcl)#2002:606:606::6 
+>(tcl)#2002:707:707::7 
+>(tcl)#} { ping $ip source lo0 repeat 2 timeout 1 }
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2002:505:505::5, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 0/0/0 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2002:606:606::6, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 124/126/128 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2002:707:707::7, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 124/216/308 ms
R5#sh int tunnel0
Tunnel0 is up, line protocol is up 
 Hardware is Tunnel
 MTU 1514 bytes, BW 9 Kbit/sec, DLY 500000 usec, 
 reliability 255/255, txload 1/255, rxload 1/255
 Encapsulation TUNNEL, loopback not set
 Keepalive not set
 Tunnel source 5.5.5.5 (Loopback0), destination UNKNOWN
 **Tunnel protocol/transport IPv6 6to4**
 Tunnel TTL 255
 Fast tunneling enabled
 Tunnel transmit bandwidth 8000 (kbps)
 Tunnel receive bandwidth 8000 (kbps)
 Last input 00:00:38, output 00:00:38, output hang never
 Last clearing of "show interface" counters never
 Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
 Queueing strategy: fifo
 Output queue: 0/0 (size/max)
 5 minute input rate 0 bits/sec, 0 packets/sec
 5 minute output rate 0 bits/sec, 0 packets/sec
 19 packets input, 2660 bytes, 0 no buffer
 Received 0 broadcasts, 0 runts, 0 giants, 0 throttles
 0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
 24 packets output, 2784 bytes, 0 underruns
 0 output errors, 0 collisions, 0 interface resets
 0 unknown protocol drops
 0 output buffer failures, 0 output buffers swapped out
R5#sh ipv6 int tunnel0
Tunnel0 is up, line protocol is up
 IPv6 is enabled, link-local address is FE80::505:505
```
Det finns dock även möjlighet att använda "vanliga" global unicast ipv6-adresser för våra siter. Vi behöver då endast använda 2002::/16 för Tunnel-interfacen och sedan skapa en statisk route för varje nät, ex:
```
!till R6
ipv6 route 2001:db8:cc1e:600::/64 2002:606:606::
!till R7
ipv6 route 2001:db8:cc1e:700::/64 2002:707:707::
```
ISATAP
------

ISATAP har väldigt mycket gemensamt med 6to4, det som skiljer är:

*   Använder inget reserverat prefix
*   Använder vanligtvis ett prefix för samtliga tunnel-interface/samma subnät
*   Kräver att vi sätter upp en statisk route för varje enskild site
*   Vi kan antingen använda oss av EUI-64 (modifierad) eller manuellt konfigurera IPv6-adresseringen enligt samma uträkningssätt för våra tunnelinterface:

Ta det /64-prefix du vill använda, ex: 2001:db8:cc1e:100::/64 och lägg till 0000:5EFE till "quartets" 5 & 6 -> 2001:db8:cc1e:100:0000:5EFE;;, Ta sedan IPv4-adressen från tunnelns IPv4-source adress och konvertera till hex (ex. 10.9.9.1) -> 0A09:0901 -> ISATAP adress; **2001:db8:cc1e:100:0:5EFE:A09:901/64 - klart!** Om vi istället konfigurerar "**ipv6 add 2001:db8:cc1e::/64 eui-64**" på Tunnel-interfacet så gör routern denna uträkning åt oss. :) Låt oss använda samma topologi men istället ta IPv6-prefixet 2001:db8:cc1e:500::/64 som länknät för våra tunnel-interface. 
![ipv6-6to4tunnel](/assets/images/2013/08/ipv6-6to4tunnel.png)

### R5
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2002:505:505::5/64

interface Tunnel0
 ipv6 add 2001:db8:cc1e:500::/64 eui-64
 tunnel source Loopback0
 tunnel mode ipv6ip isatap

ipv6 route 2002:606:606::/48 **2001:db8:cc1e:500:0:5EFE:606:606**
ipv6 route 2002:707:707::/48 **2001:db8:cc1e:500:0:5EFE:707:707**
```
Vi kan verifiera att routern använt samma uträkningsformel för adressen på tunnel-interfacet:
```
R5#sh ipv6 int tunnel0 | sec Global
 Global unicast address(es):
 2001:DB8:CC1E:500:**0:5EFE:505:505**, subnet is 2001:DB8:CC1E:500::/64 [EUI]
```
Som synes måste vi även konfigurera next-hop för våra statiska routes, och kan inte bara peka på vårat tunnel-interface som vid 6to4. R6 har som bekant IPv4 6.6.6.6 -> Hex 0606:0606 -> ISATAP EUI-64 2001:db8:cc1e:0:5EFE:606:606,samma sak gäller för R7 (7.7.7.7).

### R6
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2002:606:606::6/64

interface Tunnel0
 ipv6 add 2001:db8:cc1e:500::/64 eui-64
 tunnel source Loopback0
 tunnel mode ipv6ip isatap

ipv6 route 2002:505:505::/48 2001:db8:cc1e:500:0:5EFE:505:505
ipv6 route 2002:707:707::/48 2001:db8:cc1e:500:0:5EFE:707:707
```
### R7
```
ipv6 unicast-routing
interface lo0
 ipv6 address 2002:707:707::7/64

interface Tunnel0
 ipv6 add 2001:db8:cc1e:500::/64 eui-64
 tunnel source Loopback0
 tunnel mode ipv6ip isatap

ipv6 route 2002:505:505::/48 2001:db8:cc1e:500:0:5EFE:505:505
ipv6 route 2002:606:606::/48 2001:db8:cc1e:500:0:5EFE:606:606
```
### Verifiering
```
R5(tcl)#foreach ip { 
+>(tcl)#2001:db8:cc1e:500:0:5EFE:505:505
+>(tcl)#2001:db8:cc1e:500:0:5EFE:606:606
+>(tcl)#2001:db8:cc1e:500:0:5EFE:707:707
+>(tcl)#2002:505:505::5
+>(tcl)#2002:606:606::6
+>(tcl)#2002:707:707::7
+>(tcl)#} { ping $ip source lo0 rep 2 timeout 1 }
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2001:DB8:CC1E:500:0:5EFE:505:505, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 0/2/4 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2001:DB8:CC1E:500:0:5EFE:606:606, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 96/104/112 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2001:DB8:CC1E:500:0:5EFE:707:707, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 72/88/104 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2002:505:505::5, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 0/0/0 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2002:606:606::6, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 96/114/132 ms
Type escape sequence to abort.
Sending 2, 100-byte ICMP Echos to 2002:707:707::7, timeout is 1 seconds:
Packet sent with a source address of 2002:505:505::5
!!
Success rate is 100 percent (2/2), round-trip min/avg/max = 80/90/100 ms
```
Härligt! Måste erkänna att detta var riktigt jäkla knepigt att sätta sig in i, speciellt hur adresseringen av siterna ska gå till. Rekommenderar att labba upp detta många gånger för att få det att verkligen fastna! Funderar faktiskt på att åka till Stockholm och skriva CCNP Route nästa vecka. Men det innebär då att det blir till att nöta repetering av endast CCNP-materialet nu några dagar framöver, så de planerade posterna om IPv6 Anycast/Stateless autoconfig m.m. får vänta ett tag.
