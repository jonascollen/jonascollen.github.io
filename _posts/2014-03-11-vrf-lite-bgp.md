---
title: VRF-Lite & BGP
date: 2014-03-11 10:39
comments: true
categories: [BGP, EIGRP, OSPF, Route Manipulation]
tags: [vrf-lite]
---
Tänkte skriva ett kortare inlägg om ett rätt intressant problem jag stötte på tidigare vilket löstes med hjälp av VRF-Lite och lite trixande med BGP. Topologin som önskades var enligt följande: 
![lightvrf](/assets/images/2014/03/lightvrf.png) 
Länken mot ISP-1 önskades vara primär pga bättre serviceavtal & bandbredd samtidigt som länken till ISP-2 endast skulle användas som backup. Både ISP-1 & 2 genererar en default-route samt annonserar varsitt 10.x.0.0/23-nät. Det fanns även önskemål att AS #666 skulle agera transit mellan ISP-1 & 2. 

![lightvrfigp](/assets/images/2014/03/lightvrfigp.png?w=630) 
Som IGP användes EIGRP inom AS #666 samt mellan R1 - ISP-1 och OSPF mellan R3 - ISP-2 där respektive länknät redistributas. Detta är egentligen helt onödigt men användes för att göra uppgiften lite mer komplicerad bara. :) 

Problematiken var dock att ISP-1 & ISP-2 består av en och samma router! Den fysiska topologin ser nämligen ut enligt följande: 
![lightvrftopologi](/assets/images/2014/03/lightvrftopologi.png) Med andra ord behöver vi dela upp R1 till två virtuella routrar med separata routing tables & bgp-adjacencys. Detta löser vi med hjälp av VRF-Lite! :) Låt oss ta och kika lite närmare på konfigen för respektive router. R2

```
interface Serial1/0
 ip address 12.0.0.2 255.255.255.252
 description Primary uplink to ISP-1
 no shut
interface Loopback3
 description For testing
 ip address 172.32.0.1 255.255.255.0
 no shut
interface Loopback4
 description For testing
 ip address 172.32.1.1 255.255.255.0
 no shut

interface FastEthernet0/0
 description to MLS1
 ip address 172.16.11.11 255.255.255.0
 ip hello-interval eigrp 101 2
 ip hold-time eigrp 101 6
 ip authentication mode eigrp 1 md5
 ip authentication key-chain eigrp 1 s3cr3t
 ip summary-address eigrp 101 172.32.0.0 255.255.254.0 5
 duplex full
 speed 100
 no shut
```
**IGP-konfig** 
Då vi endast vill redistributa in länknätet mellan R1 & ISP-1 till EIGRP 101 använde jag en en route-map, passade även på att tagga route'sen om vi skulle behöva utföra någon filtrering senare.

```
!Internal IGP-routing
router eigrp 101
 redistribute eigrp 666 route-map EIGRP666-EIGRP101
 passive-interface default
 no passive-interface FastEthernet0/1
 network 172.16.0.0
 network 172.32.0.0 0.0.0.255
 network 172.32.1.0 0.0.0.255
 no auto-summary
 eigrp router-id 172.16.99.11

ip prefix-list EIGRP666-EIGRP101 seq 5 permit 12.0.0.0/30

route-map EIGRP666-EIGRP101 permit 10
 match ip address prefix-list EIGRP666-EIGRP101
 set metric 100000 100 255 1 1500
 set tag 1666

route-map EIGRP666-EIGRP101 deny 20

!External IGP routing ISP-1
router eigrp 666
 passive-interface default
 no passive-interface Serial1/0
 network 12.0.0.0 0.0.0.3
 no auto-summary
```

**BGP** 

Inga konstigheter här, peer-groups för lite mer "kompakt" konfig.

