---
title: Reliable Policy-based routing
date: 2018-04-11 20:34
comments: true
categories: [Route Manipulation]
tags: [policy routing, verify-availability]
---
Here's another lab taking on how to set up reliable policy-based routing. The basic configuration is identical to a [previous lab](https://www.jonascollen.se/posts/policy-based-routing/) I did about policy-routing, so if you're interested in the basics I suggest you also take a look there for an easier intro. This one ends up using some pretty nifty functions within route-maps to set up a secondary next-hop tied in with IP SLA-tracking & CDP that I haven't seen earlier. 

![](/assets/images/2018/04/policy-based.png)

**Lab requirements:** Basic routing:

1.  Configure a default-route on R3 towards R1s local interface (Gi1.13)
2.  Configure a default-route on R4 & R6 towards R1s local interface (Gi1.146)
3.  Configure a default-route on R5 towards R1s DMVPN-interface (Tu0)
4.  Configure static routes thru the DMVPN-cloud on R3 for R5’s Lo0, and vice versa on R5 for R3’s lo0

Policy-routing:

1.  Configure policy-routing on R1 so that traffic sourced from R4s G1.146 is routed to R3 over Gi1.13
2.  Configure policy-routing on R1 so that traffic sourced from R6s Gi1.146 is routed to R5 over DMVPN

Reliability:

1.  Configure CDP between R1 & R5 over DMVPN-cloud
2.  Configure IP SLA on R1 that confirms connection to R3 every 5 seconds
3.  Modify R1's policy-routing so that if R1 loses ICMP reachability to R3, traffic from R4 is rerouted to R5 over the DMVPN-cloud
4.  Modify R1's policy-routing so that if R1 loses R5 as CDP-neighbor, traffic from R6 is rerouted to R3 over the ethernet link (Gi1.13)

Let's start with the easy stuff, the basic routing is very... basic, no explanation needed.

```
! Basic-1 R3

ip route 0.0.0.0 0.0.0.0 Gi1.13 155.1.13.1

! Basic-2
! R4

ip route 0.0.0.0 0.0.0.0 Gi1.146 155.1.146.1

! R6

ip route 0.0.0.0 0.0.0.0 Gi1.146 155.1.146.1

! Basic-3 R5

ip route 0.0.0.0 0.0.0.0 Tu0 155.1.0.1

! Basic-4
! R3
ip route 150.1.5.5 255.255.255.255 Tu0 155.1.0.5

! R5
ip route 150.1.3.3 255.255.255.255 Tu0 155.1.0.3
```
As in the previous lab, R1 has no routes so we won't be able to pass much traffic yet.  Let's take a look at setting up the policy-routing prerequisites so we can start testing basic functionality. First we'll need an access-list to match the source-address of R4 & R6, then we manually set next-hop with a route-map and attach it to our incoming interface.

```
! Policy-routing R1

ip access-list extended R4
 permit ip host 155.1.146.4 any

ip access-list extended R6
 permit ip host 155.1.146.6 any

route-map RPR permit 10
 match ip address R4
 set ip next-hop 155.1.13.3

route-map RPR permit 20
 match ip address R6
 set ip next-hop 155.1.0.5

interface Gi1.146
 ip policy route-map RPR
```

We should now have basic connectivity finally, packets from R4 should be routed to R3 first whatever the destination:.

```
R4#ping 150.1.5.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.5.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/6/6 ms

R4#traceroute 150.1.5.5
Type escape sequence to abort.
Tracing the route to 150.1.5.5
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 5 msec 4 msec 4 msec
 **2 155.1.13.3 5 msec 5 msec 5 msec**
 3 155.1.0.5 5 msec \* 5 msec

Packets from R6 should be routed to R5 first whatever the destination:

R6#ping 150.1.3.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.3.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/12/27 ms

R6#traceroute 150.1.3.3
Type escape sequence to abort.
Tracing the route to 150.1.3.3
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 5 msec 4 msec 4 msec
 **2 155.1.0.5 5 msec 5 msec 5 msec**
 3 155.1.0.3 5 msec \* 5 msec
```

Everything is looking fine so far! Now to the tricky part with setting up reliability, cdp is easy though. We just enable it on both of our tunnel interfaces:

