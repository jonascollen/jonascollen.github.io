---
title: TSHOOT - Part II, Frame-Relay, IPv6 & OSPF
date: 2013-09-18 16:38
comments: true
categories: [Troubleshoot]
---
![tshoot-ospf](/assets/images/2013/09/tshoot-ospf.png)
Frame-Relay är min överlägset svagaste sida så det här är verkligen välkommen repetition. Innan vi börjar med OSPF etc så lär vi först fixa basic L2-connectivity.

# Frame-Relay IPv4

R1
```
R1(config)#interface s1/0
 R1(config-if)#encapsulation frame-relay
 R1(config-if)#no shut
 R1(config-if)#
 R1(config-if)#interface s1/0.12 point-to-point
 R1(config-subif)#ip add 10.1.1.1 255.255.255.252
 R1(config-subif)#frame-relay interface-dlci 102
 R1(config-fr-dlci)#desc to R2
```
R2
```
R2(config)#interface s1/0
 R2(config-if)#encapsulation frame-relay
 R2(config-if)#no shut
 R2(config-if)#
 R2(config-if)#interface serial 1/0.21 point-to-point
 R2(config-subif)#ip add 10.1.1.2 255.255.255.252
 R2(config-subif)#desc to R1
 R2(config-subif)#frame-relay interface-dlci 201
 R2(config-fr-dlci)#
 R2(config-fr-dlci)#interface serial 1/0.23 point-to-point
 R2(config-subif)#ip add 10.1.1.5 255.255.255.252
 R2(config-subif)#desc to R3
 R2(config-subif)#frame-relay interface-dlci 203
 R2(config-fr-dlci)#end
```
R3
```
R3(config)#interface s1/0
 R3(config-if)#encapsulation frame-relay
 R3(config-if)#no shut
 R3(config-if)#
 R3(config-if)#interface serial 1/0.32 point-to-point
 R3(config-subif)#ip add 10.1.1.6 255.255.255.252
 R3(config-subif)#frame-relay interface-dlci 302
 R3(config-fr-dlci)#
 R3(config-fr-dlci)#interface serial 1/0.34 point-to-point
 R3(config-subif)#ip add 10.1.1.9 255.255.255.252
 R3(config-subif)#frame-relay interface-dlci 304
 R3(config-fr-dlci)#
```
R4
```
R4(config)#interface s1/0
 R4(config-if)#encapsulation frame-relay
 R4(config-if)#no shut
 R4(config-if)#
 R4(config-if)#interface serial 1/0.43 point-to-point
 R4(config-subif)#ip add 10.1.1.10 255.255.255.252
 R4(config-subif)#desc to R3
 R4(config-subif)#frame-relay interface-dlci 403
```
# Frame-Relay IPv6