```
router bgp 666
 no synchronization
 bgp log-neighbor-changes
 network 172.16.0.0
 network 172.32.0.0 mask 255.255.254.0
 redistribute eigrp 666

 neighbor 12.0.0.1 remote-as 65001
 neighbor 12.0.0.1 description ISP-1

 neighbor IBGP peer-group
 neighbor IBGP remote-as 666
 neighbor IBGP next-hop-self
 neigbhor IBGP password s3cr3t

 neighbor 172.16.11.1 peer-group IBGP
 neighbor 172.16.11.1 description MLS1
 neighbor 172.16.13.3 peer-group IBGP
 neighbor 172.16.13.3 description MLS2
 neighbor 172.16.33.33 peer-group IBGP
 neighbor 172.16.33.33 description R3
 no auto-summary
```
Konfigen är mer eller mindre identisk för R3.

**Default-route** 

Som nämndes tidigare önskades företaget att vi använda denna länk som primär, både ISP-1 & 2 genererade varsin default-route. Att ändra local pref för samtliga routes vi lär oss från ISP-1 hade inte varit något hit då det även skulle påverka trafik vi är transit för. Tänkte istället använda en route-map som sätter en högre local pref endast för default-routen. Vi behöver även se till så att vi ej annonserar default-routen vidare utanför vårat AS. R2

```
router bgp 666
 neighbor 12.0.0.1 prefix-list DEFAULT-ROUTE-BLOCK out
 neighbor 12.0.0.1 route-map ISP1-routes in

ip prefix-list DEFAULT-ROUTE-BLOCK seq 5 deny 0.0.0.0/0
ip prefix-list DEFAULT-ROUTE-BLOCK seq 10 permit 0.0.0.0/0 le 32

ip prefix-list default-route seq 5 permit 0.0.0.0/0

route-map ISP1-routes permit 10
 match ip address prefix-list default-route
 set local-preference 150

route-map ISP1-routes permit 20

R3

router bgp 666
 neighbor 13.0.0.1 prefix-list DEFAULT-ROUTE-BLOCK out
 neighbor 13.0.0.1 route-map ISP2-routes in

ip prefix-list DEFAULT-ROUTE-BLOCK seq 5 deny 0.0.0.0/0
ip prefix-list DEFAULT-ROUTE-BLOCK seq 10 permit 0.0.0.0/0 le 32

ip prefix-list default-route seq 5 permit 0.0.0.0/0

route-map ISP2-routes permit 10
 match ip address prefix-list default-route
 set local-preference 110

route-map ISP2-routes permit 20
```

Vilket ger följande resultat:

```
R3#sh ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 4
Paths: (2 available, best #1, table Default-IP-Routing-Table)
 Not advertised to any peer
 65001
  172.16.11.11 (metric 30720) from 172.16.11.11 (172.32.1.1)
   Origin IGP, metric 0, localpref 150, valid, internal, best
 65002
  13.0.0.1 from 13.0.0.1 (2.2.2.2)
   Origin IGP, metric 0, localpref 100, valid, external
```

**VRF-Lite / R1** 

Nu över till det lite roligare. :) VRF:er har vi ju redan konfat i flera tidigare inlägg, så detta är väl inte direkt något nytt men själva användningsområdet  är något jag aldrig stött på tidigare.

```
interface Loopback1
 ip vrf forwarding ISP-1
 ip address 10.1.0.1 255.255.255.0

interface Loopback2
 ip vrf forwarding ISP-1
 ip address 10.1.1.1 255.255.255.0

interface Loopback3
 ip vrf forwarding ISP-2
 ip address 10.2.0.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Loopback4
 ip vrf forwarding ISP-2
 ip address 10.2.1.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Loopback5
 ip vrf forwarding Shared
 ip address 1.1.1.1 255.255.255.255

interface Loopback6
 ip vrf forwarding Shared
 ip address 2.2.2.2 255.255.255.255

interface Loopback11
description Management - RID
ip address 11.11.11.11 255.255.255.255

interface Serial0/0/0
 description ISP-1 to CustomerA-R1
 ip vrf forwarding ISP-1
 ip address 12.0.0.1 255.255.255.252
 ip summary-address eigrp 666 10.1.0.0 255.255.254.0 5

interface Serial0/0/1
 description ISP-2 to CustomerA-R3
 ip vrf forwarding ISP-2
 ip address 13.0.0.1 255.255.255.252
 ip ospf 1 area 666
```

