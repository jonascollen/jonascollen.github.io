---
layout: post
title: Python - Automatisera konfig med indata från Excel mha openpyxl
date: 2018-03-17 13:56
comments: true
categories: [Python, Scripting]
tags: [openpyxl]
---
Fortsätter på temat Python ett tag till då de böcker jag läst nu senast (Interconnections - Bridges, Routers, Switches and Internetworking Protocols samt TCP/IP Illustrated) inte direkt inspirerar till att skriva några inlägg om. Det är dock väldigt bra böcker med mycket matnyttig info som jag rekommenderar alla som satsar mot CCIE att läsa för att ge en bättre inblick i "the fundamentals". 

Över till Python, i detta lilla script tänkte jag ge ett exempel på hur vi kan inhämta data från ett excelblad och sedan skapa konfiguration utifrån fördefinierade templates. I mitt fall används excelbladet som ett orderunderlag som sedan kommer producera konfiguration till ASR9K för x antal switchar. Mallen jag använder ser ut enligt följande: [![](/assets/images/2018/03/bfswitch.jpg)](/assets/images/2018/03/bfswitch.jpg) Det vi frågar användaren efter är de "dynamiska" värdena vi behöver till våra konfig-templates. En mycket förkortad variant av en template jag använder idag skulle kunna se ut såhär:

```
!!!!!!!!!!!!! $BFNAME !!!!!!!!!!!!!

interface Te$BFINT
 description To $BFNAME;ETHAGG;$BFFB;;{LAG:BE$BID} 
 bundle id $BID mode on
 lacp period short
 load-interval 30
 transceiver permit pid all
 no shut
! 
interface Bundle-Ether$BID
 description To $BFNAME;ETHAGG;$BFFB;;
 bundle maximum-active links 1
 load-interval 30
 mtu 9216
!
multicast-routing
 address-family ipv4
 !
 interface Bundle-Ether$BID.243
 enable
!
router igmp
 interface Bundle-Ether$BID.243
 version 2
!
router pim
 address-family ipv4
 interface Bundle-Ether$BID.243
 enable
 !
```

Här har jag använt mig av ett antal variablar: _$BFNAME,  $BFINT, $BFFB & $BID_ men detta går ju att bygga vidare på i all oändlighet (förutom att det kommer ta längre tid att fylla i excel-bladet).. :) Jag använder mig fortfarande av flask som backend för att hämta in datan men detta kan ju göras på valfritt sett. I mitt fall ser HTML-koden ut enligt följande:

```html
<form action="/bfmulti" class="form-horizontal" method="POST" role="form" enctype="multipart/form-data">
 <fieldset>
  <div class="form-group">
   <blockquote class="blockquote">
   <p>Upload file for new BF-switches.</p><br>
   <img src="/static/images/bfswitch.jpg"><br>
   <footer class="blockquote-footer">File required to be in <cite title="Source Title">*.xlsx format.</cite></footer>
   </blockquote>
   <small class="form-text text-muted col-md-2">Download template <a href="/static/download/bygga_bf.xlsx">here.</a></small>
 </div>
 <label class="btn btn-default btn-file col-md-1">Select file:
  <input type="file" style="display: none;" accept="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" name="file"><br></label>
  <label class="control-label" for="send"></label>
  <button id="send" name="send" class="btn btn-success" type="submit">Send</button>
 </fieldset>
</form>
```
I python har jag använt mig av biblioteket "openpyxl" för att läsa in datan från excel. Först läser jag in filen från användaren och använder sedan openpyxl för att hämta data från det aktiva bladet (har endast ett i min excel-mall). Vi läser även in antal rader (max_row) så vi vet hur många gånger scriptet ska loopa. För att skapa unika filnamn använder jag mig av datetime.

```python
import openpyxl
from datetime import datetime
@app.route('/bfmulti', methods=['POST'])
def bf_multi():
 file = request.files['file']
 if file:

 wb = openpyxl.load_workbook(file)
 ws = wb.active
 maxRow = ws.max_row
 bfconfig_time = datetime.now().strftime("%Y%m%d-%H:%M:%S")
 filename = '/static/download/cfg-multi-%s.txt' % (bfconfig_time)
```
Vi loopar sedan alla rader vi hittat (maxRow) och läser in data från respektive cell. Observera att vi börjar på rad 3 i detta fall då vi ej vill loopa över våra rubriker etc:

```python
for row in range(3, maxRow):
  checkempty = ws['A' + str(row)].value
  if checkempty is not None:
    bfname = ws['A' + str(row)].value
    bffb = ws['B' + str(row)].value
    bfbid = ws['C' + str(row)].value
    bfif = ws['D' + str(row)].value
    bfif = bfif.replace("-", "/")
    link = ws['E' + str(row)].value
    kotype = ws['F' + str(row)].value

  if "10G" in str(link):
    linktype = "Te"
  elif "1G" in str(link):
    linktype = "Gi"
```

Då våra templates ser annorlunda ut beroende på om det är ett 1G- eller 10G-interface sätter vi linktype efter vad vi hittar under str(link). Vi skapar sedan en dictionary med de variablar vi läst in från excel-bladet och matchar dessa med de förutbestämda namnen i vår template-konfig enligt följande:

`replacements = {'$BID':str(bfbid), '$BFINT':str(bfif), '$BFFB':str(bffb), '$BFNAME':str(bfname)}`

Nu återstår det bara att läsa in vår template-konfig, appenda rad för rad och ersätta våra variablar med det vi hittat i excel.

```python
if linktype == "Gi":
  with open('/bfconfig/bf_1g_config.txt') as infile, open('/website/static/bfconfig/cfg-multi-%s.txt' % (bfconfig_time), 'a') as outfile:
    for line in infile:
      for src, target in replacements.items():
        line = line.replace(src, target)
        outfile.write(line)
```

När vi läst in alla rader återstår det endast att läsa in resultatet och antingen visa det för användaren eller kanske direkt pusha ut konfigurationen beroende på användingsområde?

```python
with open('/website/static/bfconfig/cfg-multi-%s.txt' % (bfconfig_time), 'r') as f:
 content = f.read()

return render_template("results_bfmulti.html", content = content, filename = filename)
 ```

Svårare än så är det inte! :)
