---
title: "Hacking Fundamentals - Scanning"
date: 2026-09-25
draft: false
summary: "Understanding Scanning"
tags: ["Hacking Fundamentals"]
categories: ["blog"]
series: ["Hacking Fundamentals"]
showToc: true
author: "Moiz Bootwala"
---

Let's start the second phase of hacking by actively probing the target.
<!--more-->
This is **Part 1 of Phase 2: Scanning and Enumeration**. In the reconnaissance post, we built a picture of what might belong to the target using mostly public information. Scanning is where we begin validating that picture: which systems are alive, which ports are reachable, and which services are actually listening.

Part 2 will pick up from the results we collect here and cover **enumeration** (interacting with those services to extract more useful information).

In this blog, we'll cover:
- what scanning is and where it fits
- discovering live systems
- understanding port states
- TCP and UDP scanning with Nmap
- service, version and OS detection
- banner grabbing

This is where we move onto the next phase of hacking - scanning
 
## What Is Scanning

**Scanning** is the process of discovering systems on a network and examining their ports, applications and services. It can be divided into two closely related activities:
- A **network scan** looks across a network to find connected systems and their IP addresses.
- A **host scan** focuses on one system to find its open ports, services, applications and operating system.

Usually, when we are part of a network, we scan its range to identify live hosts. The once these hosts have been identified, we scan them to get open ports. Then we identify what services are running on these ports and their version. Finally, we scan for identifying os information, service banners and other such information that the host may reveal within a network through a port.

The order matters. Scanning every port on every possible address is wasteful when you can first narrow the range down to systems that respond. However, do note that a host that does not answer a discovery probe is not necessarily offline as firewalls can block those probes. Scanning is less about trusting one command and more about making sense of several responses.

## Discovering Live Systems

The first question is whether any systems are active in the target range. ICMP is the easiest way to check: a host receives an Echo Request and may return an Echo Reply. A **ping sweep** repeats that process across multiple addresses.

Nmap performs host discovery without a port scan using `-sn`:

```bash
nmap -sn NETWORK_ID/CIDR
```

