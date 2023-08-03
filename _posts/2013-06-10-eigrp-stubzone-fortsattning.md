---
title: EIGRP - Stubzone fortsättning
date: 2013-06-10 23:17
comments: true
categories: [EIGRP]
tags: [sia, stubzone]
---
Fortsättning på föregående inlägg då jag inte riktigt lyckades greppa användandet av stubzones för att minska Query-overflow. Denna länk förklarade det väldigt bra med tydligt exempel på vad som kan gå snett om vi ej använder oss av stubzones, framförallt när det finns redundanta vägar. [http://routemyworld.com/2008/07/23/bsci-eigrp-queries-stuck-in-active-route-summarization-and-stub-routers/](http://routemyworld.com/2008/07/23/bsci-eigrp-queries-stuck-in-active-route-summarization-and-stub-routers/) TL:DR-versionen är att det inte direkt är till för att minska antal Query-paket som skickas totalt, utan mer det antal R1 (från vår gamla topologi) med efterföljande neighbors behöver skicka innan den får ett Reply-paket tillbaka, vilket i sin tur minskar risken för "Stuck in Active". Beteendet där R7 skickar en Query till R3 efter att den precis mottagit en Update förklarades klockrent här:

> Regarding R5's behavior in step 5... As R5 is a stub router, the R3 did not send it a Query in step 2. That means that in step 4 R5 still believes that it can get to 0.0.0.0/0 via R3 - USING THE OLD METRIC. More specifically, R5 has its feasible distance and R3's distance to 0.0.0.0/0 based on static route from R1. When R3 sends the Update with new metric to 0.0.0.0/0 in step 4, the R5 recalculates the new distance from R3 to the 0.0.0.0/0 and concludes that the R3 does not satisfy the feasibility condition anymore, as R3's new distance to 0.0.0.0/0 is higher than R5's current feasibility distance (never mind that FD is based on an old report from R3!). So R5 removes R3 from the list of feasible successors for 0.0.0.0/0 and checks if there is another one. Nope, R3 was the only one. So the R5 goes active on this particular route, puts FD to infinity, and sends Query to all the neighbors (all = just R3). The R3 sends a Reply (with the same metric as in the Update it sent a moment ago), but now the metric that R3 provides is better than the current one (infinity), so R5 accepts R3 as the successor. [http://packetlife.net/blog/2010/mar/22/understanding-eigrp-queries/](http://packetlife.net/blog/2010/mar/22/understanding-eigrp-queries/)

Detta var troligtvis sista posten relaterat till EIGRP, next up - OSPF.
