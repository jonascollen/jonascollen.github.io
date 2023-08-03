---
title: Switching - RSTP
date: 2013-09-14 11:54
comments: true
categories: [Switch]
tags: [rstp]
---
Då STP är alldeles för långsamt att konvergera och vi inte direkt kan konfigurera PortFast överallt togs istället "Rapid Spanning-Tree" 802.1w, fram.

![RSTP-Ports](/assets/images/2013/09/rstp-ports.jpg) Nya Port-states:

*   Discarding (Blocking)
*   Learning
*   Forwarding

Port-Roles:

*   Root-port
*   Designated-port
*   Alternate-port
*   Edge-port
*   Backup-port

Link-types:

*   Point-to-point Link
*   Shared Link
*   Edge

Känns tyvärr lite ointressant att lägga ner en massa tid på att skriva ett långt inlägg om hur RSTP fungerar. Kommer nog istället hålla mina inlägg till att endast ta upp de mer praktiska bitarna (konfig/labbar) inom Switching tillsvidare. Har läst klart "CCNP Switch - Official Certification Guide" så ska lägga större delen av tiden nu på att labba och skumma igenom  TSHOOT - O-C Guide inför kommande cert om 2-3 veckor. För att läsa mer om RSTP finns bl.a. följande sidor: [http://keepingitclassless.net/2013/07/ccie-spanning-tree-part-2-rstp/](http://keepingitclassless.net/2013/07/ccie-spanning-tree-part-2-rstp/) [http://blog.ine.com/wp-content/uploads/2011/11/understanding-stp-rstp-convergence.pdf](http://blog.ine.com/wp-content/uploads/2011/11/understanding-stp-rstp-convergence.pdf) [http://lostintransit.se/2013/08/08/rstp-synchronization-behind-the-scenes/](http://lostintransit.se/2013/08/08/rstp-synchronization-behind-the-scenes/)