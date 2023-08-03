---
title: RIPv2 Conditional default-routes
date: 2018-04-25 21:31
comments: true
categories: [RIP,Route Manipulation]
tags: [conditional routes]
---
![](/assets/images/2018/04/full_topology.png)

A few examples of advertising a default-route within RIPv2 using different techniques, some was a bit tricky to figure out. The requirements were as follows (three separate labs):

1.  From R6 - Advertise a default-route via RIP only outbound on Vl146, you are not allowed to use any access/prefix-lists
2.  From R4 - Advertise a default-route via RIP as long as R4 has a route to R9s loopback
3.  From R1 - Advertise a default-route via RIP as long as R1 has reachability to R7s LAN-interface 155.1.7.7, otherwise withdraw route

### First lab - Advertise by outbound interface

Advertising a default-route on a specific interface without filtering by accesss/prefix-list we could instead use a route-map.

```
! R6

route-map FILTER permit 10
 set interface Gi1.146

router rip
 default-information originate route-map FILTER
```

Big difference from ex. OSPF is that RIP doesn't require the route to be in it's actual routing-table to advertise it, which in turn leads to a routing loop in our topology. R6 advertise the route to R1 & R4, R1 will in turn advertise it to R7 who will forward it to R6. R6 will accept the route as it dosen't have a default-route in it's table and advertise that.

```
R6#sh ip rip database 0.0.0.0 0.0.0.0
0.0.0.0/0
 **\[4\]** via 155.1.67.7, 00:00:07, GigabitEthernet1.67

R6#sh ip rip database 0.0.0.0 0.0.0.0
0.0.0.0/0
 **\[8\]** via 155.1.67.7, 00:00:00, GigabitEthernet1.67

R1#sh ip route | beg Gate
Gateway of last resort is 155.1.146.6 to network 0.0.0.0

R\* 0.0.0.0/0 **\[120/13\]** via 155.1.146.6, 00:00:02, GigabitEthernet1.146
```

This can be solved in many ways, I chose to insert a dummy default-route to null0, but you could also use filtering etc.

```
! R6

ip route 0.0.0.0 0.0.0.0 null0
```

R6 will now ignore the default-route advertisement from R7 and not propagate it any further.

```
R6#sh ip rip database 0.0.0.0 0.0.0.0
0.0.0.0/0 redistributed
 \[1\] via 0.0.0.0,

R8#sh ip rip database 
0.0.0.0/0 auto-summary
0.0.0.0/0
 \[3\] via 155.1.58.5, 00:00:09, GigabitEthernet1.58
```

### Second lab - Conditional default-route

This lab requires us to originate a default-route from R4 as long as it has a route to R9s loopback0, the final solution looked like this for me:

```
! R4

ip prefix-list R9 permit 150.1.9.9/32

route-map R9\_TRACKING permit 10
 match ip address prefix-list R9

ip route 0.0.0.0 0.0.0.0 null0

router rip
 default-information originate route-map R9\_TRACKING
```

The logic is that as long as our route-map matches the prefix-list of R9s loopback it will advertise the default-route, and we add a static route to avoid routing loops via the DMVPN-hub R5 (no split-horizon). Let's verify to be sure.

```
R5#sh ip route | inc 0.0.0.0
Gateway of last resort is 155.1.45.4 to network 0.0.0.0
R\* 0.0.0.0/0 \[120/1\] via 155.1.45.4, 00:00:05, GigabitEthernet1.45
```

If we shut R9s loopback the default-route should time out eventually.

```
! R9

int Lo0
 shut

R4#sh ip route 150.1.9.9
% Subnet not in table

R5#sh ip route 0.0.0.0 
% Network not in table
```

### Third lab - IP SLA & default-route

This lab requires us to advertise a default-route as long as R1 has reachability to R7s LAN-interface 155.1.7.7, otherwise withdraw route. So obviously we're looking at setting up IP SLA to start with.

```
! R1

ip sla 1
 icmp-echo 155.1.7.7
 frequency 5

ip sla schedule 1 start-time now life forever
track 1 ip sla 1

R1#sh track 1
Track 1
 IP SLA 1 state
 State is Up
```

I couldn't figure out how to use our tracker in RIP however, eventually I found a pretty neat solution that might not be the prettiest, but it does the trick. First we create a "dummy-route" together with our tracker.

```
! R1

ip route 169.254.254.1 255.255.255.255 null0 track 1
```

Next step we borrow from our second lab, we create a prefix-list matching our dummy-route together with a route-map that we then use as a condition for our default-route advertising.

```
! R1

ip prefix-list DUMMY\_FILTER permit 169.254.254.1/32

route-map DUMMY permit 10
 match ip address prefix-list DUMMY\_FILTER

router rip
 default-information originate route-map DUMMY
```

The logic is, when our tracker (testing icmp-reachability to 155.1.7.7) goes down, our dummy static route will be removed from the routing table. This will in turn make rip stop advertising (or rather poison-reverse) the default as our route-map no longer has any match. Let's try it!

```
! R7

interface Gi1.7
 shut

! R1

R1#debug ip rip
RIP protocol debugging is on
R1#
**%TRACK-6-STATE: 1 ip sla 1 state Up -> Down**
R1#
RIP: sending v2 flash update to 224.0.0.9 via Loopback0 (150.1.1.1)
RIP: build flash update entries
 **0.0.0.0/0 via 0.0.0.0, metric 16, tag 0**
 **155.1.7.0/24 via 0.0.0.0, metric 16, tag 0**
```

As our tracker goes down, R1 poisons the default-route and it will eventually timeout in our other routers.

```
R2#sh ip route 0.0.0.0 
% Network not in table
```

Fun stuff, even RIP can be pretty tricky even though it's such a basic protocol compared to the rest. :)
