---
title: "Kioptrix 3"
date: 2024-05-10
draft: false
summary: "Writeup for Kioptrix level 3 of the Kioptrix series challenge on VulnHub."
tags: ["kioptrix", "linux", "CMS", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/kioptrix3/cover.webp"
  caption: "Kioptrix 3 VulnHub Challenge"
  alt: "Kioptrix 3 cover"
platform: "VulnHub"
---

Kioptrix-3 is the third challenge in the Kioptrix series and similar to others, there’s more than one way to “pwn” this.
<!--more-->
To download Kioptrix 3, click 
- https://www.vulnhub.com/entry/kioptrix-level-12-3,24/

## Recon

After setting up the machine locally, I scanned the network to identify the target IP.

```bash
nmap -sn 192.168.1.0/24          
```

I then performed an **nmap** aggressive scan to find open ports, services and information about those services.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
****
![](https://cdn.ziomsec.com/kioptrix3/1.webp)

## Initial Foothold

### Gaining Shell

I accessed the web application using my browser.

![](https://cdn.ziomsec.com/kioptrix3/2.webp)

A redirection link on this page took me to another page that appeared incomplete.

![](https://cdn.ziomsec.com/kioptrix3/3.webp)

I clicked opened *developer's options* and navigated to the _Network_ tab. Upon refreshing the page, I found a list of requests being made to a particular domain.

![](https://cdn.ziomsec.com/kioptrix3/4.webp)

To resolve the domain, I mapped it to the target IP in the */etc/hosts* file.

![](https://cdn.ziomsec.com/kioptrix3/5.webp)

At first, I did not find anything useful on the web page. Upon returning back to the home page, I noticed a login panel that revealed the name of the CMS being used.

So - I looked for exploits related to the CMS (**Lotus CMS**) on Google and found several results. I tried the below exploit that I found on Github and got a reverse shell.
- https://github.com/Hood3dRob1n/LotusCMS-Exploit

```shell
git clone https://github.com/Hood3dRob1n/LotusCMS-Exploit; cd LotusCMS-Exploit
```

![](https://cdn.ziomsec.com/kioptrix3/6.webp)

I started a listener using **netcat** and executed the bash script.

```bash
rlwrap nc -lnvp 4444
# On another terminal
./lotsuRCE.sh TARGET
```

![](https://cdn.ziomsec.com/kioptrix3/7.webp)

![](https://cdn.ziomsec.com/kioptrix3/8.webp)

To streamline my activities, I exported my terminal and spawned a *pty* shell for ease of use.

```bash
export TERM=xterm
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Getting User Access

I navigated to the *home* directory and discovered two additional users.

```shell
cd /home
ls
```

![](https://cdn.ziomsec.com/kioptrix3/9.webp)

The *loneferret* user had a file containing an intriguing message.

```shell
cd loneferret; cat CompanyPolicy.README
```

![](https://cdn.ziomsec.com/kioptrix3/10.webp)

Executing this command would require a password, so I examined the running services and identified MySQL.

```shell
ps -aux | grep mysql
```

![](https://cdn.ziomsec.com/kioptrix3/11.webp)

It's in **safe** mode, so I would need another set of credentials to access it. Therefore, I searched for anything interesting in my own directory.

After searching around, I found a pair of credentials in `home/www/kioptrix3/gconfig.php`

![](https://cdn.ziomsec.com/kioptrix3/12.webp)

So I logged into mysql using these credentials.

```shell
mysql -u root -p
# ENTER PASSWORD
```

![](https://cdn.ziomsec.com/kioptrix3/13.webp)

I found the md5 hashed password of both *dreg* and *loneferret* in the `gallery` database.

![](https://cdn.ziomsec.com/kioptrix3/14.webp)

To crack these hashes, I visited **crackstation**.
- https://crackstation.net/

![](https://cdn.ziomsec.com/kioptrix3/15.webp)

| username   | password |
| ---------- | -------- |
| dreg       | Mast3r   |
| loneferret | starwars |
I then logged in as *loneferret* using **ssh**.

![](https://cdn.ziomsec.com/kioptrix3/16.webp)

## Privilege Escalation

I viewed the command history of *loneferret* and found **ht** being run as **sudo**.

![](https://cdn.ziomsec.com/kioptrix3/17.webp)

Hence I execute the command `sudo ht`

![](https://cdn.ziomsec.com/kioptrix3/18.webp)

I pressed **F3** and was presented with an option to open any file. Since I was running the software using sudo privileges, I captured the flag from `/root/Congrats.txt`.

![](https://cdn.ziomsec.com/kioptrix3/19.webp)

## Backdoor
### Using SSH keys

I generated SSH keys on my terminal.

```bash
ssh-keygen -t rsa -b 4096 -C "keyfork3"
```

I then copied the public key and stored it in **id_rsa.pub**.

![](https://cdn.ziomsec.com/kioptrix3/20.webp)

Finally, I pasted this into the *authorized_keys* file in the *root* directory.

> *authorized_keys* is a special file present in the **.ssh** folder that contains public keys that are allowed to log in as that particular user. All it requires is for us to log in using our private key.

```
ALT+F -> new -> text -> paste the ssh key -> save as -> /root/.ssh/authorized_keys
```

![](https://cdn.ziomsec.com/kioptrix3/21.webp)

### Using The  `/etc/passwd` File

I updated the **id** value in the `/etc/passwd` file of *loneferret* to **0**.

![](https://cdn.ziomsec.com/kioptrix3/22.webp)

Upon reconnecting as *loneferret*, I got root access.

### Using Sudoers File

I modified the permissions of *loneferret* in the `/etc/sudoers` file to allow all commands as sudo.

![](https://cdn.ziomsec.com/kioptrix3/23.webp)

I could then run any command as root.

![](https://cdn.ziomsec.com/kioptrix3/24.webp)
## Closure

Here's a comprehensive summary of my successful penetration of the **Kioptrix L3** system:
- I exploited the CMS to gain initial access.
- Discovered user credentials within the `gconfig.php` file.
- Utilized **ht** software with **sudo** privileges to locate the flag in the root directory.
- Implemented three distinct methods to establish a backdoor.

That concludes my walkthrough. Until next time! :)

---
