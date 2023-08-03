---
title: Routing över NBMA - DMVPN & NHRP
date: 2018-04-05 19:24
author: Jonas Collén
comments: true
categories: [DMVPN, NHRP]
---
![](/assets/images/2018/04/Topologi.png)

En enklare labb på routing över DMVPN som jag inte hunnit läsa/labba så mycket på ännu, verkade dock rätt intressant för att få en bättre förståelse över hur NHRP används för att sätta upp tunnlar mellan spokes. 

*   Konfigurera en statisk default-route i R1 & R2 genom DMVPN-molnet till R5, routen får endast vara giltig när Tunnel-interfacet är i UP-state
*   Konfigurera R5 med statiska routes till R1 & R2's loopbacks genom DMVPN-molnet
*   Tunnel-interfacen för respektive router är: R1 155.1.0.1, R2 155.1.0.2, R5 155.1.0.5

Enklaste sättet att lösa första kravet är att ange utgående interface tillsammans med nexthop till R5, då blir routen endast giltig när nexthop är nåbar över specificerade interfacet. Anger vi bara nexthop kommer routen fortfarande vara giltig så länge det finns en alternativ väg (och då matchar vi inte längre kravet i labben).

```
R1(config)#ip route 0.0.0.0 0.0.0.0 Tunnel0 155.1.0.5

R2(config)#ip route 0.0.0.0 0.0.0.0 Tunnel0 155.1.0.5

R5(config)#ip route 150.1.1.1 255.255.255.255 Tunnel0 155.1.0.1
R5(config)#ip route 150.1.2.2 255.255.255.255 Tunnel0 155.1.0.2

Vi skulle även kunna krångla till det lite och använda trackers på R1 & R2 istället i stil med detta:

R1(config)#track 1 interface Tunnel0 line-protocol
R1(config)#ip route 0.0.0.0 0.0.0.0 155.1.0.5 track 1

R2(config)#track 1 interface Tunnel0 line-protocol
R2(config)#ip route 0.0.0.0 0.0.0.0 155.1.0.5 track 1
```

Hur som.. Alla routrar bör nu kunna pinga varandra utan problem.

```
R1#ping 150.1.5.5
 Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 150.1.5.5, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/4 ms

R1#ping 150.1.2.2
 Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 150.1.2.2, timeout is 2 seconds:
 .!!!!
 Success rate is 80 percent (4/5), round-trip min/avg/max = 4/4/5 ms
```

Observera att vi tappar första paketet till R2, detta beror på att R1 måste skicka ett NHRP Resolution Request till R5 där den efterfrågar den publika ip-adressen (NBMA address) för att komma till R2/150.1.2.2. R5 svarar med R2's publika address 169.254.100.2, R1 handskakar därefter upp en dynamisk tunnel mellan den & R2.  Vi kan verifiera detta med:

```
R2#
 interface Tunnel0
 ip address 155.1.0.2 255.255.255.0
 ...
  **tunnel source GigabitEthernet1.100**
 ...

interface GigabitEthernet1.100
 encapsulation dot1Q 100
 ip address 169.254.100.2 255.255.255.0

R1#sh ip nhrp
 155.1.0.1/32 via 155.1.0.1
 Tunnel0 created 00:00:03, expire 00:04:56
 Type: dynamic, Flags: router unique local
 NBMA address: 169.254.100.1
 (no-socket)
 **155.1.0.2/32 via 155.1.0.2**
 ** Tunnel0 created 00:00:03, expire 00:04:56**
 **Type: dynamic, Flags: router implicit used nhop** 
 ** NBMA address: 169.254.100.2**
 155.1.0.5/32 via 155.1.0.5
 Tunnel0 created 01:30:55, never expire
 Type: static, Flags: used
 NBMA address: 169.254.100.5
```

Detta fungerar lika bra även om vi endast anger DMVPN-molnet som utgående IF i R1 & R2:

```
R2(config)#no ip route 0.0.0.0 0.0.0.0 Tunnel0 155.1.0.5
R2(config)#ip route 0.0.0.0 0.0.0.0 Tunnel0

R1(config)#no ip route 0.0.0.0 0.0.0.0 Tunnel0 155.1.0.5
R1(config)#ip route 0.0.0.0 0.0.0.0 Tunnel0

R1#sh ip nhrp
 155.1.0.5/32 via 155.1.0.5
 Tunnel0 created 01:38:25, never expire
 Type: static, Flags: used
 NBMA address: 169.254.100.5

R1#ping 150.1.2.2
 Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 150.1.2.2, timeout is 2 seconds:
 .!!!!
 Success rate is 80 percent (4/5), round-trip min/avg/max = 4/4/5 ms
```

Vad händer dock om vi gör samma sak i vår hub/R5?

```
R5(config)#no ip route 150.1.1.1 255.255.255.255 Tunnel0 155.1.0.1
R5(config)#no ip route 150.1.2.2 255.255.255.255 Tunnel0 155.1.0.2
R5(config)#ip route 150.1.2.2 255.255.255.255 Tunnel0
R5(config)#ip route 150.1.1.1 255.255.255.255 Tunnel0

R1#ping 150.1.2.2
 Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 150.1.2.2, timeout is 2 seconds:
 .....
 Success rate is 0 percent (0/5)
```
Detta trots att våra tunnlar fortfarande är uppe!

