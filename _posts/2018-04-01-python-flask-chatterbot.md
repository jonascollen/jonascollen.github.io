---
title: Python & Flask - Chatterbot
date: 2018-04-01 17:31
comments: true
categories: [AI, Python]
tags: [chatterbot, flask]
---
![](/assets/images/2018/04/banner.png) 

Tänkte skriva ett litet inlägg om hur jag implementerade [Chatterbot](http://chatterbot.readthedocs.io/en/stable/) på min Flask/Python-webbsida, krävdes en hel del trial & error innan jag väl fick det att fungera så kanske kan vara intressant för någon mer. Ordförrådet fick dock hållas väldigt begränsat då Raspberryn inte riktigt orkar med om det blev för stora databaser att jobba med (svarstider på +20 sek).

![](/assets/images/2018/04/chatbot-1.png)

### Python

Börja med att installera Chatterbot-paketet, rekommenderar även att läsa igenom den officiella dokumentationen först för att förstå hur allt hänger ihop.

`pip install chatterbot`

Vi importerar sedan biblioteket i vår python-fil och skapar vår egen Chatbot-instans, här pekar vi även ut vilken databas vi ska använda, i mitt fall en Postgres-databas som ligger lokalt på servern:

```
from chatterbot import ChatBot
from chatterbot.trainers import ChatterBotCorpusTrainer

english_bot = ChatBot("jcAI", 
  storage_adapter="chatterbot.storage.SQLStorageAdapter",
  database_uri="postgres://user:password@localhost:5432/chatbot")
```

Nästa steg blir att "träna" vår chatbot, antingen via fördefinierade Corpus-filer som följer med vid installationen eller så skapar vi egna, i mitt fall gjorde jag både och. Filerna laddas härifrån:

```
**/www/flaskenv/lib/python3.5/site-packages/chatterbot_corpus/data**$ ls -l
totalt 68
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 bangla
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 chinese
drwxr-xr-x 2 joco02 joco02 4096 mar 9 22:58 custom
drwxr-xr-x 2 joco02 joco02 4096 mar 9 23:31 english
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 french
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 german
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 hebrew
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 hindi
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 indonesia
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 italian
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 marathi
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 portuguese
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 russian
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 spanish
drwxr-xr-x 2 joco02 joco02 4096 mar 21 15:00 swedish
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 tchinese
drwxr-xr-x 2 joco02 joco02 4096 mar 7 13:29 telugu
```

Några exempel från min egna Corpus-fil (sparad i custom), den inledande meningen/frågan börjar alltid med "- -" och efterföljande svar med "-":

```
- - Dagens lunch
  - Please use ex. !lunch KG13 or !lunch pannbiff
- - DHCP
  - DCHP-documents is found <a href="http://link.link/DHCP/">here.</a>
- - Edge
  - Edge & Core Workroom is found <a href="http://link.link/core_agg/">here.</a>
```

Vi pekar sedan på önskade Corpus-filer i scriptet (detta behövs endast köras en gång och kan sedan kommenteras bort, verifiera via apache2-loggen att allt gick bra).

```
english_bot.set_trainer(ChatterBotCorpusTrainer)
english_bot.train("chatterbot.corpus.swedish", "chatterbot.corpus.english", "chatterbot.corpus.custom")
```

Sedan återstår endast att presentera användarinput & vår Chatbots svar på en webbsida:

```
@app.route("/chatbot")
def chatbot():
  return render_template("chatbot.html")

@app.route("/get")
def get_bot_response():
  userText = request.args.get('msg')

  return str(english_bot.get_response(userText))
```

### HTML

Först skapar vi en "Chatbox", använder [Boostrap](https://getbootstrap.com/) för att höger/vänsterjustera texten mellan bot/användare samt färgval.

```html
<div class="row">
 <div class="col-md-6">
  <div class="panel panel-default">
   <div class="panel-heading"><span>AI Chat</span></div>
   <div class="panel-body">
    <p>Ask me something!<br></p>
    <div id="chatbox"></div>
   </div>
 <div class="panel-footer">
  <div id="userInput">
  <input id="textInput" type="text" name="msg" placeholder="">
  <input id="buttonInput" type="submit" value="Send" class="btn btn-success btn-xs">
  </div>
 </div>
 </div>
</div>
```

Sedan använder vi oss av ett script för att pusha/hämta info och presentera i ovanstående chatbox:

```javascript
<script>
 function getBotResponse() {
 var rawText = $("#textInput").val();
 var userHtml = '<p class="text-success text-right"><span>' + rawText + '</span></p>';
 $("#textInput").val("");
 $("#chatbox").append(userHtml);
 document.getElementById('userInput').scrollIntoView({block: 'start', behavior: 'smooth'});
 $.get("/get", { msg: rawText }).done(function(data) {
 var botHtml = '<p>' + data + '</p>';
 $("#chatbox").append(botHtml);
 document.getElementById('userInput').scrollIntoView({block: 'start', behavior: 'smooth'});
 });
 }
 $("#textInput").keypress(function(e) {
 if(e.which == 13) {
 getBotResponse();
 }
 });
 $("#buttonInput").click(function() {
 getBotResponse();
 })
 </script>
</div>
```

Kompletterande även med en liten "infobox" (tips på användbara kommandon etc):

```html
<div class="col-md-4">
 <div class="panel panel-default">
 <div class="panel-body">
  <h4>Some helpful commands:</h4><br>
  <small><strong>Jönköping:</strong></small><br>
  <small>!lunch <em>restaurant</em> (ex: <em><strong>!lunch KG13 or !lunch Vy</strong>)</em></small><br>
  <small>!lunch <em>food</em> (ex: <em><strong>!lunch kyckling</strong>)</em></small><br><br>

  <small><strong>Solna:</strong></small><br>
  <small>!solna <em>(shows todays menu at Modis)</em></small><br><br>

  <small><strong>Göteborg:</strong></small><br>
  <small>!gbg <em>(shows todays menu at Vällagat)</em></small><br><br>

  <small><strong>Other commands:</strong></small><br> 
  <small>!callguide (ex: <em><strong>!callguide or !callguide v12)</strong>)</em></small><br>
  <small>!sdp <em>node</em> (ex: <em><strong>!sdp j-sec1 or !sdp get-hsr1)</strong>)</em></small><br>
  <small>DHCP</small><br>
  <small>Common Services</small><br>
  <small>Standardporch</small><br>
  <small>etc..</small><br><br> 
  <small>I'm programmed with a few keywords so feel free to try anything, i'll try my best to respond. Let joco02 know if something else should be added for easy access.</small>
 </div>
 </div>
</div>
```

Som synes har jag byggt in ytterligare några funktioner här som använder samma chatbox men kringgår Chatterbot och presenterar resultat från andra scriptkörningar, exempelvis:

*   !lunch / !solna / !gbg - Använder Beautifulsoup4 för webscraping av menyer från lunchrestauranger i Jönköping/Solna/Göteborg (i närheten av Telia)
*   !callguide - Använder en schemafil och presenterar ansvarig person för aktuell/angiven vecka

Kanske återkommer med ett inlägg eller två om hur jag löste ovanstående framöver när det finns tid över.. :)