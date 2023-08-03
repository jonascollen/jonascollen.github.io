---
title: BGP - Advanced BGP Lab, del 2
date: 2013-11-29 16:20
comments: true
categories: [BGP]
---
### EBGP & Redistribution

*   Configure EBGP between AS-peers
*   Configure BGP authentication between R7 and R11, use password VAULT
*   Make sure all BGP neighbor relationships are working before you continue with the next steps.
*   Advertise all physical and loopback interfaces in BGP, you are not allowed to use the "network" command to achieve this.
*   Achieve full connectivity, every IP address should be pingable. Use a [TCLSH](http://gns3vault.com/Network-Services/tclsh-scripting.html) script to do this.

Detta blir en fortsättning på gårdagens lab som finns att läsa [BGP – Advanced BGP Lab, del 1](http://www.jonascollen.se/posts/bgp-gns3vault-com-advanced-bgp-lab-del-1/). Kom ihåg att varje BGP Speaker kräver en specifik route till respektive neighbor, en default-route räcker ej. Då vi endast annonserar Loopbacks inom IGPs/AS (förutom sub-AS10 & 20/confederation) behöver vi även sätta upp statiska routes. Glöm inte heller ebgp-multihop den här gången... :) 

**EBGP** **R2**

```
router bgp 100
 neighbor 4.4.4.4 remote-as 300
 neighbor 4.4.4.4 ebgp-multihop 255
 neighbor 4.4.4.4 update-source Loopback0
ip route 4.4.4.0 255.255.255.0 192.168.24.4
```
**R3**

```
router bgp 100
 neighbor 4.4.4.4 remote-as 300
 neighbor 4.4.4.4 ebgp-multihop 255
 neighbor 4.4.4.4 update-source Loopback0
ip route 4.4.4.0 255.255.255.0 192.168.34.4
```
**R4**

```
router bgp 10
 neighbor 2.2.2.2 remote-as 100
 neighbor 2.2.2.2 ebgp-multihop 255
 neighbor 2.2.2.2 update-source Loopback0
neighbor 3.3.3.3 remote-as 100
 neighbor 3.3.3.3 ebgp-multihop 255
 neighbor 3.3.3.3 update-source Loopback0
neighbor 6.6.6.6 remote-as 200
 neighbor 6.6.6.6 ebgp-multihop 2
 neighbor 6.6.6.6 update-source Loopback0
ip route 2.2.2.0 255.255.255.0 192.168.24.2
ip route 3.3.3.0 255.255.255.0 192.168.34.3
ip route 6.6.6.0 255.255.255.0 192.168.46.6
```

**R5**

```
router bgp 10
 neighbor 6.6.6.6 remote-as 200
 neighbor 6.6.6.6 ebgp-multihop 2
 neighbor 6.6.6.6 update-source Loopback0
ip route 6.6.6.0 255.255.255.0 192.168.56.6
```
EBGP-förhållandet mellan confederation-AS #10 & #20 konfigurerade vi upp i gårdagens inlägg [här](http://roadtoccie.se/2013/11/28/bgp-gns3vault-com-advanced-bgp-lab-del-1/ "BGP – Advanced BGP Lab, del 1"). **R6**

```
router bgp 200
 neighbor 4.4.4.4 remote-as 300
 neighbor 4.4.4.4 ebgp-multihop 2
 neighbor 4.4.4.4 update-source Loopback0
 neighbor 5.5.5.5 remote-as 300
 neighbor 5.5.5.5 ebgp-multihop 2
 neighbor 5.5.5.5 update-source Loopback0
ip route 4.4.4.0 255.255.255.0 192.168.46.4
 ip route 5.5.5.0 255.255.255.0 192.168.56.5
```

**R7**

```
router bgp 200
 neighbor 11.11.11.11 remote-as 400
 neighbor 11.11.11.11 ebgp-multihop 2
 neighbor 11.11.11.11 update-source Loopback0
ip route 11.11.11.0 255.255.255.0 192.168.117.11
```

**R9**

```
router bgp 20
 neighbor 10.10.10.10 remote-as 400
 neighbor 10.10.10.10 ebgp-multihop 2
 neighbor 10.10.10.10 update-source Loopback0
ip route 10.10.10.0 255.255.255.0 192.168.109.10
```

**R10**

```
router bgp 400
 neighbor 9.9.9.9 remote-as 300
 neighbor 9.9.9.9 ebgp-multihop 2
 neighbor 9.9.9.9 update-source Loopback0
ip route 9.9.9.0 255.255.255.0 192.168.109.9
```

**R11**
```
router bgp 400
 neighbor 7.7.7.7 remote-as 200
 neighbor 7.7.7.7 ebgp-multihop 2
 neighbor 7.7.7.7 update-source Loopback0
ip route 7.7.7.0 255.255.255.0 192.168.117.7
```

**Authentication** 
Enligt labben behöver vi sätta upp autentisering mellan R7 - R11 med lösenordet "VAULT".

**R7** 
```
 router bgp 200
 neighbor 11.11.11.11 password VAULT
```

**R11**
```
 router bgp 400
 neighbor 7.7.7.7 password VAULT
```

**Redistribution** 
Nästa steg är att annonsera alla fysiska interface (inkl. loopbacks) in i BGP, vi får ej använda "network". Enklast bör väl vara att köra redistribute på connected, route-map:en är bara för att göra det lite snyggare och sätta origin till IGP istället för "unknown".
```
route-map REDIST_C permit 10
 set origin igp

router bgp x
 redistribute connected route-map REDIST_C
```
La in ovanstående på samtliga routrar i topologin. Vilket gav följande i R1:
```
R1#sh ip bgp
 BGP table version is 10, local router ID is 1.1.1.1
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
 *> 1.1.1.0/24 0.0.0.0 0 32768 i
 r>i2.2.2.0/24 2.2.2.2 0 100 0 i
 r>i3.3.3.0/24 3.3.3.3 0 100 0 i
 * i4.4.4.0/24 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i5.5.5.0/24 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i6.6.6.0/24 4.4.4.4 0 100 0 300 200 i
 * i 4.4.4.4 0 100 0 300 200 i
 * i7.7.7.0/24 4.4.4.4 0 100 0 300 200 i
 * i 4.4.4.4 0 100 0 300 200 i
 * i8.8.8.0/24 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i9.9.9.0/24 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.12.0 2.2.2.2 0 100 0 i
 *> 0.0.0.0 0 32768 i
 * i192.168.13.0 3.3.3.3 0 100 0 i
 *> 0.0.0.0 0 32768 i
 *>i192.168.24.0 2.2.2.2 0 100 0 i
 *>i192.168.34.0 3.3.3.3 0 100 0 i
 * i192.168.45.0 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.46.0 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.56.0 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.58.0 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.67.0 4.4.4.4 0 100 0 300 200 i
 * i 4.4.4.4 0 100 0 300 200 i
 * i192.168.89.0 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.109.0 4.4.4.4 0 100 0 300 i
 * i 4.4.4.4 0 100 0 300 i
 * i192.168.117.0 4.4.4.4 0 100 0 300 200 i
 * i 4.4.4.4 0 100 0 300 200 i
```
Som synes är det endast näten inom AS100 den lägger till i routing-tabellen.. Anledningen till detta är rätt enkel, R1 har ingen route till 4.4.4.4 (next-hop). Enklaste lösningen är väl att konfigurera next-hop-self på våra border-routers istället. 

**R2**
```
router bgp 100
 neighbor 1.1.1.1 next-hop-self
```

**R3**
```
router bgp 100
 neighbor 1.1.1.1 next-hop-self
```
Och så vidare, behöver göra detta på varje border-router vars neighbor saknar egna routes för loopbacks i andra AS.

### TCL Verifiering
```
tclsh
foreach address {
1.1.1.1
2.2.2.2
3.3.3.3
4.4.4.4
5.5.5.5
6.6.6.6
7.7.7.7
8.8.8.8
9.9.9.9
10.10.10.10
11.11.11.11
} { ping $address repeat 1 }
```
**R1**
```
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 4/4/4 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 128/128/128 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 56/56/56 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 120/120/120 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 140/140/140 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 6.6.6.6, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 144/144/144 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 7.7.7.7, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 192/192/192 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 144/144/144 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 9.9.9.9, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 216/216/216 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 10.10.10.10, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 252/252/252 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 11.11.11.11, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 324/324/324 ms
```
**R5**
```
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 212/212/212 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 104/104/104 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 132/132/132 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 60/60/60 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 1/1/1 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 6.6.6.6, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 44/44/44 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 7.7.7.7, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 100/100/100 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 32/32/32 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 9.9.9.9, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 100/100/100 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 10.10.10.10, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 100/100/100 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 11.11.11.11, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 184/184/184 ms
```
**R11**
```
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 308/308/308 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 216/216/216 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 216/216/216 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 280/280/280 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 148/148/148 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 6.6.6.6, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 68/68/68 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 7.7.7.7, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 36/36/36 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 152/152/152 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 9.9.9.9, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 80/80/80 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 10.10.10.10, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 56/56/56 ms
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 11.11.11.11, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 1/1/1 ms
```

Vackert! Nu är det bara den roliga biten kvar med "path modifications" men det får allt vänta till nästa vecka då det är party ikväll. :)
