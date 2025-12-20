---
title: "Abusing Discord For C2"
date: 2025-12-20
draft: false
summary: "Creating Discord bot for encrypted command execution."
tags: ["c&c"]
categories: ["blog"]
series: ["lots"]
showToc: true
---

While researching phishing tradecraft, I came across the **[LOTS](https://lots-project.com/)** project and decided to explore it further by building a command-and-control (C&C) channel using **Discord**.
<!--more-->
**Note: This post is for educational research and defensive awareness. All techniques discussed should only be tested in environments you own or are explicitly authorized to assess.You are solely responsible for complying with applicable laws and guidelines.**

## Background

The goal of this project was to establish a C2 connection using trusted channels to prevent detections. The channels that were selected for this was:
- Discord - https://lots-project.com/site/646973636f72642e636f6d

Discord chatbots can be used to establish a C&C connection with a target. This is done using **webhooks**. A webhook is simply a URL that lets us send messages and files to our Discord channel. Discord bots use webhooks to perform actions that are specified to them.

To create this bot, I used **Go**:
1. **`discordgo`** → https://pkg.go.dev/github.com/bwmarrin/discordgo
2. Discord API Docs → https://discord.com/developers/docs/intro

When running these bots, all traffic is encrypted and is on verified discord channel so there is a high chance of it bypassing firewalls and achieve code execution.

## Bot Overview

For enough rights, I enabled developer mode by clicking on:

```
User Settings → Advanced → Developer Mode
```

![](https://cdn.ziomsec.com/discord-c2/1.webp)

Then, I visited https://discord.com/developers/applications/ to create a new application

```
New Application → Give it a name → Create → add pfp/description
```

![](https://cdn.ziomsec.com/discord-c2/2.webp)

In the **Bot** menu, I selected all `intents`

![](https://cdn.ziomsec.com/discord-c2/3.webp)

Finally, I clicked on **`Reset Token`** and saved the new token.

To add this bot on my server, I navigated to the `OAuth2` menu to find the `Client ID` information. This will be used to generate an invite link for the `Bot`.

The following options needs to be selected

```
Scopes → bot
Bot Permissions → Administrator
```

This will generate a **Guild URL** that can be opened in a new tab to select the server we want to add the bot to.

![](https://cdn.ziomsec.com/discord-c2/4.webp)

## Creating A Basic Prototype

The bot would work as follows:
- Once the RAT runs on the target, it listens for messages starting with `!exec`
- As soon as it receives a message, it executes whatever comes after `!exec` as a system shell command.
- It then sends the command output back into the Discord channel from where it received the message.

The code for the bot can be found on my **[Github](https://github.com/RIZZZIOM/phishfolio/blob/main/Echo/v1/)**.

After compiling the code and running it one the victim machine (Kali), I was able to execute shell commands through my private server.

![](https://cdn.ziomsec.com/discord-c2/5.webp)

### Forensics Issues

Once the commands where executed successfully, I deleted the server. However, this still leaves behind forensics evidence in the victim machine's cache. Discord stores a local cache using Chromium’s Simple Cache format. So there is a copy of attachments, emojis and webhooks in

```
%AppData%\discord\Cache\Cache_Data
```

Inside this directory, we will find:
- `index` – The Simple Cache index database
- `data_#` – Binary cache files, each containing multiple cached objects
- `f_######` – Extracted binary objects (images, attachments, etc.)

Apparently, the cached content often persists long after Discord messages have been deleted, so their timestamps can be lined up with the user activity. This also means we can match the file hashes against known databases like **VirusTotal** to know if it is a known malicious file.

> *reference: - https://www.pentestpartners.com/security-blog/discord-as-a-c2-and-the-cached-evidence-left-behind/*

## Dealing With Cache

To deal with this, I made 2 changes:
- Added option to delete or poison the discord cache
- Added a self destruct mechanism

I also added an option to switch shell types:
- `powershell` - default
- `cmd`
- `/bin/sh` - default
- `/bin/bash`

The code can be found on my **[Github](https://github.com/RIZZZIOM/phishfolio/blob/main/Echo/v2/)**.

```
!exec {COMMAND} → execute OS command
!shell {TYPE} → switch shells
!clearcache → delete Cache_Data, Code Cache, GPUCachhe folders from target
!poisoncache → clears the cache first then adds fake files with bogus data
!selfdestruct → spawns a seperate process and deletes itself
```

![](https://cdn.ziomsec.com/discord-c2/6.webp)

![](https://cdn.ziomsec.com/discord-c2/7.webp)

![](https://cdn.ziomsec.com/discord-c2/8.webp)

This **does not** make the bot **OPSEC** safe in itself. To make it suitable to evade detections it would have to generate traffic similar to a human. Some features that would make it OPSEC safe are:
- **Heartbeat packets** : Discord uses a WebSocket Gateway protocol. Discord clients maintain persistent connects and send regular *heartbeat* packets to keep the connection alive and prove they are still active.

- **Domain fronting via Discord CDN** : This is a technique where we make the TLS SNI and Host header different to hide our true destination.

```
# NORMAL HTTP REQUEST
client → [TLS SNI: discord.com] + [HTTP Host: discord.com] → Server

# DOMAIN FRONTING
client → [TLS SNI: cdn.discordapp.com] + [HTTP Host: attacker-domain.com] → CDN edge server → backend routes based on Host header
```

- **Staging** : Use different channels for command input and data output to avoid correlation.

## Detection

Detecting such RATs involves establishing a layered approach ([`DiD`](https://en.wikipedia.org/wiki/Defence_in_depth)).

1. WebSocket connects from non standard processes can be monitored. 
2. High frequency requests to CDNs from non browser processes should cause an alert.
3. From an endpoint detection point of view, shells spawning from unknown processes must be detected.
4. Anomalies in Discord cache such as mass deletion of files, cache files with unusual entropy most times point towards poisoning or evasions.
5. Watch out for process terminating immediately after spawning deletion commands
6. API calls outside business hours from endpoints or perfectly regular intervals between commands indicate bot behavior.

---
