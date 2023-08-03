---
title: TSHOOT - Part III, Dist-layer
date: 2013-09-18 20:54
comments: true
categories: [Troubleshoot]
---
![tshoot-distlayer](/assets/images/2013/09/tshoot-distlayer.png)
Tar och bygger vidare på vårat tshoot-nät med Distribution-layer den här gången. Vi behöver bl.a. konfa upp EIGRP<->OSPF & RIP_NG <->OSPFv3 redistrubution. Men först måste vi givetvis fixa L2/L3-konfig.. Tyvärr är just switch-funktionen i GNS3 väldigt begränsad (använder endast NM-16ESW-kort i 3640s), så våra port-channel nummer kommer inte stämma överens. Det finns inte heller möjlighet att skapa en L3-channel mellan DSW1 & DSW2 så här blir det endast en port som kommer användas.

### Basic L3

R4

```
R4(config)#inte fa0/0
R4(config-if)#ip add 10.1.4.5 255.255.255.252
R4(config-if)#descrip to DSW1
R4(config-if)#no shut
R4(config-if)#ipv6 add 2026::2:1/122
R4(config-if)#inte fa0/1
R4(config-if)#ip add 10.1.4.9 255.255.255.252
R4(config-if)#desc to DSW2
R4(config-if)#no shut
```
DSW1
```
DSW1(config)#ip routing
DSW1(config)#ipv6 unicast-routing
DSW1(config)#int fa0/0
DSW1(config-if)#ip add 10.1.4.6 255.255.255.252
DSW1(config-if)#descrip to R4
DSW1(config-if)#no shut
DSW1(config-if)#ipv6 add 2026::2:2/122
DSW1(config)#int range fa1/13 - 14
DSW1(config-if-range)#descrip L3 Etherchannel to DSW2
DSW1(config-if-range)#no switchport
DSW1(config-if-range)#shut
DSW1(config-if-range)#
DSW1(config-if-range)#inte fa1/13
DSW1(config-if)#ip add 10.2.4.13 255.255.255.252
DSW1(config-if)#ipv6 add 2026::3:1/122
DSW1(config-if)#no shut
```
DSW2
```
DSW2(config)#ip routing
DSW2(config)#ipv6 uni
DSW2(config)#ipv6 unicast-routing
DSW2(config)#inte fa0/0
DSW2(config-if)#ip add 10.1.4.10 255.255.255.252
DSW2(config-if)#descrip to R4
DSW2(config-if)#no shut
DSW2(config-if)#int range fa1/13 - 14
DSW2(config-if-range)#descrip L3 Etherchannel to DSW1
DSW2(config-if-range)#no switchport
DSW2(config-if-range)#shut
DSW2(config-if-range)#inte fa1/13
DSW2(config-if)#ip add 10.2.4.14 255.255.255.252
DSW2(config-if)#no shut
DSW2(config-if)#ipv6 add 2026::3:2/122
DSW2(config-if)#exit
```
### EIGRP

R4
```
R4(config)#router eigrp 10
R4(config-router)#no auto
R4(config-router)#no auto-summary
R4(config-router)#passive-
R4(config-router)#passive-interface default
R4(config-router)#no passive
R4(config-router)#no passive-interface fa0/0
R4(config-router)#no passive-interface fa0/1
R4(config-router)#network 10.1.4.4 0.0.0.3
R4(config-router)#network 10.1.4.8 0.0.0.3
```
DSW1
```
DSW1(config)#router eigrp 10
DSW1(config-router)#no auto-summary
DSW1(config-router)#passive-interface default
DSW1(config-router)#no passive-interface fa0/0
DSW1(config-router)#no passive-interface fa1/13
DSW1(config-router)#network 10.1.4.4 0.0.0.3
DSW1(config-router)#network 10.2.4.12 0.0.0.3
```
DSW2
```
DSW2(config)#router eigrp 10
DSW2(config-router)#no auto-summary
DSW2(config-router)#passive-interface default
DSW2(config-router)#no passive-interface fa0/0
DSW2(config-router)#no passive-interface fa1/13
DSW2(config-router)#network 10.1.4.8 0.0.0.3
DSW2(config-router)#network 10.2.4.12 0.0.0.3
```
### RIPng

R4
```
R4(config-router)#inte fa0/0
R4(config-if)#ipv6 rip RIP_ZONE enable
R4(config-if)#int fa0/1
R4(config-if)#ipv6 rip RIP_ZONE enable
```
DSW1
```
DSW1(config)#inte fa0/0
DSW1(config-if)#ipv6 rip RIP_ZONE enable
DSW1(config)#int fa1/13
DSW1(config-if)#ipv6 rip RIP_ZONE enable
```
DSW2
```
DSW2(config)#inte fa0/0
DSW2(config-if)#ipv6 rip RIP_ZONE enable
DSW2(config)#int fa1/13
DSW2(config-if)#ipv6 rip RIP_ZONE enable
```
### Redistribution

Då vi inte har multipoint-redistribution behöver vi inte använda oss av route-maps/tags i det här fallet.
```
R4(config)#router eigrp 10
R4(config-router)#redistribute ospf 1 metric 1500 1 255 1 1500
R4(config-router)#router ospf 1
R4(config-router)#redistribute eigrp 10 subnets
R4(config)#ipv6 router ospf 6
R4(config-rtr)#redistribute rip RIP_ZONE include-connected metric 20
R4(config)#ipv6 router rip RIP_ZONE
R4(config-rtr)#redistribute ospf 6 metric 5 include-connected
```
### Verifiering
```
DSW2#sh ipv6 rip database
RIP process "RIP_ZONE", local RIB
 2026::2:0/122, metric 2, installed
 FastEthernet1/13/FE80::CE03:13FF:FE54:F10D, expires in 165 secs
 2026::3:0/122, metric 2
 FastEthernet1/13/FE80::CE03:13FF:FE54:F10D, expires in 165 secs
 2026::34:0/122, metric 7, installed
 FastEthernet1/13/FE80::CE03:13FF:FE54:F10D, expires in 165 secs
 ::/0, metric 7, installed
 FastEthernet1/13/FE80::CE03:13FF:FE54:F10D, expires in 165 secs
```
Ping till R1 från DSW2
```
DSW2#ping ipv6 2026::12:1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2026::12:1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 136/160/200 ms
DSW2#ping 10.1.1.1
Translating "10.1.1.1"
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 52/111/192 ms
```
Vackert! En trace till webservern fungerar dock ej:
```
DSW2#traceroute 209.65.200.241 Translating "209.65.200.241"
Type escape sequence to abort.
 Tracing the route to 209.65.200.241
1 10.1.4.9 16 msec 48 msec 48 msec
 2 10.1.1.9 108 msec 48 msec 44 msec
 3 10.1.1.5 40 msec 80 msec 84 msec
 4 10.1.1.1 140 msec 108 msec 156 msec
 5 * * *
```
Detta beror helt enkelt på att vi inte konfigurerat upp någon NAT ännu i R1 så trafiken hittar inte tillbaka. Det fixar vi imorgon tillsammans med DHCP-tjänsten & access-layer. :)
