---
layout: post
title: Ny labbmiljö på ingång..
date: 2018-03-17 18:43
comments: true
categories: [Homelab]
tags: [R610]
---
Har precis lagt beställning på en ny labbmiljö som förhoppningsvis ska göra resan mot CCIE lite smidigare. Försökte bygga lite mer framtidssäkert utifall jag även känner för att gå vidare mot CCIE SP sen (vilket vore rätt naturligt sett till mitt nuvarande jobb).

[![](/assets/images/2018/03/R610-Blank.jpg)](/assets/images/2018/03/R610-Blank.jpg) Specs enligt följande:

*   Dell R610
*   2x Quad core Xeon 5640
*   96GB RAM
*   3x 500GB HDD i RAID5
*   2x 717W PSU
*   4x Gigabit Ethernet
*   VMware ESXi

Valet föll på en R610 istället för R710 då vad jag lyckats läsa mig till så ska den vara snäppet tystare. Förhoppningsvis låter den inte som en jetmotor på idle men det återstår att se.. :) Denna kommer i sin tur att driva:

*   20x CSR-1000V
*   4x XRv
*   2012 Windows Server för Wireshark
*   GNS3 VM
*   Debian Webbserver (Flask/Python) för att ersätta min trogna lilla raspberry

Sedan återstår att se om jag kommer komplettera med 4x 3560 framöver eller om det känns smidigare att hyra labbtid på ex. INE.
