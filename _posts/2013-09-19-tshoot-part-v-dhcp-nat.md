---
title: TSHOOT - Part V, DHCP & NAT
date: 2013-09-19 21:44
comments: true
categories: [Troubleshoot]
---
![bgp-internet](/assets/images/2013/09/bgp-internet1.png)
Då var det endast NAT & DHCP kvar att konfa upp (tidigare inlägg finns [här](http://www.jonascollen.se/posts/troubleshooting/). Förhoppningsvis sparar jag in en del tid nu till certet när jag faktiskt "hittar i nätet" och bara behöver fokusera på att leta efter fel istället. Har sett att gns3vault.com har en hel del troubleshoot-labbar jag tänkte försöka mig på under veckan för lite extra träning. 

Men det är bara 7 dagar kvar nu och på något vis ska jag även försöka hinna nöta in repetition av Switch också... Att försöka sig på två cert samtidigt efter bara 3 veckors plugg = låååånga dagar (8-22 ftw). Ett under att jag inte drömmer om cisco än.Inbillar mig dock att switch är enklare än route, och tshoot räknar jag kallt med att inte behöva läsa någon teori inför. Så om allt går vägen så är jag faktiskt CCNP-certifierad nästa vecka, ett litet steg på vägen iaf..

### NAT

Gjorde det enkelt för mig och tillät NAT för alla 10.1.x.x-10.2.x.x adresser, men har ingen info om hur cisco själva gör, antagligen är det väl bara ex. vlan10&20 som NATas.

```
R1(config)#ip access-list extended NAT
R1(config-ext-nacl)#remark NAT-ACL
R1(config-ext-nacl)#permit ip 10.1.0.0 0.0.255.255 any
R1(config-ext-nacl)#permit ip 10.2.0.0 0.0.255.255 any
R1(config-ext-nacl)#exit
R1(config)#interface fa0/1
R1(config-if)#ip nat outside
R1(config-if)#interface s1/0.12 point-to-point
R1(config-subif)#ip nat inside
R1(config-subif)#exit
R1(config)#ip nat inside source list NAT interface fa0/1 overload

HostA#ping 209.65.200.241
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 209.65.200.241, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 172/207/236 ms

R1#sh ip nat translations
Pro Inside global Inside local Outside local Outside global
icmp 209.65.200.225:1 10.1.1.6:1 209.65.200.241:1 209.65.200.241:1
icmp 209.65.200.225:34 10.2.1.10:34 209.65.200.241:34 209.65.200.241:34
```
Vackert. :)

### DHCP

R4
```
R4(config)#ip dhcp excluded-address 10.2.1.1 10.2.1.10
R4(config)#ip dhcp excluded-address 10.2.1.254
R4(config)#ip dhcp excluded-address 10.2.2.1 10.2.2.10
R4(config)#ip dhcp pool VLAN10
R4(dhcp-config)#network 10.2.1.0 /24
R4(dhcp-config)#domain-name TSHOOT
R4(dhcp-config)#dns-server 10.2.1.254
R4(dhcp-config)#default-router 10.2.1.254
R4(dhcp-config)#exit
R4(config)#ip dhcp pool VLAN20
R4(dhcp-config)#network 10.2.2.0 /24
R4(dhcp-config)#domain-name TSHOOT
R4(dhcp-config)#dns-server 10.2.2.1
R4(dhcp-config)#default-router 10.2.2.1
R4(dhcp-config)#exit
```

Då R4 ligger utanför broadcast-domänen för både VLAN10 & 20 behöver vi konfa upp ip-helpers på både DSW1 & DSW2. 

DSW1
```
DSW1(config)#inte vlan 10
DSW1(config-if)#ip helper-address 10.1.4.5
DSW1(config-if)#int vlan 20
DSW1(config-if)#ip helper-address 10.1.4.5
```
DSW2
```
DSW2(config)#inte vlan 10
DSW2(config-if)#ip helper-address 10.1.4.9
DSW2(config-if)#int vlan 20
DSW2(config-if)#ip helper-address 10.1.4.9
```
Vi kan verifiera så allt är ok med vår host på vlan10.
```
HostA(config)#inte fa0/0
HostA(config-if)#no ip add
HostA(config-if)#ip add dhcp
HostA(config-if)#end

HostA#sh inte fa0/0
FastEthernet0/0 is up, line protocol is up
 Hardware is AmdFE, address is cc04.05e0.0000 (bia cc04.05e0.0000)
 Description: Host-connection to Vlan 10
 **Internet address will be negotiated using DHCP**

 
*Mar 1 10:00:32.361: %DHCP-6-ADDRESS\_ASSIGN: Interface FastEthernet0/0 a**ssigned DHCP address 10.2.1.11, mask 255.255.255.0**, hostname HostA


HostA#sh inte fa0/0
FastEthernet0/0 is up, line protocol is up
 Hardware is AmdFE, address is cc04.05e0.0000 (bia cc04.05e0.0000)
 Description: Host-connection to Vlan 10
 **Internet address is 10.2.1.11/24**
```
Det var sista delen i min "TSHOOT"-serie. Vi har nu konfat upp hela det här nätet från grunden, mycket enklare än vad jag hade räknat med faktiskt. 
[!][tshoot-whiteboard](/assets/images/2013/09/tshoot-whiteboard.jpg)

Och det färdiga resultatet: 

[!][tshoot-done](/assets/images/2013/09/tshoot-done1.png)
GNS3-filen med färdiga konfigs finns att tanka hem [här](https://dl.dropboxusercontent.com/u/47263262/roadtoccie-tshoot.zip), troligtvis behöver du dock lägga till vlan:en på Dist & Access-switcharna igen.
