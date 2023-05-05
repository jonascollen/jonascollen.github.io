---
title: E-Bike conversion - Nishiki Pro Air BBSHD
date: 2019-05-30 22:34
author: Jonas Collén
comments: true
categories: [Projects, Other]
tags: bbshd, em3ev, Electric bike
---
Something as unusual as a non-networking post, but it was a fun project so thought I could do a short post about it. We're soon moving a bit further out of town to a house so i've been eyeing an eletric bike for a while, but in the end decided to build my own as the bikes for sale here in Sweden is pretty bland - and tinkering with stuff is fun... :)

![](/assets/images/2019/05/Pro-Air-Herr-3396.png)

The bike I used is a 2016 Nishiki Pro Air which has a BSA 68mm bottom-bracket thread which is required to mount Bafangs middrive motors which I went with. For wider bottom brackets there are a few alternatives going up to 120mm, but BSA threading is required however. Electricbike.com has a great [guide](https://electricbike.com/forum/forum/knowledge-base/motors-and-kits/bbshd/10736-how-to-order-the-right-bbsxx-kit) on this, very import to get this right or you will be stuck with a motor that doesn't fit.

![](/assets/images/2019/05/IMG_3825.jpg)

I ordered a Bafang BBSHD 1000W (~1500W Max), Jumbo Shark Battery 52v 14.8Ah HG2 (~44A Continous/55A Max discharge) and a 46t Lekkie Blingring. This is **ooof cooourse** limited to 250W & 25km/h however to keep it road legal here in Sweden. :) But i've heard rumors that you can put a throttle on it and it will do ~60km/h even uphill with current gearing and some light pedaling, so it pretty much turns in to a rocket.

![](/assets/images/2019/05/58463123_2017575868552001_7226363293919608832_n.jpg)

The installation was a breeze after getting the old bottom bracket out, hardest part was probably routing all the extra cables and making it look nice. I easily doubled the weight of the bike however.. The motor must weigh in around ~6kg + battery ~3-4kg, it's very very heavy so let's hope I never run out of juice and have to pedal back home uphill... :)

![](/assets/images/2019/05/q904u487o5131.jpg)

All finished up! Been using it for a month now and it's crazy fun, the performance you get is so far beyond the ridiculously expensive pre built bikes - which still usually have cheap components and limited to 25km/h & 250/350w anyway.

The BBSHD has a pretty weird pedal assist however due to missing a torque sensor, so it will basically work less the harder you pedal, totally opposite of other electric bikes which I didn't really like.

Not that I have, but a way to fix this is installing a throttle instead and do some reprogramming of the controller unit. There's cheap cables to buy on ebay or you can build one yourself, the software is free however. I really recommend reading [this page](https://electricbike-blog.com/2015/06/26/a-hackers-guide-to-programming-the-bbs02/) to learn more.

Some settings you could use is for example:

_PAS 0 - I=100%, S= 0%  
PAS 1 - I= 50%, S= 72%  
PAS 2 - I= 71%, S= 72%  
PAS 3 - I=100%, S=100%_

_PAS designated assist level = 0  
Throttle designated assist level = By Display's Command  
Set throttle to "current" and start current to 1% for super smooth power delivery._

This will give three power levels for the throttle, 0 = off, 3 = max fun and turn off the annoying PAS. :) I ordered my kit from Paul over at www.em3ev.com based in China which has excellent reviews and I highly recommend them myself, they build their batteries on order and can't be compared to the stuff you get from aliexpress etc. Price is higher though but you get what you pay for!