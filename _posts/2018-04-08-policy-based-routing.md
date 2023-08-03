---
title: Policy-based routing
date: 2018-04-08 05:20
comments: true
categories: [Route Manipulation]
tags: [policy-based routing, route-maps]
---
![](/assets/images/2018/04/policy-based.png) 

Just a small lab taking a closer look at how policy-based routing works. 

**Requirements:**

1.  Configure a default-route on R3 towards R1s local interface (Gi1.13)
2.  Configure a default-route on R4 & R6 towards R1s local interface (Gi1.146)
3.  Configure a default-route on R5 towards R1s DMVPN-interface (Tu0)
4.  Configure static routes thru the DMVPN-cloud on R3 for R5's Lo0, and vice versa on R5 for R3's lo0
5.  Configure policy-routing on R1 so that traffic sourced from R4s G1.146 is routed to R3 over Gi1.13
6.  Configure policy-routing on R1 so that traffic sourced from R6s Gi1.146 is routed to R5 over DMVPN

Let's start with the basic static routes first:

```
! 1-1 R3

ip route 0.0.0.0 0.0.0.0 Gi1.13 155.1.13.1

! 2-1 R4 & R6

ip route 0.0.0.0 0.0.0.0 Gi1.146 155.1.146.1

! 3-1 R5

ip route 0.0.0.0 0.0.0.0 Tu0 155.1.0.1

! 4-1
! R3

ip route 150.1.5.5 255.255.255.255 Tu0 155.1.0.5

! R5

ip route 150.1.3.3 255.255.255.255 Tu0 155.1.0.3
```

That should be all the routing we need to start with. Notice however that R1 doesn't have a clue what to do with the packets trying to reach loopbacks and will instead timeout.

```
R1#sh ip route static 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
 D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
 N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
 E1 - OSPF external type 1, E2 - OSPF external type 2
 i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
 ia - IS-IS inter area, \* - candidate default, U - per-user static route
 o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
 a - application route
 + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

```


We'll solve this problem by using policy-based routing and manually setting next-hop ip-address depending on source address instead. First we need access-lists to separate traffic coming from R4 and R6.

```
ip access-list extended SRC-R4
 10 permit ip host 155.1.146.4 any
ip access-list extended SRC-R6
 10 permit ip host 155.1.146.6 any
```

Next step is to use a route-map to manually set the next-hop address depending on which ACL matches. 

```
route-map PBR\_LAB permit 10
 match ip address SRC-R4
 set ip next-hop 155.1.13.3
route-map PBR\_LAB permit 20
 match ip address SRC-R6
 set ip next-hop 155.1.0.5
```

Last step is to activate policy-based routing on our incoming interface, in this case R1s Gi1.146.

```
interface Gi1.146
 ip policy route-map PBR\_LAB
```
All done! To verify that traffic is routed correctly we can use traceroute.

```
R4#ping 150.1.3.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.3.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/6/10 ms

R4#traceroute 150.1.3.3
Type escape sequence to abort.
Tracing the route to 150.1.3.3
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 5 msec 4 msec 4 msec
 **2 155.1.13.3 4 msec \* 3 msec**

R6#ping 150.1.5.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.5.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/12/27 ms

R6#traceroute 150.1.5.5
Type escape sequence to abort.
Tracing the route to 150.1.5.5
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 5 msec 5 msec 4 msec
 **2 155.1.0.5 5 msec**

R1#sh route-map 
route-map PBR\_LAB, permit, sequence 10
 Match clauses:
 ip address (access-lists): SRC-R4 
 Set clauses:
 ip next-hop 155.1.13.3
 **Policy routing matches: 20 packets, 1280 bytes**
route-map PBR\_LAB, permit, sequence 20
 Match clauses:
 ip address (access-lists): SRC-R6 
 Set clauses:
 ip next-hop 155.1.0.5
 **Policy routing matches: 14 packets, 1004 bytes**
```

If we however change the source-address in either R4 or R6 our access-list in R1 will no longer match, so packets will just timeout instead.

```
R6#ping 150.1.5.5 source lo0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.5.5, timeout is 2 seconds:
Packet sent with a source address of 150.1.6.6 
.....
Success rate is 0 percent (0/5)
```

For more information how to configure policy-based routing I managed to find the section in CiscoDoc at:  _Configuration Guides -> IP Routing: Protocol-Independent Configuration Guide - > Policy-based Routing_
