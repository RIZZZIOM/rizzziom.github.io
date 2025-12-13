---
title: "Sakura"
date: 2025-12-13
draft: false
summary: "Writeup for Sakura CTF challenge on TryHackMe."
tags: ["osint"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/sakura/cover.webp"
  caption: "Sakura TryHackMe Challenge"
  alt: "Sakura cover"
platform: "TryHackMe"
---

Use a variety of OSINT techniques to solve this room created by the OSINT Dojo.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/sakura

## Tip off

The *OSINT Dojo* recently found themselves the victim of a cyber attack. Forensic analysis revealed an image that was left behind by the cybercriminals.

![](https://cdn.ziomsec.com/sakura/1.webp)

I downloaded the image locally and analyzed it using `exiftool` to find hidden metadata.

```
exiftool sakurapwnedletter.svg
```

![](https://cdn.ziomsec.com/sakura/2.webp)

The `Export-filename` revealed the username for the attacker.

*What username does the attacker go by?* 
→ SakuraSnowAngelAiko

## Reconnaissance

The attacker seems to have reused their username across other social media platforms as well. This can help reveal additional information about them especially if the username is reused on job hunting sites where users are more likely to provide real information. Our task is to find more information about the attacker based on the username revealed earlier. 

I did a google search on the username and found a github and X profile:
- https://github.com/sakurasnowangelaiko
- https://x.com/SakuraLoverAiko

The X profile revealed the full name of the attacker in one of the posts

![](https://cdn.ziomsec.com/sakura/3.webp)

Moving on to Github, I found a repository with PGP keys,

![](https://cdn.ziomsec.com/sakura/4.webp)

When I decoded this key, I found an email address.
- https://cirw.in/gpg-decoder/

![](https://cdn.ziomsec.com/sakura/5.webp)

*What is the attacker's full real name?*
→ Aiko Abe

*What is the full email address used by the attacker?*
→ SakuraSnowAngel83@protonmail.com

## Unveil

The instructions state that the cybercriminal is most likely aware that we are onto them. They have begun editing and deleting information in order to throw us off their trail. For this task, we need to trace some of the attacker's cryptocurrency transactions by diving deep into their Github.

I noticed repositories related to cryptocurrency and analyzed them:
- https://github.com/sakurasnowangelaiko/ETH
- https://github.com/sakurasnowangelaiko/cpuminer
- https://github.com/sakurasnowangelaiko/xmrig
- https://github.com/sakurasnowangelaiko/bitcoin
- https://github.com/sakurasnowangelaiko/ethminer

The first repository had 1 script that was updated. This is where I found something related to Ethereum

![](https://cdn.ziomsec.com/sakura/6.webp)

I did a google search with this string and found information about the wallet on https://etherscan.io

![](https://cdn.ziomsec.com/sakura/7.webp)

This is where I was able to view all the incoming and outgoing transactions of this wallet. This revealed the mining pool being used by the criminal and the type of cryptocurrency being used : Ethereum and Tether.

![](https://cdn.ziomsec.com/sakura/8.webp)

*What cryptocurrency does the attacker own a cryptocurrency wallet for?*
→ Ethereum

*What is the attacker's cryptocurrency wallet address?*
→ 0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef

*What mining pool did the attacker receive payments from on January 23, 2021 UTC?*
→ Ethermine

*What other cryptocurrency did the attacker exchange with using their cryptocurrency wallet?*
→ Tether

## Taunt

The criminal has left a message for us using an alternate account. We need to discover additional information by following the leads from the account to the Dark web and other platforms.

![](https://cdn.ziomsec.com/sakura/9.webp)

Since I'd already discovered both the twitter/X accounts of the attacker, I started analyzing the SakuraLoverAiko profile for more information and found some interesting posts that hinted towards where the criminal was heading.

**Clue 1**: An airport lounge for `Japan Airways`

![](https://cdn.ziomsec.com/sakura/10.webp)

**Clue 2**: Image of Japan

![](https://cdn.ziomsec.com/sakura/11.webp)

**Clue 3**: SSID of the Wifi

![](https://cdn.ziomsec.com/sakura/12.webp)

There was also a post made by the attacker where they hinted towards a dark web site:

```
Not too concerned about someone else finding them on the Dark Web.

Anyone who wants them will have to do a real DEEP search to find where I PASTEd them.
```

I searched for 'deep pasted' sites on tor and found http://depastedihrn3jtw.onion/ that matched the screenshot on twitter. Here, I pasted the md5 hash `0a5c6e136a98a60b8a21643ce8c15a74` and found the following details

```
Saving here so I do not forget

School WiFi Computer Lab: 			GTRI-Device		GTgfettt4422!@
Mcdonalds: 					Buffalo-G-19D0-1        Macdonalds2020
School WiFi: 					GTvisitor		GTFree123
City Free WiFi: 				HIROSAKI_Free_Wi-Fi 	H_Free934!
Home WiFi: 					DK1F-G			Fsdf324T@@
```

Since we required the BSSID, I navigated to wigle.net and searched the SSID shared by the criminal

![](https://cdn.ziomsec.com/sakura/13.webp)

*What is the attacker's current Twitter handle?*
→ SakuraLoverAiko

*What is the BSSID for the attacker's Home WiFi?*
→ 84:AF:EC:34:FC:F8

## Homebound

Since the target is heading home, we need to piece together their route back home using their X photos.

The attacker shared the following image before catching their flight to head home

![](https://cdn.ziomsec.com/sakura/14.webp)

In a distance, I was able to view the **Capitol** (located in Washington DC). I did a google search and found that the closest airport to this monument was Ronald Reagan Washington National Airport (DCA). To find the location of layover, I did a reverse image search of the image shared on twitter and found that it was taken in Haneda (HND)

![](https://cdn.ziomsec.com/sakura/10.webp)

To find the lake, I matched the image shared by the criminal with Google maps.
- https://goo.gl/maps/6ooEJXdu7FwoZ25z5

In the previous task, I found an interesting entry about a *City Wide* Wifi. This mostly likely was the city where the criminal lived
- http://depastedihrn3jtw.onion/show.php?md5=%7B0a5c6e136a98a60b8a21643ce8c15a74%7D

*What airport is closest to the location the attacker shared a photo from prior to getting on their flight?*
→ DCA

*What airport did the attacker have their last layover in?*
→ HND

*What lake can be seen in the map shared by the attacker as they were on their final flight home?*
→ Lake Inawashiro

*What city does the attacker likely consider "home"?*
→ HIROSAKI

That concludes my writeup for **Sakura**!
Until Next time :)

---