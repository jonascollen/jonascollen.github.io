---
title: CCDP - MPLS & MP-BGP
date: 2014-01-08 18:53
comments: true
categories: [BGP, MPLS, Route Manipulation]
tags: [vrf]
---
Är mitt uppe i tentavecka så finns inte direkt någon tid att posta här just nu tyvärr, räknar med att vara "back on track" till nästa vecka igen. Tänkte hur som helst ta och skriva ett kortare inlägg om implementeringen av MPLS & MP-BGP för en mindre ISP i projektet som nämndes i gårdagens inlägg. 

![projekt-light](/assets/images/2014/01/projekt-light.png)

Mitt uppdrag var att föreslå förbättringar och ta fram detaljerade designförslag för både core-nät och internkontor utifrån ovanstående topologi samt följande information:

*   Agerar ISP åt börsnoterade företag
*   IPv4 37.0.0.0/8
*   IPv6 2001:beef::/16
*   1000st anställda
    *   Prag 500st (Huvudkontor)
    *   Brno 300st
    *   Plzeñ 50st
    *   Cern (Schweiz) 25st
    *   Amsterdam 25st
    *   Birmingham 25st
    *   Kista 25st
    *   Santa Cruz de Tenerife 25st
    *   Kiruna 25st
*   Består av dotterbolagen:
    *   NetherLight
    *   CanarieLight
    *   NorthernLight
    *   UKLight
*   Önskar informativa övervakningsmöjligheter för kundens länkar
*   Önskar erbjuda en kostnadseffektiv tjänst med QoS
*   Kontoren i Plzeñ & Brno erbjuder även Datacenter & Storage-lösningar

IGP
---

![igp](/assets/images/2014/01/igp.png)

Då det befintliga core-nätet helt saknar redundans (exempelvis endast en switch utplacerad i Amsterdam) föreslogs istället  ovanstående uppgradering av nätet. Core-nätet byggs upp i par för respektive land/ort (rött & blått), där varje nod förutom en direktlänk till sin överliggande motsvarighet även har en länk till sin ”core-partner”. 

Som underliggande IGP-protokoll till BGP användes OSPF #300 inom core-nätet. Detta då både OSPF & BGP tillskillnad från EIGRP är en öppen standard och tillåter företaget att använda en mixed-vendor lösning. OSPF låter oss även segmentera nätet i enlighet med de confederations som tagits fram för BGP, med undantaget att CERN här flyttades till en egen area för att hålla Area 0 så liten som möjligt.

Summering utförs i varje ABR, vilket blir väldigt enkelt tack vare att ip-adresseringen följer samma numrering som confederations & area-nummer (ex. NorthernLight 37.46.x.x/16). Reference-bandwith har även justerats till 2 Terabit då OSPF per default ej ser skillnad på länkar över 100Mbit. Konfigurationsexempel - NorthernLight KistaBlue

```
router ospf 300
 router-id 37.46.255.102
 log-adjacency-changes
 passive-interface default
 no passive interface Serial0/1
 no passive interface Serial0/2
 no passive interface FastEthernet0/0
 no passive interface FastEthernet0/1
 auto-cost reference-bandwidth 20000000
 area 46 range 37.46.0.0 255.255.0.0
 network 37.31.46.4 0.0.0.3 area 0
 network 37.46.0.0 0.0.255.255 area 46
```

BGP
---

BGP Confederations används för att segmentera nätet och minska antalet IBGP neighbor-relationer som annars skulle behövas, detta då IBGP kräver en full-mesh relation för att utbyta routinguppdateringar (förutsatt att de ej använder sig av route-reflectors). Nedan följer ett konfigurationsexempel för confederations i NetherLights PEs, exempelvis AmsterdamRed.

```
router bgp 31
 bgp confederation identifier 300
 bgp confederation peers 34 42 44 46
```

Inom varje confederation används sedan full-mesh IBGP peering, förutom i Czechlight där antalet BGP Speakers var något fler än övriga länder och nätet kompletterades därför även med Route-Reflectors.

![czechlights-bgp](/assets/images/2014/01/czechlights-bgp.png)

PE PragRed

```
router bgp 42
 no synchronization
 bgp router-id 37.42.63.101
 bgp cluster-id 1
 bgp log-neighbor-changes
 bgp confederation identifier 300
 bgp confederation peers 31 
 neighbor czechlight peer-group
 neighbor czechlight remote-as 42
 neighbor czechlight update-source Loopback0
 neighbor czechlight route-reflector-client
 neighbor 37.31.42.1 remote-as 31
 neighbor 37.31.42.1 description AmsterdamRed
 neighbor 37.42.63.102 peer-group czechlight
 neighbor 37.42.63.102 description PragBlue
 neighbor 37.42.127.101 peer-group czechlight
 neighbor 37.42.127.101 description PlzenRed
 neighbor 37.42.127.102 peer-group czechlight
 neighbor 37.42.127.102 description PlzenBlue
 neighbor 37.42.191.101 peer-group czechlight
 neighbor 37.42.191.101 description BrnoRed
 neighbor 37.42.191.102 peer-group czechlight
 neighbor 37.42.191.102 description BrnoBlue
 no auto-summary
```

MPLS
----

För att sätta upp MPLS användes följande konfig, vi behöver endast peera mot andra PEs över vpnv4 (vilket dock i detta nät blir varje core-router).


PE AmsterdamRed

