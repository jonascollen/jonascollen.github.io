---
title: EIGRP - Debug dump
date: 2013-06-11 15:47
comments: true
categories: [EIGRP]
tags: [debug]
---
[![eigrp debug as](/assets/images/2013/06/eigrp-debug-as.png)](/assets/images/2013/06/eigrp-debug-as.png) 

Gjorde följande labb från GNS3-vault idag som inte var helt enkel. Scenariot är att du precis installerat en ny router (Jack) men saknar tillgång och dokumentation angående router John. Du behöver nu få igång en EIGRP-adjacency mellan dessa två, hur gör du (behöver få fram AS-numret på något vis)? Jag började först med att testa mig fram mellan alla EIGRP-debug kommandon utan något resultat. Satte upp EIGRP med AS 1 på Jack som test, men detta var det enda jag kunde se:
```
*Mar 1 00:54:57.899: EIGRP: Sending HELLO on FastEthernet0/0
*Mar 1 00:54:57.903: AS 1, Flags 0x0, Seq 0/0 idbQ 0/0 iidbQ un/rely 0/0
*Mar 1 00:55:02.439: EIGRP: Sending HELLO on FastEthernet0/0
*Mar 1 00:55:02.443: AS 1, Flags 0x0, Seq 0/0 idbQ 0/0 iidbQ un/rely 0/0
```
Tydligen så när det är mismatch på AS-nummer så discardas hello-paketet från John, Gjorde istället ett försök med en access-lista, ip access-list extended 100 permit eigrp host 192.168.12.2 any debug ip packet 100 visade så följande:
```
*Mar 1 00:57:03.131: IP: s=192.168.12.2 (FastEthernet0/0), d=224.0.0.10, len 60, rcvd 2
*Mar 1 00:57:07.643: IP: s=192.168.12.2 (FastEthernet0/0), d=224.0.0.10, len 60, rcvd 2
*Mar 1 00:57:11.943: IP: s=192.168.12.2 (FastEthernet0/0), d=224.0.0.10, len 60, rcvd 2
```
debug ip packet 100 detail var inte heller till mycket hjälp:
```
*Mar 1 00:57:53.027: IP: s=192.168.12.2 (FastEthernet0/0), d=224.0.0.10, len 60, rcvd 2, proto=88
*Mar 1 00:57:57.355: IP: s=192.168.12.2 (FastEthernet0/0), d=224.0.0.10, len 60, rcvd 2, proto=88
```
Efter lite googlande så visade det sig att det finns ett dolt kommando under debug ip packet, debug ip packet 100 **dump.**
```
*Mar 1 00:35:21.451: IP: s=192.168.12.2 (FastEthernet0/0), d=224.0.0.10, len 60, rcvd 2
07A014C0: 0100 5E00000A ..^...
07A014D0: CC0617A8 00000800 45C0003C 00000000 L..(....E@.<....
07A014E0: 01580BF6 C0A80C02 E000000A 0205EEC0 .X.v@(..`.....n@
**07A014F0: 00000000 00000000 00000000 0000000C ................**
07A01500: 0001000C 01000100 0000000F 00040008 ................
07A01510: 0C040102 ....
```
Det vi letar efter är raden med "leading 0's", dvs den 4:e raden. **0000000C** är ju bevisligen i hex, så om vi gör om detta till binärt får vi = 12. Låt oss testa!
```
Jack(config)#router eigrp 12
Jack(config-router)#network 0.0.0.0
Jack(config-router)#
*Mar 1 01:00:35.291: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 12: Neighbor 192.168.12.2 (FastEthernet0/0) is up: new adjacency
```
Kanske inte något man kommer använda varje dag direkt men ändå. :)