![discovering live systems with network ping sweep](https://cdn.ziomsec.com/hacking-fundamentals-scanning/1.webp)

This gives us a smaller list of live hosts to investigate. Nmap performs discovery before a port scan by default, which creates one important problem: some live systems block ICMP or other discovery probes. If a target is known to be live but Nmap treats it as down, `-Pn` skips discovery and proceeds as if the host is up:

```bash
nmap -Pn TARGET_IP
```

On a local network, **Netdiscover** gives us another option. It uses ARP broadcasts at Layer 2 to ask for the MAC address associated with each IP:

```bash
netdiscover -r NETWORK_ID/CIDR
```

To scan through a particular network adapter:

```bash
netdiscover -i eth0
```

The result of this stage should be a list of IP addresses, not a conclusion about what each system does.

## Understanding Port States

A port is a virtual connection point used to identify a service or process on a system. When Nmap probes one, it assigns a state based on the response:

| **State**              | **What It Means**                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------- |
| **Open**               | A service is listening on the port.                                                               |
| **Closed**             | No service is listening, but the port is reachable and not blocked by a firewall.                 |
| **Filtered**           | Nmap cannot determine the state because a firewall or another security device blocks the traffic. |
| **Unfiltered**         | The port is reachable, but the scan cannot determine whether it is open or closed.                |
| **Open \| Filtered**   | Nmap cannot decide whether the port is open or filtered.                                          |
| **Closed \| Filtered** | Nmap cannot decide whether the port is closed or filtered.                                        |

For a beginner, **open**, **closed** and **filtered** are the three results to understand first. An open port gives us a service to investigate. A closed port confirms that the host is reachable but nothing is listening there. A filtered port tells us that something between us and the port is interfering with the probe.

## TCP Scanning With Nmap

In the introduction to this series, we watched the TCP three-way handshake in Wireshark:

```plaintext
SYN → SYN-ACK → ACK
```

Port scanning uses the same behaviour. If we send a `SYN` to a TCP port:
- a `SYN-ACK` indicates that the port is open
- a `RST` indicates that the port is closed
- no useful response may mean the traffic is being filtered

### SYN Scan Vs TCP Connect Scan

The two TCP scans worth understanding first are:

| **Scan**            | **Switch** | **Behaviour**                                                                                   |
| ------------------- | ---------- | ----------------------------------------------------------------------------------------------- |
| **SYN scan**        | `-sS`      | Starts the handshake. When a `SYN-ACK` confirms an open port, Nmap sends a `RST` instead of completing it. |
| **TCP connect scan**| `-sT`      | Completes the full TCP handshake and immediately closes the connection.                         |

A SYN scan is commonly called a *stealth* or *half-open* scan because it does not complete the connection:

```bash
sudo nmap -sS TARGET_IP
```

![nmap stealth scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/2.webp)
> The `source`, `destination` and `info` tabs show the packets exchanged between the hosts during this scan. “Stealth” does not mean invisible. A SYN scan can still be detected by an IDS. It only describes how the TCP connection is handled.

A TCP connect scan is reliable and does not require the same special privileges, but it is more easily detected and consumes more resources:

```bash
nmap -sT TARGET_IP
```

![nmap tcp connect scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/3.webp)

> The `source`, `destination` and `info` tabs show the packets exchanged between the hosts during this scan.

### Scanning Every TCP Port

Nmap lets us scan a single port, a range, a set of common ports or all **65,535** ports:

```bash
nmap TARGET_IP -p PORT
```

![scanning a single tcp port](https://cdn.ziomsec.com/hacking-fundamentals-scanning/4.webp)

```bash
nmap TARGET_IP -p START-STOP
```

![scanning a tcp port range](https://cdn.ziomsec.com/hacking-fundamentals-scanning/5.webp)

```bash
nmap TARGET_IP --top-ports NUMBER
```

![scanning top tcp ports](https://cdn.ziomsec.com/hacking-fundamentals-scanning/6.webp)

```bash
nmap TARGET_IP -p-
```

![scanning all tcp ports](https://cdn.ziomsec.com/hacking-fundamentals-scanning/7.webp)

A useful first pass against one host is:

```bash
sudo nmap -sS -p- -T4 -oA tcp-all TARGET_IP
```

- `-sS` performs the SYN scan.
- `-p-` checks every TCP port.
- `-T4` uses the aggressive timing template.
- `-oA tcp-all` saves the result in normal, XML and grepable formats.

Saving the output matters. The open ports from this pass become the input for the more focused scan that follows, and the files give us something to revisit when the engagement reaches reporting.

## Service, Version And OS Detection

An open port number is only the beginning. We next want the application behind it, its version, and clues about the operating system.

Suppose the first pass returns ports `21`, `22`, `80` and `445`. Instead of repeating every port, we can concentrate the noisier checks on those results:

```bash
sudo nmap -sV -sC -O -p 21,22,80,445 -oA services TARGET_IP
```

- `-sV` attempts to identify the service and its version.
- `-sC` runs Nmap's default NSE scripts.
- `-O` uses TCP/IP stack fingerprinting for operating-system detection.
- `-p` limits the scan to the ports we already found.

![scanning service version, running default scripts and performing os scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/8.webp)

![scanning service version, running default scripts and performing os scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/9.webp)

Nmap also provides `-A`, an aggressive scan that combines port scanning, service detection, OS detection, script scanning and traceroute:

```bash
nmap -A -p 21,22,80,445 TARGET_IP
```

> It is convenient, but knowing the individual switches makes the scan easier to control. You can decide which checks are useful instead of treating `-A` as a magic “scan everything” button.

![aggressive scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/10.webp)

![aggressive scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/11.webp)

## UDP Scanning

UDP does not establish a connection or use TCP flags. There is no handshake, acknowledgment or teardown sequence, which makes its scan results less definite.

A basic UDP scan is:

```bash
sudo nmap -sU TARGET_IP
```

We can focus on specific services such as DNS and NTP:

```bash
sudo nmap -sU -p U:53,123 TARGET_IP
```

![nmap udp scan](https://cdn.ziomsec.com/hacking-fundamentals-scanning/12.webp)

Or scan selected TCP and UDP ports together:

```bash
sudo nmap -sS -sU -p T:80,443,U:53,123 TARGET_IP
```

![scanning tcp and udp services together](https://cdn.ziomsec.com/hacking-fundamentals-scanning/13.webp)

UDP responses are interpreted differently:

| **Response**                                    | **State**             |
| ----------------------------------------------- | --------------------- |
| A UDP response comes back from the target port  | **Open**              |
| ICMP Type 3, Code 3 (port unreachable)          | **Closed**            |
| Another ICMP unreachable response is returned   | **Filtered**          |
| Nothing comes back, even after retransmissions  | **Open \| Filtered**  |

The last result is the important one. Silence does not tell Nmap whether a UDP service accepted the packet or a firewall discarded it, so UDP scans can be slower and less reliable than TCP scans.

## Nmap Scripting Engine

The **Nmap Scripting Engine (NSE)** extends Nmap with scripts for discovery, service enumeration and vulnerability detection. The scripts are written in Lua and are normally stored in:

```plaintext
/usr/share/nmap/scripts/
```

The default set runs with `-sC`, while a specific script can be selected with `--script`:

```bash
nmap -sC TARGET_IP
nmap --script=banner TARGET_IP
```

![running banner scripts](https://cdn.ziomsec.com/hacking-fundamentals-scanning/14.webp)

You can locate scripts for a service from the scripts directory:

```bash
locate *.nse | grep <service-name>
```

![listing ftp scripts](https://cdn.ziomsec.com/hacking-fundamentals-scanning/15.webp)

NSE is also where scanning begins to turn into enumeration. A generic port scan discovers that SMB is listening; an SMB-specific script can then ask that service for SMB-specific information. We will use that approach throughout Part 2.

## Banner Grabbing

Many services identify themselves when a client connects. **Banner grabbing** collects that response to learn about the service or operating system. Nmap's `banner` script is one option, but **Netcat** makes the interaction much easier to see.

To connect to a service:

```bash
nc <TARGET_IP> <PORT>
```

For a web server on port 80, we can manually send an HTTP request:

```http
nc 192.168.1.10 80
GET / HTTP/1.1
Host: netcat

```

![grabbing ftp and http banner](https://cdn.ziomsec.com/hacking-fundamentals-scanning/16.webp)

The response can reveal the web server and other useful headers. The point of doing this manually is not speed—Nmap can collect banners across many ports; but understanding that enumeration tools are often automating ordinary conversations with a service.

## Scan Speeds

Nmap's timing templates range from `-T0` (paranoid) to `-T5` (insane), with `-T3` used by default. Slower templates reduce bandwidth use, while faster templates finish sooner but are more likely to be detected or blocked.

More precise controls include:
- `--scan-delay` to add time between probes
- `--max-retries` to control retransmissions
- `--min-rate` and `--max-rate` to control the packet rate
- `--host-timeout` to stop spending time on one host

## Lab

> For this lab, set up an intentionally vulnerable machine locally called **[metasploitable-2](https://vulnhub.com/entry/metasploitable-2,29/)**.

To practice everything we've learnt, you can perform the following:
**1. Find live hosts on your network and identify the ip belonging to the target**

```
nmap -sn network-id/cidr
```

**2. Scan every TCP port on one discovered host**

```
sudo nmap -sS -p- -T4 -oA tcp-all TARGET_IP
```

**3. Run detailed checks only against the ports that were open**

```
sudo nmap -sV -sC -O -p 21,22,80,445 -oA services TARGET_IP
```

**4. Check relevant UDP services**

```
sudo nmap -sU -p U:53,123,161 -oA udp-services TARGET_IP
```

**5. Manually verify an interesting banner** : 

```
nc TARGET_IP PORT
```

## Conclusion

That covers scanning at a practical level: discovering live systems, interpreting port states, comparing SYN and TCP connect scans, checking all TCP ports, following up with service/version and OS detection, handling the uncertainty of UDP, using NSE scripts, and manually grabbing a banner with Netcat.

The output is still only a map. Knowing that port 21 runs FTP, port 445 runs SMB or port 80 serves a website does not tell us what those services expose. In **Part 2 of Phase 2: Enumeration**, we'll start talking to them directly.

---