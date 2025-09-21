---
title: "Mr Robot"
date: 2024-07-15
draft: false
summary: "Writeup for Mr Robot CTF challenge on VulnHub."
tags: ["linux", "suid", "wordpress"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/mrrobot/cover.webp"
  caption: "Mr Robot VulnHub Challenge"
  alt: "Mr Robot cover"
platform: "VulnHub"
---

**Mr-Robot** is an intermediate level boot2root CTF challenge. It has three keys hidden in different locations. Our goal is to find all three. Each key is progressively difficult to find. 
<!--more-->
To download **Mr-Robot**, click on the link given below:
- https://www.vulnhub.com/entry/mr-robot-1,151/

## Recon

I performed a network scan to find the target IP.

```bash
nmap -sn 192.168.1.0/24      
```

Following this, I conduct an **nmap** aggressive scan to gather information about the active ports and services.

```shell
nmap -A -p- TARGET --min-rate 10000
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 443      | https       |

![](https://cdn.ziomsec.com/mrrobot/1.webp)

## Initial Foothold

The **nmap** scan revealed that ports **80** and **443** are active. I proceeded to access the target using a web browser for further investigation.

![](https://cdn.ziomsec.com/mrrobot/2.webp)

After interacting with the web server by entering various commands, I didn't find anything significant. Therefore, I conducted a **ffuf** scan on the target to gather information about directories and files.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 200,302
```

![](https://cdn.ziomsec.com/mrrobot/3.webp)

I discovered several interesting files such as _readme_, _wp-login_, and _robots.txt_. 

### Capturing Flag 1

Upon visiting _robots.txt_, I uncovered the first key.

![](https://cdn.ziomsec.com/mrrobot/4.webp)

![](https://cdn.ziomsec.com/mrrobot/5.webp)

### Enumerating Wordpress Credentials

The _robots.txt_ file contained a reference to another file named _fsocity.dic_. When I tried accessing  this file, it automatically downloaded to my system. Upon inspection, I discovered that it was a wordlist.

```shell
$ head fsocity.dic
$ wc -l fsocity.dic # to find the number of words by counting the lines
```

![](https://cdn.ziomsec.com/mrrobot/6.webp)

![](https://cdn.ziomsec.com/mrrobot/7.webp)

Since the wordlist could have contained duplicate words, I filtered out the unique words and saved them in a new file named _newfsocity.dic_.

```shell
cat fsocity.dic | sort -u > newfsocity.dic
```

![](https://cdn.ziomsec.com/mrrobot/8.webp)

During my **ffuf** scan, I also came across a WordPress login page located at _/wp-login_.

![](https://cdn.ziomsec.com/mrrobot/9.webp)

By trying default username and password combinations, I observed that the error message specified which field was incorrect. Leveraging this logic flaw, I used the wordlist accessed earlier to determine a valid username.

![](https://cdn.ziomsec.com/mrrobot/10.webp)

I used Burp Suite to enumerate the username, following these steps:

> **Note:** Since the wordlist is extensive, you'll need Burp Suite Pro to perform brute force attacks. If you're using the community edition, an alternative method is discussed after the Burp Suite method.

1. I inputted a default username and password, then monitored the requests sent by the server.

![](https://cdn.ziomsec.com/mrrobot/11.webp)

I discovered that the username is transmitted via the variable _log_, and the password is sent through _pwd_.

2. I forwarded this request to _Intruder_ and included the username field in the scope for the attack.

![](https://cdn.ziomsec.com/mrrobot/12.webp)

3. I pasted the wordlist into the _Payloads_ sub-tab.

![](https://cdn.ziomsec.com/mrrobot/13.webp)

4. I navigated to the _Settings_ sub-tab to extract the error message received.

![](https://cdn.ziomsec.com/mrrobot/14.webp)

5. I started the attack.

![](https://cdn.ziomsec.com/mrrobot/15.webp)

These usernames elicited a different response, indicating their validity.

An alternate to perform this attack would be using **hydra**.

> Reference: https://www.geeksforgeeks.org/crack-web-based-login-page-with-hydra-in-kali-linux/

1. I attempted to login and viewed the requests made by the server through the *http-history* sub tab in **Burp Suite**.

![](https://cdn.ziomsec.com/mrrobot/16.webp)

2. The fields used to send the username and password are _log_ and _pwd_, respectively. Additionally, the response I receive is _: Invalid username_. 
3. With this information, I could utilize **hydra** to perform a brute force attack on the login panel.

```bash
hydra -L <username_list> -p admin <target> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:Invalid"
```

![](https://cdn.ziomsec.com/mrrobot/17.webp)

Now that I have the username, I can attempt to crack the password using the same wordlist. Given that it's a WordPress site, I used **wp-scan** for brute forcing the correct password. 

```bash
wpscan --url http://TARGET/wp-login.php -U elliot -P newfsocity.dic
```

![](https://cdn.ziomsec.com/mrrobot/18.webp)

Now that I had the username and password, I logged into the website.

![](https://cdn.ziomsec.com/mrrobot/19.webp)

### Getting Reverse Shell

The _Appearance_ tab offered options to customize various aspects of the web server's appearance. Navigating to the _Editor_ sub-tab, I found templates for various response types.

![](https://cdn.ziomsec.com/mrrobot/20.webp)

To test it out, I navigated to the _404 template_ and added the following HTML code:

```html
<p>HELLO WORLD!!</p>
```

![](https://cdn.ziomsec.com/mrrobot/21.webp)

Then to trigger this template, I tried accessing a page that did not exist.

![](https://cdn.ziomsec.com/mrrobot/22.webp)

My query was executed successfully. Next, I navigated to [**revshells.com**](https://www.revshells.com/) and selected a payload for a reverse shell. Since the site was running on PHP, I chose the _php pentestmonkey_ script.

I deleted the existing code from the *404 template* and pasted the reverse shell payload.

![](https://cdn.ziomsec.com/mrrobot/23.webp)

Then I started a listener using **nc** and triggered the *404 template*.

```bash
rlwrap nc -lnvp 8080
```

![](https://cdn.ziomsec.com/mrrobot/24.webp)

![](https://cdn.ziomsec.com/mrrobot/25.webp)

This granted me a reverse shell. Navigating to the home directory, I discovered another user named _robot_.

![](https://cdn.ziomsec.com/mrrobot/26.webp)

I found the second flag in this folder, but I didn't have permission to read it.

![](https://cdn.ziomsec.com/mrrobot/27.webp)

### Capturing Flag 2

I checked the file permissions using the **ls** command.

```shell
ls -la
```

![](https://cdn.ziomsec.com/mrrobot/28.webp)

Since I had read permission for the second file, I read it and found an MD5 hash of the _robot_ user's password.

```shell
cat password.raw-md5
```

![](https://cdn.ziomsec.com/mrrobot/29.webp)

I cracked the hash using **Crackstation**'s online hash cracker.
- https://crackstation.net/

![](https://cdn.ziomsec.com/mrrobot/30.webp)

I then tried switching to *robot* but couldn't.

![](https://cdn.ziomsec.com/mrrobot/31.webp)

I needed to spawn a TTY shell. I found a Python script online from this [article](https://sushant747.gitbooks.io/total-oscp-guide/content/spawning_shells.html) and used it to spawn a TTY shell. Then, I switched to the _robot_ user.

```shell
$ python -c 'import pty;pty.spawn("/bin/bash")'
$ su robot
```

![](https://cdn.ziomsec.com/mrrobot/32.webp)

Then I read the 2nd flag

![](https://cdn.ziomsec.com/mrrobot/33.webp)

## Privilege Escalation

To escalate my privileges, I identified binaries with **SUID** bits that were owned by the root user.

```bash
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/mrrobot/34.webp)

the **nmap** file had an suid bit which could be exploited. I referred to **GTFOBins** to exploit this flaw and gain root access.
- https://gtfobins.github.io/

![](https://cdn.ziomsec.com/mrrobot/35.webp)

I then used this to spawn an interactive shell as root.

```shell
nmap --interactive
```

![](https://cdn.ziomsec.com/mrrobot/36.webp)

I accessed privileged mode from Bash using **bash -p**.

```shell
!bash -p
```

![](https://cdn.ziomsec.com/mrrobot/37.webp)

Finally, I captured the third flag from the *root* directory.

![](https://cdn.ziomsec.com/mrrobot/38.webp)

## CLOSURE

I located all the keys in the following locations:
1. The first key was found in the _robots.txt_ file.
2. I used the provided wordlist to get valid credentials and then a reverse shell from the 404 template.
3. The second key was discovered in the home directory under the _robot_ user.
4. I exploited the SUID bit set on **nmap** to gain root access.
5. The final key was captured in the root directory.

That's it from my side! Happy Hacking :)

---
