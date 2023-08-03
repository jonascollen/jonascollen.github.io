---
title: Multipoint Broadcast routing & Proxy-ARP
date: 2018-04-04 19:49
author: Jonas Collén
comments: true
categories: [Route Manipulation]
tags: [proxy arp, static arp]
---
![](/assets/images/2018/04/topologi.png) 

Back to basics, en liten labb på hur proxy-arp fungerar när vi routar ut på ett multipoint (broadcast) interface med/utan proxy-arp (default alltid på i ios).

```
R1#sh run int gi1.146
 interface GigabitEthernet1.146
 encapsulation dot1Q 146
 ip address 155.1.146.1 255.255.255.0
 ipv6 address 2001:155:1:146::1/64

R1#sh ip int gi1.146 | inc Proxy
 **Proxy ARP is enabled**
```

Labben gick ut på följande:

*   Skapa en statisk route i R1 för R4's Loopback med R4 som nexthop
*   Skapa en statisk route i R1 för R6's Loopback med endast utgående interface som nexthop
*   Stängt av proxy-arp i R6 och fixa så dess loopback fortfarande är nåbar från R1

Ezpz, vi lägger in statiska rötterna till att börja med.

```
R1(config)#ip route 150.1.4.4 255.255.255.255 155.1.146.4
R1(config)#ip route 150.1.6.6 255.255.255.255 GigabitEthernet1.146

R1#sh ip route static | beg Gate
Gateway of last resort is not set

150.1.0.0/32 is subnetted, 3 subnets
S 150.1.4.4 \[1/0\] via 155.1.146.4
S 150.1.6.6 is directly connected, GigabitEthernet1.146

R1#ping 150.1.4.4 rep 1 
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 150.1.4.4, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 6/6/6 ms

R1#ping 150.1.6.6 rep 1
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 150.1.6.6, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 6/6/6 ms
```

Allt bra så långt, vad händer när vi stänger av Proxy-arp i R6?

```
R6(config)#int gi1.146
R6(config-subif)#no ip proxy-arp

R1#clear arp
R1#ping 155.1.6.6 rep 1 
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 155.1.6.6, timeout is 2 seconds:
.
Success rate is 0 percent (0/1)
```

Kikar vi närmare på debug i R1 & R6 ser vi snabbt vad det är som går på tok.

```
R6#debug arp
ARP packet debugging is on 

R1#debug ip packet detail 
IP packet debugging is on (detailed)
R1#debug arp
ARP packet debugging is on

R1#ping 150.1.6.6 rep 1
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 150.1.6.6, timeout is 2 seconds:

IP: s=155.1.146.1 (local), d=150.1.6.6, len 100, local feature
 ICMP type=8, code=0, feature skipped, Auth Proxy(16), rtype 0, forus FALSE, sendself FALSE, mtu 0, fwdchk FALSE
FIBipv4-packet-proc: route packet from (local) src 155.1.146.1 dst 150.1.6.6
FIBfwd-proc: packet routed by adj to GigabitEthernet1.146 150.1.6.6
FIBipv4-packet-proc: packet routing succeeded
IP: tableid=0, s=155.1.146.1 (local), d=150.1.6.6 (GigabitEthernet1.146), routed via FIB
IP: s=155.1.146.1 (local), d=150.1.6.6 (GigabitEthernet1.146), len 100, sending
 ICMP type=8, code=0
**IP ARP: creating incomplete entry for IP address: 150.1.6.6 interface GigabitEthernet1.146**
**IP ARP: sent req src 155.1.146.1 000c.2947.8992,**
 **dst 150.1.6.6 0000.0000.0000 GigabitEthernet1.146.**
Success rate is 0 percent (0/1)
```

R1 broadcastar ut en ARP-request till 150.1.6.6, men då vi stängt av proxy-arp i R6 ignorerar den paketet trots att den känner till var adressen finns. Aktiverar vi proxy-arp temporärt så ser vi hur R6 svarar på arp-requesten med mac-adressen till Gi1.146.

```
IP ARP: **rcvd** req src 155.1.146.1 000c.2947.8992, dst **150.1.6.6** GigabitEthernet1.146
IP ARP: **sent** rep src **150.1.6.6 000c.29e8.1c2e,**
 dst 155.1.146.1 000c.2947.8992 GigabitEthernet1.146
```

Hur löser vi då detta? En statisk arp-mappning i R1.

```
R1(config)#arp 150.1.6.6 000c.29e8.1c2e arpa

R1#sh arp static 
Protocol Address Age (min) Hardware Addr Type Interface
Internet 150.1.6.6 - 000c.29e8.1c2e ARPA

R1#ping 150.1.6.6 rep 1
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to 150.1.6.6, timeout is 2 seconds:
!
Success rate is 100 percent (1/1), round-trip min/avg/max = 6/6/6 ms
```

Output från debug:

```
IP: s=155.1.146.1 (local), d=150.1.6.6, len 100, local feature
 ICMP type=8, code=0, feature skipped, Auth Proxy(16), rtype 0, forus FALSE, sendself FALSE, mtu 0, fwdchk FALSE
FIBipv4-packet-proc: route packet from (local) src 155.1.146.1 dst 150.1.6.6
FIBfwd-proc: packet routed by adj to GigabitEthernet1.146 150.1.6.6
FIBipv4-packet-proc: packet routing succeeded
IP: tableid=0, s=155.1.146.1 (local), d=150.1.6.6 (GigabitEthernet1.146), routed via FIB
IP: s=155.1.146.1 (local), d=150.1.6.6 (GigabitEthernet1.146), len 100, sending
 ICMP type=8, code=0
IP: s=150.1.6.6 (GigabitEthernet1.146), d=155.1.146.1, len 100, enqueue feature
```

Bra exempel på varför man alltid ska ange nexthop för statiska rötter (undantaget p2p-länkar), istället för att routa direkt via FIB måste den även göra ARP-uppslag på dst-adressen.