Vi konfar först upp några VRF-instanser, shared simulerar i detta fallet externa routes/internet. La även till en export-map för att endast exportera 1.1.1.1/32 och 2.2.2.2/32 från Shared till ISP-1 & ISP-2 vrf:erna.

```
ip vrf ISP-1
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1

ip vrf ISP-2
 rd 65000:2
 route-target export 65000:2
 route-target import 65000:2

ip vrf Shared
 rd 65000:3
 export map ISP-Loopback-Inject
 route-target export 65000:3
 route-target import 65000:3
 route-target import 65000:1
 route-target import 65000:2

route-map ISP-Loopback-Inject permit 10
 match ip address prefix-list ISP-1
 set extcommunity rt 65000:1 additive

route-map ISP-Loopback-Inject permit 20
 match ip address prefix-list ISP-2
 set extcommunity rt 65000:2 additive

route-map ISP-Loopback-Inject deny 30
```

**IGP** 

Då vi använder oss av vrf:er måste vi även justera våra IGP-instanser precis som tidigare.

```
router eigrp 666
 passive-interface default
 no passive-interface Serial0/0/0
 no auto-summary

 address-family ipv4 vrf ISP-1
  network 10.1.0.0 0.0.0.255
  network 10.1.1.0 0.0.0.255
  network 12.0.0.0 0.0.0.3
  no auto-summary
  autonomous-system 666
  exit-address-family
 eigrp router-id 1.1.1.1

router ospf 1 vrf ISP-2
 router-id 2.2.2.2
 log-adjacency-changes
 area 0 range 10.2.0.0 255.255.254.0
 passive-interface default
 no passive-interface Serial0/0/1
```

**BGP** 

