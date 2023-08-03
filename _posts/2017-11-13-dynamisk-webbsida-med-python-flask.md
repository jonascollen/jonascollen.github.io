---
title: Python & Flask del 1 - Dynamisk webbsida
date: 2017-11-13 21:28
comments: true
categories: [Flask, Python]
---
1022 dagar sedan senaste inlägget så kändes som det kanske var dags att ge den här sidan lite kärlek igen.. Liten uppdatering - inget CCIE-cert ännu, däremot nytt jobb som IP Specialist på Telia sedan ~1 år tillbaka. Med SDN på intåg samt att certifieringar i det stora hela börjar kännas allt mer icke-relevanta pga braindumps m.m. så vet jag inte riktigt hur jag ska göra framöver. Just nu är det mer lärande för lärandets skull.. :) 

Har haft som sidoprojekt senaste tiden att automatisera arbetsuppgifter/-moment via (shell)script, men tänkte försöka gå över till att använda Python istället i samband med lansering av en ny scriptserver på jobbet. Så i väntan på lanseringen har jag börjat kika lite på små pythonscript, men hade även en idé att försöka lyfta ur vissa bef. script som inte kräver någon nätaccess till en webbserver istället (och samtidigt skriva dessa i python istället) för att göra det mer användarvänligt/lättillgängligt. 

![logo-full](/assets/images/2017/11/logo-full.png)
Då jag ville använde mig av Python3 som "backend" verkade [Flask](http://flask.pocoo.org/) intressant, webbservern driver jag just nu på en raspberry (rasbian)/apache2. Efter x antal timmar börjar det nu faktiskt lira helt ok så tänkte det kunde vara på sin plats att dokumentera ner det hela lite grann. För att komma igång:

*   Installera [Python](http://python.org/download/)
*   Installera [Virtual Enviroment](http://pypi.python.org/pypi/virtualenv) (ej ett krav men rekommenderat)
*   Installera [Apache2](https://httpd.apache.org/)

Skapar sedan en katalog för min "hemsida" under /var/www/html/ (default för debian):

```bash
$ mkdir website
# För bilder/css/js etc
$ mkdir website/static
# För html-filer
$ mkdir website/templates
$ cd website
# Skapar en Virtual enviroment för Py3 med namnet flaskenv
$ virtualenv --python=python3 flaskenv
\# Aktivera VE
$ source flaskenv/bin/activate
```
Fungerar allt som det ska bör prompten se ut likt detta (min prompt ser kanske lite udda ut men är endast för jag skapat en symlänk mellan hemkatalogen & /var/www/html/):

```bash
(flaskenv) joco02@webdev:~/www/website$
(flaskenv) joco02@webdev:~/www/website$ python -V
Python 3.5.3
# Installera Flask:
(flaskenv) joco02@webdev:~/www/website$ pip install flask
```
Vi kan nu testa slänga ihop ett litet script:

```python
(flaskenv) joco02@webdev:~/www/website$ nano test.py

from flask import Flask
app = Flask(\_\_name\_\_)

@app.route('/')
def hello\_world():
 return 'Hello World!'

if \_\_name\_\_ == '\_\_main\_\_':
 app.run()
```

Starta sedan Flasks inbyggda webbserver med:

```bash
(flaskenv) joco02@webdev:~/www/website$  python test.py
 \* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

I default-läget är dock Flasks webbserver endast nåbar från localhost, vi kan alltid verifiera mer curl om det fungerar men mer lämpligt är väl att öppna upp så du kan testa från andra enheter:

```bash
(flaskenv) joco02@webdev:~/www/website$ export LC_ALL=C.UTF-8
(flaskenv) joco02@webdev:~/www/website$ export LANG=C.UTF-8
(flaskenv) joco02@webdev:~/www/website$ export FLASK_APP=test.py
(flaskenv) joco02@webdev:~/www/website$ flask run --host=0.0.0.0
 \* Serving Flask app "test"
 \* Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)

# Alt. så justerar vi vår test.py enligt följande:
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000, debug=True)
```
Surfar vi nu in till vår server på port 5000 bör du mötas av en fin "Hello World!". Men för att få det att fungera med apache2 krävs det ytterligare en hel del jobb. Vi behöver först installera WSGI för Python3:

```bash
(flaskenv) joco02@webdev:~/www/website$ deactivate
joco02@webdev:~/www/website$ sudo apt-get install libapache2-mod-wsgi-py3
joco02@webdev:~/www/website$ # Aktivera mod\_wsgi och starta om apache2
joco02@webdev:~/www/website$ a2enmod wsgi 
joco02@webdev:~/www/website$ sudo /etc/init.d/apache2 restart
```
Skapa sedan en Virtual Host i Apache2:

`joco02@webdev:/var/www/html/website$ sudo nano /etc/apache2/sites-available/website.conf`

Min fil ser ut enligt följande (observera att WSGIProcessGroup måste matcha din WSGI-fil du skapar i nästa steg):

```
<VirtualHost \*:80>
 WSGIDaemonProcess website user=xx group=xx threads=5
 WSGIScriptAlias / /var/www/html/website/website.wsgi

ServerName x.se
 ServerAdmin x@x
 <Directory /var/www/html/website/>
 WSGIProcessGroup website
 WSGIApplicationGroup %{GLOBAL}
 WSGIScriptReloading On

Require all granted

</Directory>
 Alias /static /var/www/html/website/static
 <Directory /var/www/html/website/static/>
 Order allow,deny
 Allow from all
 </Directory>
 ErrorLog ${APACHE\_LOG\_DIR}/error.log
 LogLevel warn
 CustomLog ${APACHE\_LOG\_DIR}/access.log combined
</VirtualHost>
```

Vi akviterar den sedan med:

`sudo a2ensite website`

Vi närmar oss.. Nu behövs en WSGI-fil som vi skapar i vår "website"-mapp. Efter en hel del trixande/trial & error ser min fil tillslut ut enligt följande:

```bash
joco02@webdev:/etc/apache2/sites-available$ cd /var/www/html/website/
joco02@webdev:/var/www/html/website$ nano website.wsgi
```

```python
#!/usr/bin/python
import sys
import logging
import site

site.addsitedir('/var/www/website/flaskenv/lib/python3.5/site-packages')

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/html/website/")

from test import app as application
```

Starta sedan om apache och du ska *förhoppningsvis* få upp din hemsida om du surfar in på http://\*serverip\*, det var en del svett, blod & tårar innan jag kom såhär långt själv... :) Kommer ytterligare något inlägg om hur vi bygger vidare på applikationen framöver, min egna lilla server rullar fortfarande på: [www.jonascollen.se](http://www.jonascollen.se).
