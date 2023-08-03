---
title: EIGRP - The basics
date: 2013-06-04 20:40
comments: true
categories: [EIGRP]
---
*   Distance Vector Protokoll (_hybrid_)
*   EIGRP-trafik skickas via multi- & unicast
*   Klarar förutom IPv4 även IPv6 och äldre protokoll som AppleTalk & IPX
*   Protokollnummer 88

**Tables** **Neighbor Table**

*   Samtliga neighbors som är directly connected (_Nexthop + Interface_)

_R2#sh ip eigrp neighbors_  _IP-EIGRP neighbors for process 1_ _H Address Interface Hold Uptime SRTT RTO Q Seq_ _(sec) (ms) Cnt Num_ _0 10.0.0.1 Se0/0 13 00:04:58 56 336 0 3_ **Topology Table**

*   Samtliga routes den lärt sig av sina neighbors (_Destination + Metric_)
*   Successor (_Bäst/Lägst metric_) / Feasible Successor routes (_Backup-route_)

_R2#sh ip eigrp topology_  _IP-EIGRP Topology Table for AS(1)/ID(2.2.2.2)__Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,_ _r - reply Status, s - sia Status_ _P 1.1.1.1/32, 1 successors, FD is 2297856_ _via 10.0.0.1 (2297856/128256), Serial0/0_ _P 2.2.2.2/32, 1 successors, FD is 128256_ _via Connected, Loopback0_ _P 10.0.0.0/30, 1 successors, FD is 2169856_ _via Connected, Serial0/0_ **Routing Table**

*   Standard, innehåller de bäst utträknade route's (_via DUAL_) från Topology table (_successor_)

_Gateway of last resort is not set__1.0.0.0/32 is subnetted, 1 subnets_ **_D 1.1.1.1 [90/2297856] via 10.0.0.1, 00:06:07, Serial0/0_** _2.0.0.0/32 is subnetted, 1 subnets_ _C 2.2.2.2 is directly connected, Loopback0_ _10.0.0.0/30 is subnetted, 1 subnets_ _C 10.0.0.0 is directly connected, Serial0/0_ **Metric** EIGRP Metric = 256*((K1*Bw) + (K2*Bw)/(256-Load) + K3*Delay)*(K5/(Reliability + K4)))

*   K1 - Bandwith
*   K2 - Load
*   K3 - Delay
*   K4 - Reliability
*   K5 - MTU

Som standard används dock endast K1 & K3 (_värdet är satt till 1_) för att räkna ut metric, K2, K4 & K5 är per default satt till 0 och används ej. Detta kan verifieras genom: _R1#sh ip protocols_ _Routing Protocol is "eigrp 10"_ _Outgoing update filter list for all interfaces is not set_ _Incoming update filter list for all interfaces is not set_ _Default networks flagged in outgoing updates_ _Default networks accepted from incoming updates_ _**EIGRP metric weight K1=1, K2=0, K3=1, K4=0, K5=0**_ _EIGRP maximum hopcount 100_ _EIGRP maximum metric variance 1_ Vi får då istället följande uträkning: EIGRP Metric = 256*(Bw + Delay). Tyvärr är det inte riktigt så enkelt, då både Bw (Bandwith) och Delay har egna utträkningar. Bw = (10^7/minimum Bw in kilobits per second) Delay =  Route delay in tens of microseconds Den slutgiltiga uträkningen blir således: EIGRP Metric = 256*((10^7 / min. Bw) + Delay) Bandwith syftar förövrigt på den **lägsta** bandbredden mellan punkt A och punkt B, och delay hänvisar till den **totala** delayen mellan punkt A och B - viktigt att hålla isär dessa! [![EIGRP Metric  K-values](/assets/images/2013/06/2-routrar1.jpg)](/assets/images/2013/06/2-routrar1.jpg) Ett försök att räkna ut metric för ett loopback-interface på R1 från R2 kan se ut enligt följande. Vi loggar först in på R2, kom ihåg att vi letar efter den lägsta bandbredden samt den totala delay'en. _R2#sh int s0/0 | i BW_ _MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,_ Vi gör sedan samma sak på R1. _R1#sh int s0/0 | i BW_ _MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,_ _R1#sh int l0 | i BW_ _MTU 1514 bytes, BW 8000000 Kbit/sec, DLY 5000 usec,_ Så vi kan enkelt konstatera att den minsta bandbredden är 1544Kbit/sec, och den totala delayen 25000 usec. 25,000 / 10 (_Kom ihåg - Route delay in tens of microseconds_) = 2500. Om vi för in dessa värden i uträkningen får vi följande: EIGRP Metric = 256*((10^7 / 1544) + 2500) -> EIGRP Metric = 256*((10,000,000 / 1544) + 2500) EIGRP Metric = 256*(6476 + 2500) = 2297856 R2#sh ip eigrp top | beg 1.1.1.1 **P 1.1.1.1/32, 1 successors, FD is 2297856** via 10.0.0.1 (2297856/128256), Serial0/0 P 2.2.2.2/32, 1 successors, FD is 128256 via Connected, Loopback0 P 10.0.0.0/30, 1 successors, FD is 2169856 via Connected, Serial0/0 Suveränt! Fortsättning följer..