```
mpls label protocol ldp
interface FastEthernet0/0
 description to AmsterdamBlue
 ip address 37.31.255.1 255.255.255.252
 mpls ip

interface Serial0/0
 description to PragRed
 ip address 37.31.42.1 255.255.255.252 
 mpls ip

interface FastEthernet0/1
 description to ISP-Peer

  interface FastEthernet0/1.10
   encapsulation dot1Q 1 native
   ip vrf forwarding Internet_Access
   ip address 37.31.255.5 255.255.255.252

interface Serial0/1
 description to KistaRed
 ip address 37.31.46.1 255.255.255.252
 mpls ip

ip bgp-community new-format

router bgp 31
 no synchronization
 bgp router-id 37.31.255.1
 bgp log-neighbor-changes
 bgp confederation identifier 300
 bgp confederation peers 32 42 44 46 
 neighbor 37.31.42.2 remote-as 42
 neighbor 37.31.42.2 description PragRed
 neighbor 37.31.46.2 remote-as 46
 neighbor 37.31.46.2 description KistaRed
 neighbor 37.31.255.102 remote-as 31
 neighbor 37.31.255.102 description AmsterdamBlue
 neighbor 37.31.255.102 update-source Loopback0
 no auto-summary

  address-family vpnv4
   neighbor 37.31.42.2 activate
   neighbor 37.31.42.2 send-community both
   neighbor 37.31.46.2 activate
   neighbor 37.31.46.2 send-community both
   neighbor 37.31.255.102 activate
   neighbor 37.31.255.102 send-community both
   exit-address-family

mpls ldp router-id Loopback0 force
```

VRFs
----

Följande VRFs sattes upp tillsvidare:

*   300:10 Light-Internal  
*   300:20 Light-Datacenter 
*   300:30 Light-Storage  
*   300:300 Internet_Access 
    *   Här importeras samtliga kund-VRF:er som önskar Internet-access och innehåller full BGP routing-table, kräver att kunden har en publik ip-adress och annonseras till andra peering-partners
*   300:301 Internet_Defaultroute 
    *   Innehåller endast en default-route ut mot internet (till VRF Internet_Access). Används för kunder som ej använder BGP och importeras in i kundens VRF (vi vill inte exportera full bgp-table till varje VRF-instans)

Konfigexempel:

```
ip vrf Internet_Access
 rd 300:300
 export map DEFAULT-INJECT
 route-target import 300:300
 route-target import 300:310
ip vrf Internet_Defaultroute
 rd 300:301
 route-target export 300:301
 route-target import 300:301
ip vrf LIGHT
 rd 300:10
 route-target export 300:10
 route-target import 300:10
```

För att exportera default-routen från Internet_Access till VRF: 300:301 användes en export-map (route-map).

```
ip prefix-list DEFAULT-ROUTE seq 5 permit 0.0.0.0/0
route-map DEFAULT-INJECT permit 10
 match ip address prefix-list DEFAULT-ROUTE
 set extcommunity rt 300:301
route-map DEFAULT-INJECT permit 20
 set extcommunity rt 300:300
```

Vill vi sedan ge en kund internetaccess exporterar vi deras VRF till 300:300 och importerar 300:301 enligt följande:

```
ip vrf Cust-FortiAB
 rd 300:1001
 route-target export 300:1001
 route-target export 300:300
 route-target import 300:1001
 route-target import 300:301
```

För att få in full bgp-table till VRF: Internet_Access behöver vi endast peera mot andra ISPs över den specifika VRF:en. För att säkerställa att varken våra peering-partner eller Lightkoncernen och dess externa kunder av misstag skulle börja annonsera privata adresser används prefixlistor för att filtrera routing-uppdateringar både in & ut.

```
interface FastEthernet0/1.10
 description to ISP-1
 encapsulation dot1Q 1 native
 ip vrf forwarding Internet_Access
 ip address 37.31.255.5 255.255.255.252

ip prefix-list rfc1918 deny 0.0.0.0/8 le 32
ip prefix-list rfc1918 deny 10.0.0.0/8 le 32
ip prefix-list rfc1918 deny 127.0.0.0/8 le 32
ip prefix-list rfc1918 deny 169.254.0.0/16 le 32
ip prefix-list rfc1918 deny 172.16.0.0/12 le 32
ip prefix-list rfc1918 deny 192.0.2.0.0/24 le 32
ip prefix-list rfc1918 deny 192.168.0.0/16 le 32
ip prefix-list rfc1918 deny 224.0.0.0/3 le 32
ip prefix-list rfc1918 permit 0.0.0.0/0 le 32

router bgp 31
 address-family ipv4 vrf Internet_Access
  neighbor 37.31.255.6 remote-as 100
  neighbor 37.31.255.6 activate
  neighbor 37.31.255.6 prefix-list rfc1918 in
  neighbor 37.31.255.6 prefix-list rfc1918 out
  aggregate-address 37.0.0.0 255.0.0.0 summary-only
  exit-address-family
```

Vilket ger följande routing-tabell på ISP-1:

```
ISP-1#sh ip route bgp
 200.200.200.0/32 is subnetted, 1 subnets
 B 200.200.200.200 [20/0] via 37.31.255.5, 00:02:59
 217.21.0.0/26 is subnetted, 2 subnets
 B 217.21.0.64 [20/0] via 37.31.255.5, 00:02:59
 B 217.21.0.0 [20/0] via 37.31.255.5, 00:02:59
 37.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
 **B 37.0.0.0/8 [20/0] via 37.31.255.5, 00:00:03**
```

Blev tyvärr en väldigt kort inlägg det här men har inte riktigt tid just nu att gå på djupet och förklara mer ingående, får ta nya tag efter nästa veckas tentor. :)