```
! Reliability-1 R1 & R5
cdp run
interface Tu0
 cdp enable
```

Next requirement was to set up IP SLA that tracks the connection between R1 & R3's Gi1.13 every 5 seconds. We'll do this in R1 as it will be the router doing the policy-routing later.

```
! Reliability-2 R1
ip sla 100
 icmp-echo 155.1.13.3 source-interface Gi1.13
 frequency 5

ip sla schedule start-time now life forever

track 1 ip sla 100 state
```

Let's verify that everything's OK this far:

```
R1#sh ip sla statistics 100
IPSLAs Latest Operation Statistics
IPSLA operation id: 100
 Latest RTT: 4 milliseconds
Latest operation start time: 17:00:09 UTC Wed Apr 11 2018
Latest operation return code: OK
Number of successes: 63
Number of failures: 0
Operation time to live: Forever

R1#sh track
Track 1
 IP SLA 100 state
 State is Up
 3 changes, last change 00:16:29
 Latest operation return code: OK
 Latest RTT (millisecs) 3
 Tracked by:
 Route Map 0

R5#sh cdp neighbors detail 
-------------------------
**Device ID: R1**
Entry address(es): 
 **IP address: 155.1.0.1**
Platform: cisco CSR1000V, Capabilities: Router IGMP 
**Interface: Tunnel0, Port ID (outgoing port): Tunnel0**
Holdtime : 138 sec

Version :
Cisco IOS Software, CSR1000V Software (X86\_64\_LINUX\_IOSD-UNIVERSALK9-M), Version 15.5(3)S6, RELEASE SOFTWARE (fc3)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2017 by Cisco Systems, Inc.
Compiled Mon 24-Jul-17 20:01 by mcpre

advertisement version: 2
Management address(es): 
 IP address: 155.1.0.1
```

Next step on how to use tracking with policy-routing had me stuck for quite a while, scrolling thru the Cisco Docs I finally found a page that had some examples of what we're trying to do at: _Configuration Guides -> IP Routing: Protocol-Independent Configuration Guide – > [PBR Support for Multiple Tracking Options](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-pbr-mult-track.html)_ First requirement was to use our IP SLA 100 / Track 1 that's currently verifying connection is up between R1 & R3. Our route-map currently looks like this for reference:

```
route-map RPR permit 10
 match ip address R4
 set ip next-hop 155.1.13.3
route-map RPR permit 20
 match ip address R6
 set ip next-hop 155.1.0.5
```

We'll modify the first entry by removing the static next-hop for R4 and replace it with the command "_set ip next-hop verify-availability \[next-hop-address sequence track object\]"._ We also have to add something as a fallback next-hop if the tracker goes down, here the "set ip default" command looks like what we want to use. You can find more info here: _Configuration Guides -> IP Routing: Protocol-Independent Configuration Guide – > [Policy-Based Routing Default Next-Hop Routes](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-pbr-default-nexthop-route.html)_ _"set ip default next-hop ip-address \[ip-address\]"- Specifies the next hop for routing packets if there is no explicit route for this destination._ Our final route-map for policy-routing R4 will look like this:

```
route-map RPR permit 10
 match ip address R4
 set ip-next verify availability 155.1.13.3 1 track 1
 set ip default next-hop 155.1.0.5
```

For R6 I had much bigger problems however. Here we were supposed to use CDP for tracking but I couldn't manage to find any specific information about it more than that "set ip-next verify availability" has CDP-support. A quick google-search showed that it's really basic however, we only use the actual command "set ip-next verify availability" and IOS will automatically do CDP-lookups before setting the chosen next-hop, in this case we still need the static set ip next-hop however. The final result will look something like this:

```
route-map RPR permit 20
 match ip address R6
 set ip next-hop 155.1.0.5
 set ip next-hop verify-availability
 set ip default next-hop 155.1.13.3
```

Let's verify again:

```
R1#sh route-map
route-map RPR, permit, sequence 10
 Match clauses:
 ip address (access-lists): **R4** 
 Set clauses:
 ip next-hop verify-availability 155.1.13.3 1 track 1 **\[up\]**
 ip default next-hop 155.1.0.5
 **Policy routing matches: 6 packets, 276 bytes**
route-map RPR, permit, sequence 20
 Match clauses:
 ip address (access-lists): **R6** 
 Set clauses:
 ip next-hop 155.1.0.5
 ip next-hop verify-availability
 ip default next-hop 155.1.13.3
 **Policy routing matches: 18 packets, 828 bytes**
```