Här stöter vi på ett litet problem då vi endast kan ha en aktiv BGP-instans, dvs skriver vi "router bgp 65001" kan vi ej konfa upp "router bgp 65002" för ISP-2 efteråt. BGP har ju dock som bekant en hel del roliga funktioner vi kan använda oss av, och i detta fall kan vi lösa problemet med hjälp av "[local-as](http://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13761-39.html)", "[no-prepend](http://www.cisco.com/c/en/us/td/docs/ios/12_2s/feature/guide/fsbgphla.html)" & "[replace-as](http://www.cisco.com/c/en/us/td/docs/ios/12_2s/feature/guide/fsbgpdas.html)". Klicka på respektive för mer info, Lostintransit.se har även en läsvärd artikel om detta [här](http://lostintransit.se/2012/08/13/bgp-local-as-command/)!

```
router bgp 65000
 no synchronization
 bgp log-neighbor-changes
 no auto-summary

 address-family ipv4 vrf Shared
  redistribute connected
  no synchronization
  exit-address-family

 address-family ipv4 vrf ISP-2
  neighbor 13.0.0.2 remote-as 666
  neighbor 13.0.0.2 local-as 65002 no-prepend replace-as
  neighbor 13.0.0.2 activate
  neighbor 13.0.0.2 default-originate
  no synchronization
  bgp router-id 2.2.2.2
  network 10.2.0.0 mask 255.255.255.0
  network 10.2.1.0 mask 255.255.255.0
  aggregate-address 10.2.0.0 255.255.254.0 summary-only
  exit-address-family

 address-family ipv4 vrf ISP-1
  neighbor 12.0.0.2 remote-as 666
  neighbor 12.0.0.2 local-as 65001 no-prepend replace-as
  neighbor 12.0.0.2 activate
  neighbor 12.0.0.2 default-originate
  no synchronization
  bgp router-id 1.1.1.1
  network 10.1.0.0 mask 255.255.255.0
  network 10.1.1.0 mask 255.255.255.0
  aggregate-address 10.1.0.0 255.255.254.0 summary-only
  exit-address-family
```
Vilket ger följande resultat:

```
R1#sh ip bgp neighbors 12.0.0.1
 BGP neighbor is 12.0.0.1, remote AS 65001, external link
 BGP version 4, remote router ID 1.1.1.1
 BGP state = Established, up for 00:45:25
 Last read 00:00:16, last write 00:00:31, hold time is 180, keepalive interval is 60 seconds

R3#sh ip bgp neighbors 13.0.0.1
 BGP neighbor is 13.0.0.1, remote AS 65002, external link
 BGP version 4, remote router ID 2.2.2.2
 BGP state = Established, up for 00:46:18
 Last read 00:00:05, last write 00:00:21, hold time is 180, keepalive interval is 60 seconds

R2#sh ip bgp vpnv4 vrf ISP-1
 BGP table version is 60, local router ID is 1.1.1.1
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
 Network Next Hop Metric LocPrf Weight Path
 Route Distinguisher: 65000:1 (default for vrf ISP-1) VRF Router ID 1.1.1.1
 *> 1.1.1.1/32 0.0.0.0 0 32768 ?
 *> 2.2.2.2/32 12.0.0.2 0 666 65002 ?
 s> 10.1.0.0/24 0.0.0.0 0 32768 i
 r> 10.1.0.0/23 0.0.0.0 32768 i
 s> 10.1.1.0/24 0.0.0.0 0 32768 i
 *> 10.2.0.0/23 12.0.0.2 0 666 65002 i
 *> 13.37.0.0/16 0.0.0.0 0 32768 ?
 *> 172.16.0.0 12.0.0.2 28416 0 666 i
 *> 172.32.0.0/23 12.0.0.2 128256 0 666 i

R2#sh ip bgp vpnv4 vrf ISP-1
 BGP table version is 60, local router ID is 1.1.1.1
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
 Network Next Hop Metric LocPrf Weight Path
 Route Distinguisher: 65000:1 (default for vrf ISP-1) VRF Router ID 1.1.1.1
 *> 1.1.1.1/32 0.0.0.0 0 32768 ?
 *> 2.2.2.2/32 12.0.0.2 0 666 65002 ?
 s> 10.1.0.0/24 0.0.0.0 0 32768 i
 r> 10.1.0.0/23 0.0.0.0 32768 i
 s> 10.1.1.0/24 0.0.0.0 0 32768 i
 *> 10.2.0.0/23 12.0.0.2 0 666 65002 i
 *> 13.37.0.0/16 0.0.0.0 0 32768 ?
 *> 172.16.0.0 12.0.0.2 28416 0 666 i
 *> 172.32.0.0/23 12.0.0.2 128256 0 666 i
```

Vi kan även verifiera att vår transit fungerar som önskat, ISP-1 har följande information för 2.2.2.2/32 som ligger på samma router men under ISP-2s VRF.

```
R2#sh ip bgp vpnv4 vrf ISP-1 2.2.2.2/32
 BGP routing table entry for 65000:1:2.2.2.2/32, version 50
 Paths: (1 available, best #1, table ISP-1)
 Not advertised to any peer
 **666 65002**
 12.0.0.2 from 12.0.0.2 (172.16.99.11)
 Origin incomplete, localpref 100, valid, external, best
 Extended Community: RT:65000:1
 mpls labels in/out 31/nolabel
```
Vackert! VRF-Lite ger oss ett enkelt sätt att segmentera upp en router om vi exempelvis måste separera två avdelningar från varandra som ansluter till en och samma router. Fler läsvärda artiklar om detta finns här: http://packetlife.net/blog/2009/apr/30/intro-vrf-lite/ http://ccieblog.co.uk/mpls/inter-vrf-routing