```
R1#sh ip nhrp
 150.1.2.2/32 via 155.1.0.2
 Tunnel0 created 00:01:11, expire 00:03:49
 Type: dynamic, Flags: router
 NBMA address: 169.254.100.2
 **155.1.0.2/32 via 155.1.0.2**
 ** Tunnel0 created 00:01:10, expire 00:03:49**
 **Type: dynamic, Flags: router nhop** 
 ** NBMA address: 169.254.100.2**
 155.1.0.5/32 via 155.1.0.5
 Tunnel0 created 01:39:55, never expire
 Type: static, Flags: used
 NBMA address: 169.254.100.5
```
En debug ger lite bättre inblick vad det är som går fel:

```
R5#debug nhrp
 NHRP protocol debugging is on
 R5#debug ip pack
 R5#debug ip packet detail
 IP packet debugging is on (detailed)
R5#ping 150.1.1.1 rep 1
 Type escape sequence to abort.
 Sending 1, 100-byte ICMP Echos to 150.1.1.1, timeout is 2 seconds:

IP: s=155.1.0.5 (local), d=150.1.1.1, len 100, local feature
 ICMP type=8, code=0, feature skipped, Auth Proxy(16), rtype 0, forus FALSE, sendself FALSE, mtu 0, fwdchk FALSE
 FIBipv4-packet-proc: route packet from (local) src 155.1.0.5 dst 150.1.1.1
 FIBfwd-proc: packet routed by adj to Tunnel0 150.1.1.1
 FIBipv4-packet-proc: packet routing succeeded
 IP: tableid=0, s=155.1.0.5 (local), d=150.1.1.1 (Tunnel0), routed via FIB
 IP: s=155.1.0.5 (local), d=150.1.1.1 (Tunnel0), len 100, sending
 ICMP type=8, code=0
 IP: s=155.1.0.5 (local), d=150.1.1.1 (Tunnel0), len 100, output feature
 ICMP type=8, code=0, feature skipped, TCP Adjust MSS(58), rtype 1, forus FALSE, sendself FALSE, mtu 0, fwdchk FALSE
 **NHRP: NHRP could not map 150.1.1.1 to NBMA, cache entry not found**
 NHRP: MACADDR: if\_in null netid-in 0 if\_out Tunnel0 netid-out 1
 NHRP: Checking for delayed event NULL/150.1.1.1 on list (Tunnel0 vrf: global(0x0))
 NHRP: No delayed event node found.
 NHRP: No cache for forwarding(0)
 NHRP: Node found.
 Success rate is 0 percent (0/1)
```

Det är hubrouterns uppgift att hålla kolla på mappningarna mellan intern/publik nexthop och den har ingen annan att fråga än sig själv. Därför ska man alltid ange nexthop-adress när man konfigurerar rötter i hubroutern/R5. Vi kan dock lösa detta genom en workaround och göra statiska mappningar för loopback-näten i R5.

```
R5(config)#int tu0
R5(config-if)#ip nhrp map 150.1.1.1 169.254.100.1
R5(config-if)#ip nhrp map 150.1.2.2 169.254.100.2

R5#sh ip nhrp
 **150.1.1.1/32 via 150.1.1.1**
 Tunnel0 created 00:00:12, never expire
 Type: **static**, Flags:
 NBMA address: 169.254.100.1
 **150.1.2.2/32 via 150.1.2.2**
 Tunnel0 created 00:00:04, never expire
 Type: **static**, Flags:
 NBMA address: 169.254.100.2
 155.1.0.1/32 via 155.1.0.1
 Tunnel0 created 01:50:06, expire 00:04:24
 Type: dynamic, Flags: unique registered used nhop
 NBMA address: 169.254.100.1
 155.1.0.2/32 via 155.1.0.2
 Tunnel0 created 01:50:01, expire 00:04:28
 Type: dynamic, Flags: unique registered used nhop
 NBMA address: 169.254.100.2

R1#ping 150.1.2.2
 Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 150.1.2.2, timeout is 2 seconds:
 .!!!!
 Success rate is 80 percent (4/5), round-trip min/avg/max = 4/4/5 ms

R5#sh ip cef 150.1.1.1 internal
150.1.1.1/32, epoch 2, RIB\[S\], refcnt 6, per-destination sharing
 sources: RIB 
 feature space:
 IPRM: 0x00048000
 Broker: linked, distributed at 4th priority
 ifnums:
 **Tunnel0(16): 155.1.0.1**
 path list 7F324BEC8258, 3 locks, per-destination, flags 0x49 \[shble, rif, hwcn\]
 path 7F324C152DB0, share 1/1, type attached nexthop, for IPv4
 **nexthop 155.1.0.1 Tunnel0, IP midchain out of Tunnel0, addr 155.1.0.1 7F322D228540**
 output chain:
 **IP midchain out of Tunnel0, addr 155.1.0.1 7F322D228540**
 **IP adj out of GigabitEthernet1.100, addr 169.254.100.1 7F322D228CE0**
```
Kanske får komma tillbaka och revidera texten lite när jag har bättre koll på DMVPN sen men förhoppningsvis är det inte allt för många felaktigheter.. :)
