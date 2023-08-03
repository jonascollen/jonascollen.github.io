---
title: CCDP - Cisco's PPDIOO Lifecycle
date: 2013-11-19 16:41
comments: true
categories: [CCDP]
tags: [ppdioo]
---
![cisco-ppdioo](/assets/images/2013/11/cisco-ppdioo.jpg)
PPDIOO är Cisco's best practice-upplägg för hur ett nätverk ska implementeras/designas/uppgraderas och är uppbyggt enligt följande sex faser.

*   Prepare

Går egentligen endast ut på att sammanställa information från "High level managers" om vad företaget har för affärsmål, vilka produker de använder, funktioner som saknas och vad som skulle kunna tänkas behövas i framtiden. Exempel: Om kunden vill implementeras ett WLAN behöver vi först ta reda på mer information. Hur många användare kommer använda WLAN:et? Kommer det användas av endast anställda eller ska det även finnas tillgängligt för gäster? Vilka applikationer ska supportas? Säkerhet? Hastigheter? Nästa steg är ett leta efter hårdvara som matchar dessa krav.

*   Plan

Här gör vi en "audit" av det nuvarande nätverket och beroende på vilken typ av projekt vi ska implementera bestämmer vad vi behöver se över mer ingående. Exempelvis IOS Versioner, CPU/Minnes-användning, nuvarande trafiktyper och beslastning via Netwflow, länkbelaastningar etc. Nästa steg är att planera implementationen genom att göra steg för steg instruktioner, vi måste även inkludera "stopping points" där vi testar konfigurationen och "roll backs" om något skulle gå fel. Exempel: Din kund vill implementera ett 802.11N WLAN, vi behöver då bl.a. kontrollera om switcharna har stöd för PoE, finns det tillräckligt med tillgänglig bandbredd i det befintliga nätverket för att supporta dessa hastigheter m.m. Vi bör även utföra en site-survey och kartlägga eventuella störningskällor och existerande WLAN för kanal-planering.

*   Design

Baserat på vilka krav från kunden vi fått fram i "Prepare"-fasen och den tekniska information vi sammanställt i "Prepare"-fasen kan vi nu börja designa en ny topologi. Detta bör inkludera alla delar (IP-adressering, VLANs, redundans, säkerhet etc) som vi kan behöva senare i projektet. Exempel: Efter site-survey och inventering av det nuvarande nätverket kan vi designa en plan innehållande inköp av hårdvara (AP's/WLC/PoE-Switchar etc), ritning med placering av APs, nätverksdiagram, subnetting osv.

*   Implement

Det är nu vi implementerar vår nya topologi genom att installera ny hårdvara och konfigurera nätet. Om vi gjort ett bra jobb i "Plan"-fase ska det egentligen bara att följa dokumentationen steg för steg. Vi bör även testa lösningen flera gånger under tiden vi implementerar nätet för att upptäcka problem tidigt. Exempel : Hårdvaran vi beställt (AP, WLC, Radius-server & switchar) har nu anlänt till kunden. Vi följer implementeringsplanen och installerar switcharna, konfigurerar WLCs, sätter upp APs och konfigurerar alla enheter. Vi bör även testa WLAN:et så allt fungerar tillfredsställande innan vi exempelvis implementerar säkerhet som Dot1x och ACLs.

*   Operate

Det nya nätverket är nu i drift och används i det dagliga arbetet. Vi bör här även ha implementerat övervakning av både hårdvara och prestande samt se till att vi har de senaste mjukvaruuppdateringarna etc.

*   Optimize

Utifrån de eventuella problem/brister vi upptäcker i "Operate"-fasen bör nätet optimeras genom att införa både små och stora förbättringar till det nuvarande nätet. Om det är något väldigt kritiskt bör vi dock gå tillbaka till första steget igen och göra om hela cykeln. Exempel: Användare av WLANet klagar på att det är undermålig prestanda i en viss del av lokalen. Via en spectrum/protocol-analyzer upptäcker vi att det finns annan utrustning i närheten som också använder 5 GHz-bandet som vi missat i vår site-survey. Vi löser problemet genom att byta frekvens på närliggande APs.