R1
```
R1(config)#ipv6 unicast-routing
 R1(config)#int s1/0.12 point-to-point
 R1(config-subif)#ipv6 add 2026::12:1/122
```
R2
```
R2(config)#ipv6 unicast-routing
 R2(config)#interface s1/0.21 point-to-point
 R2(config-subif)#ipv6 add 2026::12:2/122
R2(config-subif)#interface s1/0.23 point-to-point
 R2(config-subif)#ipv6 add 2026::1:1/122
 R2(config-subif)#end
```
R3
```
R3(config)#ipv6 unicast-routing
 R3(config)#interface s1/0.32 point-to-point
 R3(config-subif)#ipv6 add 2026::1:2/122
 R3(config-subif)#end
```
Mellan R3 & R4 ska vi enligt skissen istället ha en GRE-tunnel. Se tidigare inlägg [här](http://roadtoccie.se/2013/08/20/ipv6-static-gre-6to4-tunnels/ "IPv6 – Static, 6to4 & ISATAP Tunnels") för mer info om hur vi sätter upp statiska/6to4/ISATAP-IPv6 tunnlar. R3
```
R3(config)#interface Tunnel0
 R3(config-if)#ipv6 address 2026::34:1/122
 R3(config-if)#tunnel source s1/0.34
 R3(config-if)#tunnel destination 10.1.1.10
 R3(config-if)#tunnel mode ipv6ip
```
R4
```
R4(config)#ipv6 unicast-routing
 R4(config)#interface Tunnel0
 R4(config-if)#ipv6 address 2026::34:2/122
 R4(config-if)#tunnel source s1/0.43
 R4(config-if)#tunnel destination 10.1.1.9
 R4(config-if)#tunnel mode ipv6ip
```
# OSPFv2

Inget avancerat här direkt, glöm bara inte att annonsera default-routen från R1 till övriga (default-information originate). R1
```
R1(config)#router ospf 1
 R1(config-router)#router-id 1.1.1.1
 R1(config-router)#network 10.1.1.0 0.0.0.3 area 12
 R1(config-router)#default-information originate
```
R2
```
R2(config)#router ospf 1
 R2(config-router)#router-id 2.2.2.2
 R2(config-router)#network 10.1.1.0 0.0.0.3 area 12
 R2(config-router)#network 10.1.1.4 0.0.0.3 area 0
```
R3 - Kom ihåg "Totally Not-so-stubby" för area 34!
```
R3(config)#router ospf 1
 R3(config-router)#router-id 3.3.3.3
 R3(config-router)#network 10.1.1.4 0.0.0.3 area 0
 R3(config-router)#network 10.1.1.8 0.0.0.3 area 34
 R3(config-router)#area 34 nssa no-summary
```
R4
```
R4(config)#router ospf 1
 R4(config-router)#router-id 4.4.4.4
 R4(config-router)#network 10.1.1.8 0.0.0.3 area 34
 R4(config-router)#area 34 nssa
```
# OSPFv3

R1
```
R1(config-rtr)#interface Serial1/0.12 point-to-point
R1(config-subif)# ipv6 ospf 6 area 12
```
R2
```
R2(config)#interface Serial1/0.21 point-to-point
R2(config-subif)# ipv6 ospf 6 area 12
R2(config-subif)#interface Serial1/0.23 point-to-point
R2(config-subif)# ipv6 ospf 6 area 0
```
R3
```
R3(config)#ipv6 router ospf 6
R3(config-rtr)#area 34 nssa no-summary
R3(config-rtr)#exit
R3(config)#interface Serial1/0.32 point-to-point
R3(config-subif)#ipv6 ospf 6 area 0
R3(config)#interface Tunnel0
R3(config-if)# ipv6 ospf 6 area 34
```
R4
```
R4(config)#ipv6 router ospf 6
 R4(config-rtr)#area 34 nssa
 R4(config-rtr)#exit
 R4(config)#interface Tunnel0
 R4(config-if)# ipv6 ospf 6 area 34
```
# Verifiering

En hel del adresser att testa så enklast är att göra ett TCL-script istället.
```
R1#tclsh
 R1(tcl)#tclsh
R1(tcl)#foreach address {
 +>(tcl)#209.65.200.241
 +>(tcl)#209.65.200.242
 +>(tcl)#209.65.200.226
 +>(tcl)#209.65.200.225
 +>(tcl)#10.1.1.1
 +>(tcl)#10.1.1.2
 +>(tcl)#10.1.1.5
 +>(tcl)#10.1.1.6
 +>(tcl)#10.1.1.9
 +>(tcl)#10.1.1.10
 +>(tcl)#2026::12:1
 +>(tcl)#2026::12:2
 +>(tcl)#2026::1:1
 +>(tcl)#2026::1:2
 +>(tcl)#2026::34:1
 +>(tcl)#2026::34:2
 +>(tcl)#} { ping $address re 3 }
 Translating "209.65.200.241"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 209.65.200.241, timeout is 2 seconds:
 ..!
 Success rate is 33 percent (1/3), round-trip min/avg/max = 136/136/136 ms
 Translating "209.65.200.242"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 209.65.200.242, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 16/34/60 ms
 Translating "209.65.200.226"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 209.65.200.226, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 20/32/44 ms
 Translating "209.65.200.225"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 209.65.200.225, timeout is 2 seconds:
 ...
 Success rate is 0 percent (0/3)
 Translating "10.1.1.1"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 10.1.1.1, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 28/58/76 ms
 Translating "10.1.1.2"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 10.1.1.2, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 28/34/48 ms
 Translating "10.1.1.5"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 10.1.1.5, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 8/38/76 ms
 Translating "10.1.1.6"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 10.1.1.6, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 60/89/112 ms
 Translating "10.1.1.9"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 10.1.1.9, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 36/64/92 ms
 Translating "10.1.1.10"
Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 10.1.1.10, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 92/118/152 ms
 Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 2026::12:1, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 0/1/4 ms
 Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 2026::12:2, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 8/46/84 ms
 Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 2026::1:1, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 16/25/36 ms
 Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 2026::1:2, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 44/70/88 ms
 Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 2026::34:1, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 24/52/68 ms
 Type escape sequence to abort.
 Sending 3, 100-byte ICMP Echos to 2026::34:2, timeout is 2 seconds:
 !!!
 Success rate is 100 percent (3/3), round-trip min/avg/max = 84/112/160 ms
 R1(tcl)#tclquit
 R1#ping 2026::34:2
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 2026::34:2, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 36/100/160 ms
 R1#ping ipv6 2026::34:2
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 2026::34:2, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 60/80/104 ms
 R1#ping 2026::34:1
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 2026::34:1, timeout is 2 seconds:
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 40/56/76 ms
 ```
 