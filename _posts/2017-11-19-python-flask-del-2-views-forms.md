---
title: Python & Flask del 2 - Views & Forms
date: 2017-11-19 14:19
comments: true
categories: [Flask, Python]
---
[![](/assets/images/2017/11/start.jpg)](/assets/images/2017/11/start.jpg) 

I del två tänkte jag bygga vidare på vår lilla "Hello World!"-sida och skapa ett formulär där våra användare kan fylla i vissa parametrar, detta skickas vidare till ett litet python-script som sedan presenterar resultatet för användaren. I detta exempel blir det konfiguration för driftsättning av ett nytt interface till en switch efter en fördefinierad konfigurationsmall. 

Vi börjar med att skapa vårat formulär, för detta behöver vi först en ny route i vår test.py-fil. En route matchar på adressen vi/apache skickar från webbläsaren, @app.route('/') matchar dvs på vår "root"-katalog, ex. min domän www.jonascollen.se/.  För att presentera en webbsida istället för en enkel string-variabel som "Hello World!" behöver vi först importera ytterligare en flask-modul, "render_template", och för att hämta data från formulären importerar vi "request". Vi skapar sedan får nya 'view' med namnet newconfig.

```python
from flask import Flask, render_template, request
app = Flask(__name__)

# route for default-paget
@app.route('/')
def hello_world():
    return 'Hello World!'

# route for new config-page
@app.route('/newconfig')
def newconfig():
    return render_template('newconfig.html')

if __name__ == '__main__':
    app.run()
```

Vi skapar sedan en ny html-fil under /templates-katalogen, newconfig.html med valfritt formulär. Rekommenderar starkt användandet av [Bootstrap](https://getbootstrap.com/) för att snygga till det hela, kommer dock inte lägga någon tid på att förklara html/css, där finns det vettigare sidor att vända sig till...  Mitt formulär ser ut enligt följande:

newconfig.html

```html

<div class="panel-heading"> <a data-toggle="collapse" href="#noBundle">New switch, no bundle</a></div>
 <div class="panel-body collapse in" id="noBundle">
 <form action="/newconfig" class="form-horizontal" method="POST" role="form">
 <fieldset>

<div class="form-group">
 <label class="col-md-1 control-label" for="swname">Switch:</label>
 <div class="col-md-2">
 <input id="swname" name="swname" class="form-control input-md" type="text">
 </div>
 </div>

<div class="form-group">
 <label class="col-md-1 control-label" for="swif">Interface:</label>
 <div class="col-md-2">
 <input id="swif" name="swif" class="form-control input-md" type="text">
 </div>
 </div>

<div class="form-group">
 <label class="col-md-1 control-label" for="swid">ID-number:</label>
 <div class="col-md-2">
 <input id="swid" name="swid" class="form-control input-md" type="text">
 </div>
 </div>

<div class="form-group">
 <label class="col-md-1 control-label" for="swbid">Bundle-ID:</label>
 <div class="col-md-2">
 <input id="swbid" name="swbid" class="form-control input-md" type="text">
 </div>
 </div>

<div class="form-group">
 <label class="col-md-1 control-label" for="userid">User:</label>
 <div class="col-md-2">
 <input id="userid" name="userid" class="form-control input-md" type="text">
 </div>
 </div>

<div class="form-group">
 <label class="col-md-1 control-label" for="send"></label>
 <div class="col-md-2">
 <button id="send" name="send" class="btn btn-info" type="submit">Generate config</button>
 </div>
 </div>
</fieldset>
 </form>
 </div>
 </div>
```

Det viktiga här är egentligen dessa rader:

*  `<form action="/newconfig" class="form-horizontal" method="POST" role="form"`> - Här matchar vi mot den route vi kommer definiera i vårat script för att presentera config-resultatet senare. Som synes går det att använda samma route som vi redan använder till vårat formulär, mer om detta strax.
*   `input id="x"` - Detta blir namnet på variablerna vi vill hämta från formuläret i vårat python-script.

[![](/assets/images/2017/11/form.jpg)](/assets/images/2017/11/form.jpg)

Surfar vi nu in på sidan /newconfig, fyller i fälten och klickar "Generate config" kommer du få ett felmeddelande, varför? När vi klickar send/generate config skickas formulärdatan med metoden 'POST', detta accepteras dock inte per default i vår /newconfig-route - och det vill vi ju inte heller. Vi vill fånga in datan och presentera det på en annan sida, låt oss därför skapa en ny route där vi explicit tilltåter just 'POST', hämtar in datan och spar ner till variabler.

```python
# route for switch config results
@app.route('/newconfig', methods=['POST'])
def sw_config():
    swname = request.form['swname']
    swif = request.form['swif']
    swid = request.form['swid']
    swbid = request.form['swbid']
    user = request.form['userid']
```

Har användaren fyllt i "Core-SW1" under "Switch:" kommer det skrivas ut om vi kör ex. print(swname). Allt bra så långt, vi behöver nu en konfigmall som vi sedan slår ihop med infon vår användare skickat in. Detta kan hämtas från vilken mapp som helst på servern, i detta exempel skapar jag en liten textfil direkt i ~.

```
nano sw-default.txt

interface Te$SWIF
 description To $SWNAME;$SWID;BE$SWBID;{work+in+progress+$USERID}
 bundle id $SWBID mode on
 no shut
 !
interface Bundle-Ether$SWBID
 description To $SWNAME;$SWID;BE$SWBID;;{work+in+progress+$USERID}
 bundle maximum-active links 1
 load-interval 30
```

Tillbaka till vår lilla python-fil. För att ersätta variablerna i vår konfigmall gjorde jag på detta vis:

# Spar ner våra variabler till en dictionary, här matchar vi $SWBID i vår konfigmall med swbid från användaren
`replacements = {'$SWBID':swbid, '$SWIF':swif, '$SWID':swid, '$SWNAME':swname, '$USERID':user}`

```python
# Läser in vår konfigmall och skapar en kopia med namnet "cfg-$swname-$user.txt" - W = Write
with open('/home/joco02/sw-default.txt') as infile, open('/home/joco02/cfg-%s-%s-%s.txt' % (swname, user), 'w') as outfile:
    for line in infile:
        # Byter ut orden som matchar med vår dictionary 'replacements'
        for src, target in replacements.items():
            line = line.replace(src, target)
        outfile.write(line)

# Läser in vår nya configfil till variabeln _swconfig_ - R = Read
with open('/home/joco02/cfg-%s-%s-%s.txt' % (swname, user), 'r') as f:
    swconfig = f.read()

# Skickar resultatet till results.html med tillhörande variabler: _content_ (innehåller allt konfig), samt _swname_ (variabel från användaren)
return render_template("results.html", content = _swconfig_, swname = _swname_)
```
Vi behöver nu bara en skapa en html-sida att presentera resultatet på, results.html. I mitt fall ser den ut enligt följande:

```html
<div class="panel panel-default">
 <div class="panel-heading"> <a data-toggle="collapse" href="#noBundle">Configuration for {{ swname }}</a></div>
 <div class="panel-body collapse in" id="noBundle">
 <pre> {{ content }} </pre>
 </div>
</div>
```

`{{ swname }}` kommer bytas ut mot namnet vår användare angav i formuläret, och content med innehållet från den konfigfil vi precis har skapat. Testar vi nu återigen vår sidan /newconfig och fyller i formuläret bör vår nya konfig presenteras när vi klickar på send.

[![](/assets/images/2017/11/configexblur.jpg)](/assets/images/2017/11/configexblur.jpg)

It works! :)