A traceroute from R4 should always take the route via R3 as long as there's reachability between R1 & R3:

```
R4#traceroute 150.1.5.5
Type escape sequence to abort.
Tracing the route to 150.1.5.5
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 8 msec 5 msec 5 msec
 **2 155.1.13.3 5 msec 5 msec 5 msec**
 3 155.1.0.5 5 msec \* 5 msec
```

Let's see what happens when we take down the connection between R1 & R3:

```
R1#debug ip policy 
Policy routing debugging is on

R3(config)#int gi1.13
R3(config-subif)#shut


PBR Nexthop: Unregister tracking for 155.1.13.3, handle: 7F74CD6A6768
**PBR Nexthop Callback invoked: 7F74F117B6C0, (155.1.13.3) tableid 0, status: 0, type: SET NEXTHOP TRACKINGmap: RPR, sequence: 10**
PBR Control Plane Notification: 155.1.0.5 PBR\_CP\_SET\_DEFAULT\_NEXTHOP
**Policy NextHop Inquiry: RPR seq: 10, type: SET DEFAULT NEXTHOP Nexthop: 155.1.0.5SW\_OBJ\_TYPE: 1E, SW\_HANDLE: 7F74F1108FE8**
PBR CP Notification sent: Type:SET DEFAULT NEXTHOP, 155.1.0.5SW\_OBJ\_TYPE: 1E, SW\_HANDLE: 7F74F1108FE8
```

Let's do another traceroute, the traffic should jump directly to R5 instead:

```
R4#traceroute 150.1.5.5
Type escape sequence to abort.
Tracing the route to 150.1.5.5
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 4 msec 4 msec 4 msec
 **2 155.1.0.5 4 msec \* 4 msec**
```

Sweet! Let's try the same for R6, all traffic should always first be redirected to R5:

```
R6#traceroute 150.1.3.3
Type escape sequence to abort.
Tracing the route to 150.1.3.3
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 5 msec 5 msec 4 msec
 **2 155.1.0.5 5 msec 5 msec 5 msec**
 3 155.1.0.3 5 msec \* 5 msec
```

Looking good so far. Lets disable CDP on R5s Tunnel0 interface and wait out the hold-timer (180 seconds), not a very fast switchover with default timers...

```
! R5
interface Tu0
 no cdp enable
```

After a long wait R5 is no longer showed in cdp-table:

```
R1#sh cdp neighbors 
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
 S - Switch, H - Host, I - IGMP, r - Repeater, P - Phone, 
 D - Remote, C - CVTA, M - Two-port Mac Relay

Device ID Local Intrfce Holdtme Capability Platform Port ID
R10 Gig 1 172 R I CSR1000V Gig 1
R2 Gig 1 126 R I CSR1000V Gig 1
R3 Gig 1 131 R I CSR1000V Gig 1
R6 Gig 1 132 R I CSR1000V Gig 1
R7 Gig 1 166 R I CSR1000V Gig 1
R4 Gig 1 151 R I CSR1000V Gig 1
R8 Gig 1 150 R I CSR1000V Gig 1
R9 Gig 1 158 R I CSR1000V Gig 1
```

A traceroute should now go directly to R3 instead (don't forget to open the interface we previously shutdown).

```
R6#traceroute 150.1.3.3
Type escape sequence to abort.
Tracing the route to 150.1.3.3
VRF info: (vrf in name/id, vrf out name/id)
 1 155.1.146.1 6 msec 5 msec 3 msec
 **2 155.1.0.5 5 msec 5 msec 5 msec**
 3 155.1.0.3 5 msec \* 5 msec
```
Not what we were hoping for.. Traffic is for some reason still routed over to R5 even though we have no CDP-reachability. I tried troubleshooting this forever but finally found a notation that there's apparently a bug in the CSR-1000V routers when it comes to verify-availability & CDP tracking(!!). The config looks OK though from other examples I could find so this is how it's "supposed" to work at least.. Fun times. :)
