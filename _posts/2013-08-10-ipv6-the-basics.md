---
title: IPv6 - The Basics
date: 2013-08-10 16:02
comments: true
categories: [IPv6]
---
*   IPv6 adresser är 128 bitar långa
*   Separeras med en kolon för varje 16:e bits (16 bits : 16 bits : etc..)
*   Subnätmask specificeras med hjälp av /64 /32 etc
*   Skrivs i Hex istället för binärt (0-9, A-F)
*   Broadcast existerar inte i IPv6 (ersatt av multicast)
*   Stödjer autokonfigurering av adresser (stateless autoconfiguration)
*   Använder ej fragmentering
*   IPv6-headern använder ej checksum utan förlitar sig på underliggande protokoll
*   IPv6-headern är förenklad i jämförelse med sin lillebror IPv4, vilket ger snabbare hantering
*   Ett interface kan ha flera olika IPv6-adresser konfigurerat samtidigt (Global & Link-local osv)

IPv6-adresser
-------------

En IPv6-adress skrivs enligt följande: 2001:0db8:0000:0000:abc0:0001:ffff:2025/64 Varje "grupp" representerar 16 bitar, 16 x 8 = 128 bitar. Vi kan dock förenkla skrivandet genom att summera grupper bestående av endast nollor genom att istället skriva "::", detta går endast att använda en gång dock! 
[![Ipv6_address_leading_zeros.svg](/assets/images/2013/08/ipv6_address_leading_zeros-svg.png)](/assets/images/2013/08/ipv6_address_leading_zeros-svg.png) 

**2001:0db8::abc0:0001:ffff:2025/64** Vi kan även utesluta alla inledande 0:or, detta ger oss följande: **2001:db8::abc0:1:ffff:2025/64** Om vi vill konvertera adressen till binärt så använder vi följande tabell. Kom ihåg att varje "grupp" i adressen består av 16 bitar (vilket är binärt 0000 0000 0000 0000). Varje siffra/bokstav motsvarar med andra ord fyra bitar binärt. 

[![binary-to-hex](/assets/images/2013/08/binary-to-hex.jpg)](/assets/images/2013/08/binary-to-hex.jpg) 
2001 = 0010 0000 0000 0001 0db8 = 0000 1101 1011 1000 0000 = 0000 0000 0000 0000 0000 = 0000 0000 0000 0000 abc0 = 1010 1011 1100 0000 0001 = 0000 0000 0000 0001 ffff = 1111 1111 1111 1111 2025 = 0010 0000 0010 0101

IPv6 Adress-typer
-----------------

*   **Global (Unicast/Anycast)**

Adressen börjar alltid binärt med 001..., dvs i hex: 2000::/3 - 3fff::/3. Är alltid "Globally available"/Publik adress som är nåbar över internet förutsatt att routing finns givetvis. [RFC 6177](http://tools.ietf.org/html/rfc6177) rekommenderar att ISPs och dylikt aldrig bör ge sina kunder nät mindre än /64.

*   **Multicast**

Börjar alltid binärt med 1111 1111, dvs i hex: FF00::/8. Används betydligt mer i IPv6 då broadcast inte längre finns kvar. Är alltid destinations-adressen, ej source.

*   **Loopback**

Till skillnad från IPv4 så har endast en adress reserverats, ::1/128.

*   **Link-local**

Börjar alltid binärt med 1111 1110 10.., dvs i hex: FE80::/8. Konfigureras automatiskt när vi aktiverar IPv6 på ett interface och tillåter oss att kommunicera över ett L2-segment utan ytterligare konfig. Hostadressen (EUI-64) genereras från enhetens MAC-adress. Men då en MAC-adress endast är 48-bitar behövdes ytterligare 16 bitar läggas till. De löste detta genom att dela vår mac-adress på mitten och lägga till FF:FE. En mac-adress på ex. 0000.abcd.1111 blir med andra ord: 0000.abFF:FEcd.1111 - vilket ger oss 64 bitar. För att krångla till det ytterligare så konverteras den inledande 7:e biten för att identifiera att detta är en genererad adress. Om vi först skriver om vår inledande grupp till binärt så ser vi vad resultat blir: 0000 0000 0000 0000, inverterar vi den 7:e biten blir adressen istället: 0000 0010 0000 0000, tillbaka till hex får vi: 0200. Den nya EUI64-adressen blir med andra ord: 0200.abFF:FEcd.1111, kombinerat med den inledande nätadressen FE80:: får vi, FE80::200:abFF:FEcd.1111/64. Vi kan verifiera detta på en router enligt följande:
```
R1(config)#inte fa0/0
R1(config-if)#no shut
R1(config-if)#mac-address 0000.abcd.1111
R1(config-if)#ipv6 enable
R1#sh ipv6 int fa0/0
FastEthernet0/0 is up, line protocol is up
 IPv6 is enabled, link-local address is FE80::200:ABFF:FECD:1111
```
Stiligt! :) Innan routern börjar använda link-local adressen skickas dock först ett ICMPv6-paket kallat "Neighbor solicitation" med source-adressen "::", för att vara säker på att adressen inte redan används. 
[![IPv6-neighborsolicitation](/assets/images/2013/08/ipv6-neighborsolicitation.png)](/assets/images/2013/08/ipv6-neighborsolicitation.png) 

Får den inget svar skickas istället ett "neighbor advertisement" och informerar resterande enheter på L2-segmentet om sin nya adress. 
[![IPv6-neighboradvertisement](/assets/images/2013/08/ipv6-neighboradvertisement.png)](/assets/images/2013/08/ipv6-neighboradvertisement.png) 

Om vi testar i följande topologi att endast aktivera IPv6 på interfacet och pinga PCs link-local FE80::C005:22FF:FE40:0: 
[![IPv6](/assets/images/2013/08/ipv6.png)](/assets/images/2013/08/ipv6.png)

```
R1#ping ipv6 FE80::C005:22FF:FE40:0
Output Interface: FastEthernet0/0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FE80::C005:22FF:FE40:0, timeout is 2 seconds:
Packet sent with a source address of FE80::200:ABFF:FECD:1111
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 16/24/32 ms
```
