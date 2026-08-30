---
title: "Hacking Fundamentals - Intro"
date: 2026-08-30
draft: false
summary: "Introduction to the fundamentals of hacking"
tags: ["Hacking Fundamentals"]
categories: ["blog"]
series: ["Hacking Fundamentals"]
showToc: true
author: "Moiz Bootwala"
---

Getting started in cybersecurity and hacking
<!--more-->
Cybersecurity is one of those fields everyone tells you to "just start learning", but rarely tells you where to begin. The *"Hacking Fundamentals"* series of blogs covers the fundamentals of hacking. By the end of it, you should have a good understanding on the core concepts and terminology

The series is structured around the actual phases of a hacking engagement. White the phases of hacking apply across every genre, the way each of the phases are performed vary a lot. This series focuses specifically on tools and techniques for network and web hacking, and the methodology leans towards how a "penetration test" is run.

In this blog, we'll understand the building blocks of hacking. This includes any pre-requisite knowledge, terms, concepts that is generally required in order to learn hacking. Here's what we'll cover:
- what is infosec and why is it even needed
- types of hackers, types of testing
- methodologies, phases and frameworks related to security
- key terminologies
- networking
- key protocols

## Understanding Infosec And Its Importance

Information security or "*infosec*" is simply the practice of protecting information from threats like hackers. 

Infosec exists because information is valuable to the company, and its loss or compromise affects far more than just that company's direct employees. Customers, users, and stakeholders all bear the consequences too. Failing to protect it can mean significant consequences for the organization, including financial loss and reputational damage.

Infosec consists of 5 key elements:
1. **Confidentiality**: ensures that information is accessible only to authorized individuals.
2. **Integrity**: guarantees the accuracy and completeness of data,  ensuring it hasn't been altered or tampered with by unauthorized parties.
3. **Availability**: ensures that data is available, stored and processed as needed by the user.
4. **Authenticity**: verifies that the information is genuine and uncorrupted.
5. **Non-Repudiation**: prevents the sender from denying having sent the information, and the receiver from denying having received it.

> **Confidentiality**, **Integrity** and **Availability** are collectively known as the **[CIA Triad](https://www.geeksforgeeks.org/cybersecurity/the-cia-triad-in-cryptography/)**.

Now, let's look into cyber attacks.

Every cyber attack is driven by a *motive*, follows a particular *method* and involves exploitation of a *vulnerability*. A motive arises when the target system holds or processes something valuable, attracting attackers to exploit weaknesses in the system. Common motives behind cyber attacks involve:
- Disrupting business operations
- Stealing information
- Manipulating data
- Creating fear and chaos
- Causing financial losses
- Seeking revenge
- Demanding ransom

The method is the specific path or means used to carry out the attack — phishing emails, malware, social engineering, or exploiting an exposed service, to name a few. The vulnerability is the weakness or bug that makes the method possible in the first place — an unpatched server, a weak password policy, or an employee who clicks the wrong link.

### Types Of "Attackers"

The attackers that perform cyber attacks are known as Hackers. Hackers are individuals who break into networks, sometimes for malicious purposes.

Based on their authorization to perform these attacks, hackers can be categorized into:
1. **Black Hats**: These hackers engage in malicious activities. For example, ransomware authors infect devices with malicious code and hold data for ransom.
2. **White Hats**: These are also known as "ethical hackers" and are ones who use their skills to defend systems. For example, a penetration tester performing an authorized security engagement on a company.
3. **Gray Hats**: These are hackers who operate both offensively and defensively, depending on the situation. For example, a hacker taking down a scamming website.

From time to time, you may come across these terms that also describe hackers:
- **Suicide Hackers**: Individuals who aim to destroy infrastructure without concern for consequences.
- **Script Kiddies**: Unskilled hackers or newbies who rely on pre-written scripts and tools made by others and are sloppy.
- **Cyber Terrorists**: Hackers who use cyber attacks to promote terrorism.
- **State Sponsored Hackers**: Employed by governments to carry out cyberattacks.
- **Hacktivists**: Hackers who use their skills to push political or social agendas.

With so many threats, you may now understand the importance of "Ethical Hacking"... It helps answer critical questions such as:
1. What information can an attacker access on the target system?
2. What damage can be done with this information?
3. Are the attacker's activities being detected by the systems?

Additionally, security testing can be categorized into 3 types based on how much information we know about the target:
- **Black Box**: The tester has no prior knowledge of the system.
- **White Box**: The tester has full knowledge of the system.
- **Gray Box**: The tester has partial knowledge of the system.

## Hacking Methodologies

Like other areas, hacking follows a loose methodology of how things are done. The steps of hacking can be loosely categorized into 5 broad phases, and each phase can involve multiple sub-activities that don't necessarily happen in strict order:

**PHASE 1:** Footprinting and reconnaissance involves collecting as much information about the target as possible. The information gathered may include IP ranges, domain names, employee details, work patterns, business areas, customer base etc.
**PHASE 2:** Scanning involves identifying active hosts, open ports and services on the target system to validate the information.
**PHASE 3:** Enumeration is the process of establishing active connections to the target system for further information gathering.
**PHASE 4:** Vulnerability analysis involves evaluating how vulnerable a system is to different types of attacks.
**PHASE 5:** Finally, system hacking involves exploiting the target. This includes gaining initial access using anything discovered in the above phases, escalating privileges, establishing persistence and clearing any tracks that would otherwise get us caught.

We will dive into these phases in the upcoming blogs but feel free to search and read about them online. Besides the above methodology, there is also the **[Cyber Kill Chain Methodology](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)** which involves more stages and clearer distinction in each of the phases.

> These methodologies can seem confusing at first, but their point is simply to give a loose guide for how to proceed with an attack. You'll often notice that, loosely, almost all methodologies follow the same underlying flow: information gathering → looking for vulnerabilities → exploiting vulnerabilities (and gain access to the system) → post exploitation steps.

Beyond methodologies, there are also a handful of well-known frameworks that can be used for referencing a specific stage of an attack, or for understanding someone else's when reading about a breach.
- **[MITRE ATT&CK FRAMEWORK](https://www.ibm.com/think/topics/mitre-attack)**: This framework categorizes hacking techniques into specific phases to better understand and defend against them. You can access the framework here: https://attack.mitre.org/
- **[DIAMOND MODEL OF INTRUSION ANALYSIS](https://www.eccouncil.org/cybersecurity-exchange/ethical-hacking/diamond-model-intrusion-analysis/)**: This highlights the four core elements involved in any cyber attack: adversary, victim, capability and infrastructure.
- **[OWASP](https://owasp.org/)**: The Open Web Application Security Project is a framework that is community driven, and updated frequently. As the name suggests, it is used solely to test the security of [web applications](https://owasp.org/Top10/2025/) and services. However, they have also expanded their frameworks to other areas like [mobile](https://owasp.org/www-project-mobile-top-10/), [API](https://owasp.org/www-project-api-security/), [LLMs](https://owasp.org/www-project-top-10-for-large-language-model-applications/). 
- **[NIST CYBERSECURITY FRAMEWORK](https://www.nist.gov/cyberframework)**: This is a framework used to improve an organizations cybersecurity standards and manage the risks of cyber threats. It provides guidelines on security controls and benchmarks for success for orgs from critical infrastructure.

## Key Terminologies

These are just some terms/buzz words you'll hear when working in security.

| **Term**        | **Meaning**                                                                                                                                                                                               |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vulnerability   | A flaw or weakness in a system that can be exploited.                                                                                                                                                     |
| Threat          | Any potential danger that seeks to exploit a vulnerability. Something that poses risk to an asset we care about.                                                                                          |
| Threat actor    | A person or group of people embodying a threat.                                                                                                                                                           |
| Exploit         | A method used to take advantage of a security flaw to gain unauthorized access or benefit from it.                                                                                                        |
| Attack surface  | All the points of contact on a system or network that could be vulnerable to exploitation.                                                                                                                |
| Attack vector   | A specific vulnerability and exploitation combination that can further a threat actor's objective.                                                                                                        |
| Payload         | The part of an attack that performs a malicious action, such as delivering malware.                                                                                                                       |
| Zero-day attack | An attack that happens before a vendor becomes aware of or is able to patch the vulnerability.                                                                                                            |
| TTP             | Tactics, techniques and procedures used by cybersecurity professionals to describe the behaviors, processes, actions and strategies used by a threat actor to develop threats and engage in cyberattacks. |
| APT             | Advanced persistent threat is a cyber attack on a network where the attacker gains and maintains access to the targeted network and remains undetected for a significant period.                          |
| Protocol        | A set of rules that have been agreed upon to perform a particular task. This involves standards followed for communicating, or performing anything for that matter.                                       |

Vulnerabilities and exploits are often confused. In computer programs, vulnerabilities occur when someone who interacts with the program can achieve specific objectives that are unintended by the programmer. When these objectives provide the user with access or privileges that they aren't supposed to have, and when they are pursued deliberately and maliciously, the user's actions become an **exploit**.

Besides the above terminologies, there are certain actions that help make an environment more secure:
1. Principle of least privilege: Each part of a system (like users, machines or code) should only have the minimum access needed to perform its task.
2. Defense in depth: Security should be layered so that if one layer fails, others still protect the system. Example: a garage secured by an electronic code, a key, and a voice-activated alarm system.
3. Backups: having copies of data taken at a specific time. These are used to recover data if it gets deleted or corrupted. There are different types of backups based on how much data is copied and stored, each with its own benefits and drawbacks. There are also different ways to store backups, either online or offline.
4. Encryption: this is a powerful tool for protecting sensitive information, but it needs to be used properly. Encrypting everything is only useful if you can decrypt it when needed. Some encryption, like TLS, is temporary and only lasts for the duration of a session. Proper management of encryption keys is essential, and it's important to ensure only authorized people or systems can access decrypted data.

## Networking

While you don't necessarily need to know how to setup an entire network from scratch, having some basic understanding of what a network is and some key networking concepts is quite helpful.

A network is a system where multiple devices are connected and can share resources with each other. These devices can be connected either via wires (twisted pair cables, fibre optic cables) or wirelessly (WiFi, radio).

Within a network, devices need an identity. Just like we require an address to deliver something, devices need addresses to communicate with each other. There are 2 main types of addresses within a network that are used for communication, [IP address](https://en.wikipedia.org/wiki/IP_address) and [MAC address](https://en.wikipedia.org/wiki/MAC_address). Both are assigned to the **[NIC (network interface card)](https://en.wikipedia.org/wiki/Network_interface_controller)** of the system.

- IP addresses are like labels such as `192.168.1.1` or `1002:1db9:0000:0000:8a2e:0370:7334:0000` that are assigned to a device connected to a network that communicates using an internet protocol. It helps with identifying the device's location and identifying the network interface of the device. IP addresses help devices communicate with others in a different network.
- MAC addresses are unique identifiers assigned to NIC for use as a network address in communication within the same network. These are assigned by device manufacturers and are hence also known as burned-in address.

Additionally, each IP on the NIC have 65535 ports. Port 0 is used to select a random available port and cannot be bound to a particular service. A **port** refers to either a physical or virtual connection point for devices to communicate. **Virtual ports** are associated with an IP address and the protocol used for communication. Ports uniquely identify applications and processes running on a single computer.

I would recommend spending some time understanding IP addresses, subnetting and MAC addresses. There are plenty of resources out on the internet that will teach you everything you need to know about those topics so I won't spend time getting into the specifics.

> I found these resources quite helpful so you can have a look at them.
> - NetworkChuck Subnetting Playlist: https://www.youtube.com/playlist?list=PLIhvC56v63IKrRHh3gvZZBAGvsvOhwrRF
> - FreeCodeCamp MAC Address: https://www.youtube.com/watch?v=tcvTjQldnBI

Before proceeding further, let's quickly have a look at the OSI model that make up and define networking.
- The **[OSI Model](https://en.wikipedia.org/wiki/OSI_model)** (Open Systems Interconnection) describes the functions of a communication system by dividing them into 7 layers.
- The **[TCP/IP Model](https://en.wikipedia.org/wiki/Internet_protocol_suite)** is a simplified, 4-layer version of the OSI model.

| Number | Layer              | Description                                                                |
| ------ | ------------------ | -------------------------------------------------------------------------- |
| 7      | Application Layer  | Manages human-computer interactions and network services for applications. |
| 6      | Presentation Layer | Ensures data is in a usable format; handles encryption and decryption.     |
| 5      | Session Layer      | Manages and maintains sessions, ports, and connections.                    |
| 4      | Transport Layer    | Ensures reliable data transfer using protocols like TCP and UDP.           |
| 3      | Network Layer      | Determines the physical path for data transmission (routing).              |
| 2      | Data Link Layer    | Defines the format of data on the network (frames, MAC addresses).         |
| 1      | Physical Layer     | Transmits raw bit streams over physical media (e.g., cables, radio waves). |

With that out of the way, let's understand 2 of the most widely used protocols. TCP and UDP. Protocol is just an established set of rules that governs how data is transmitted, received and interpreted between different devices across a network. Protocols allow different types of devices to communicate with each other and is what let's us create new devices, services and applications that can easily communicate with others.

> To better understand how these protocols work and look in a network, I'll be using a popular network protocol analyzer called **[Wireshark](https://www.wireshark.org/)**.

### Transmission Control Protocol (TCP)

TCP protocol is a stateful protocol used during communication. A stateful protocol is a communication protocol in which the receiver may retain the session state and other information from previous communication and requests.

Famous services that use TCP for communication include:
- HTTP/S - used by web apps
- FTP/S - used for file transfer
- SSH / Telnet / RDP - used for remote connections
- SMTP / IMAP / POP3 - used for email retrieval and routing.

TCP connection establishment involves the infamous 3 way handshake:
1. **Host A** sends a **SYN** (synchronize) packet with a proposed initial sequence number to **Host B**.
2. **Host B** receives the SYN and responds with a **SYN-ACK** packet (synchronization + acknowledgment).
3. **Host A** receives the SYN-ACK and sends an **ACK** packet to confirm.
4. **Host B** receives the ACK, and the connection is established.

```plaintext
SRC --> SYN --> DST
SRC <-- SYN-ACK <-- DST
SRC --> ACK --> DST
```

When a connection needs to terminate, it does the following:
1. **Host A** sends a **FIN** (finish) flag to indicate the end of data transmission.
2. **Host B**, in a **passive close** state, acknowledges the FIN with an **ACK** but keeps the connection open temporarily.
3. **Host A** enters a **time_wait** state and sends an ACK to confirm.
4. **Host B** receives the ACK and then closes the connection.

```plaintext
SRC --> FIN --> DST
SRC <-- ACK <-- DST
SRC <-- FIN <-- DST
SRC --> ACK --> DST
```

Let's practically look at this connection establishment and termination of a TCP connection using **WireShark**:

> Run WireShark on loopback by selecting `lo` interface to monitor.

***Let's start by having a look at the complete TCP lifecycle:*** 
```
SYN -> SYN,ACK -> ACK -> PSH,ACK -> FIN,ACK
```

**STEP 1**: Create a demo file and serve it using an HTTP server
```
$ echo "hello from the lab" > index.html
$ python3 -m http.server 8080
```

**STEP 2**: In WireShark, add the following filter:
```
tcp.port == 8080
```
> This will help us capture traffic initiating or ending on TCP port 8080.

**STEP 3**: From another terminal, make a request to the served html page
```
$ curl -v http://127.0.0.1:8080/index.html
```

![HTTP communication via netcat](https://cdn.ziomsec.com/hacking-fundamentals-intro/1.webp)

WireShark will capture the following:
1. client → server : `[SYN]`
2. server → client : `[SYN, ACK]`
3. client → server : `[ACK]`

This establishes the connection.

![Traffic captured by WireShark](https://cdn.ziomsec.com/hacking-fundamentals-intro/2.webp)

4. client → server : `[PSH, ACK]`
This is our actual `GET /index.html HTTP/1.1` request. We can click it and check `Follow -> TCP Stream` to see the literal HTTP request text.

![Inspecting the HTTP request](https://cdn.ziomsec.com/hacking-fundamentals-intro/3.webp)

> This can also be verified by selecting the `GET /index.html` request and checking `Transimission Control Protocol -> Flags` to find the Push and Acknowledgement packets set.

5. server → client : `[PSH, ACK]`
This is the HTTP response, containing *"Hello from the lab"*.

6. both directions: `[FIN,ACK]` / `[ACK]` pairs
The next couple of requests is caused because Python's `http.server` closes the connection after every response by default, so we get a teardown immediately.

![Termination of connection](https://cdn.ziomsec.com/hacking-fundamentals-intro/4.webp)

***Let's look at `RST` flag.***
This isn't something that an app triggers on purpose, it's what the networks tack itself does when a connection is refused or aborted.

**STEP 1**: Start SSH service
```
$ sudo service ssh start
```

**STEP 2:** In WireShark, add the following filter
```
tcp.port == 22 or tcp.port 4444
```

**STEP 3:** Perform a scan on both ports using `nmap`
```
sudo nmap -sS -p 22,4444 127.0.0.1
```
> `nmap` is a network scanning tool that can be used for scanning and enumeration. We will look into it in the upcoming blogs.

![Performing nmap scan to capture RST packet](https://cdn.ziomsec.com/hacking-fundamentals-intro/5.webp)

Since we started the SSH service earlier, we get port 22 as open. The sequence of packets in WireShark is `SYN -> SYN,ACK -> RST`.

![Analyzing traffic captured by WireShark](https://cdn.ziomsec.com/hacking-fundamentals-intro/6.webp)

`-sS` is a flag that makes nmap never complete the handshake. As soon as it sees `SYN-ACK` confirming the port is open, it sends `RST` to tear the half-open connection down rather than finishing it. This is the actual textbook behaviour of an **NMAP stealth scan** (which we will be covering in upcoming blogs).

As for port 4444, since nothing was running on it, it returned closed. The sequence of packets are `SYN ->  RST,ACK`. This is just what any TCP/IP stack does on a `SYN` to a port nothing is listening on. The OS itself sends the `RST`.

![Analyzing traffic captured by WireShark](https://cdn.ziomsec.com/hacking-fundamentals-intro/7.webp)

Notice the contrast here. `FIN` is negotiated (both sides agree, 3-4 packets are sent), whereas `RST` is one way thing (one packet, no negotiations).

Here's what these flags denote:

| Flag | Name           | Function                                                                                  |
| ---- | -------------- | ----------------------------------------------------------------------------------------- |
| SYN  | Synchronize    | Set during initial connection setup to negotiate sequence numbers and connection details. |
| ACK  | Acknowledgment | Confirms the receipt of a SYN or data packet; used after the initial SYN.                 |
| RST  | Reset          | Forces the immediate termination of a connection, in both directions.                     |
| FIN  | Finish         | Gracefully closes a connection after all data has been sent.                              |
| PSH  | Push           | Immediately sends data without waiting for buffering.                                     |
| URG  | Urgent         | Marks data as urgent, requiring priority handling (e.g., interrupting a message).         |

That sums up how TCP works. Any service using TCP for communication will follow this approach. Some common services that run on TCP along with their default ports are:

| **Default Port** | **Service**                                                                       | **Used For**                                           |
| ---------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 20 / 21          | [FTP](https://en.wikipedia.org/wiki/File_Transfer_Protocol)                       | File transfer(20) for data, (21) for control/commands  |
| 22               | [SSH](https://www.cloudflare.com/learning/access-management/what-is-ssh/)         | Secure remote connection / administration              |
| 23               | [Telnet](https://en.wikipedia.org/wiki/Telnet)                                    | Remote connection (unencrypted)                        |
| 25               | [SMTP](https://www.cloudflare.com/learning/email-security/what-is-smtp/)          | Sending/relaying email                                 |
| 53               | [DNS](https://www.cloudflare.com/learning/dns/what-is-dns/)                       | Domain name resolution                                 |
| 80               | [HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)                         | Web communication (unencrypted)                        |
| 110              | [POP3](https://en.wikipedia.org/wiki/Post_Office_Protocol)                        | Receiving email                                        |
| 135              | [MS RPC](https://www.akamai.com/blog/security-research/msrpc-security-mechanisms) | Microsoft RPC / Windows service communication          |
| 137–139          | [NetBIOS](https://wirexsystems.com/resource/protocols/netbios/)                   | Windows network communication and file/printer sharing |
| 143              | [IMAP](https://en.wikipedia.org/wiki/Internet_Message_Access_Protocol)            | Receiving/managing email                               |
| 389              | [LDAP](https://ldap.com/)                                                         | Directory services / authentication                    |
| 443              | [HTTPS](https://www.cloudflare.com/learning/ssl/what-is-https/)                   | Secure web communication                               |
| 445              | [SMB](http://en.wikipedia.org/wiki/Server_Message_Block)                          | Windows file/printer sharing                           |

### User Datagram Protocol (UDP)

UDP protocol is a stateless protocol widely used for communication where speed matters and does not require retaining any session states or information. Hence, UDP services do not rely on or expect a confirmation for any of the packet/data transferred through it.

Let's practically look at UDP connection and communication through WireShark to understand it better.

Similar to the TCP practicals, we'll start WireShark and select the `eth0` interface (or any other one which is connected to the network)

**STEP 1:** Add the following filter on WireShark
```
udp.port == 53
```

**STEP 2:** Run a DNS query using `dig`
```
$ dig example.com

$ # if you have no internet point it to your gateway
$ dig @192.168.x.1 example.com
```

![Performing DNS query](https://cdn.ziomsec.com/hacking-fundamentals-intro/8.webp)

WireShark will display 2 packets - one query and one response.

![Analyzing packets captured in WireShark](https://cdn.ziomsec.com/hacking-fundamentals-intro/9.webp)

Unlike TCP, UDP packets have no flag field at all. Hence, this demonstrates that nothing here is retained, negotiated or acknowledged at the Transport layer, which is why UDP is stateless.

The same result can also be observed using `netcat` instead of a domain:
```
$ # terminal 1
$ nc -ulp 9999

$ # terminal 2
$ nc -u 127.0.0.1 9999
```

![UDP communication with netcat](https://cdn.ziomsec.com/hacking-fundamentals-intro/10.webp)

![Analyzing UDP packets with WireShark](https://cdn.ziomsec.com/hacking-fundamentals-intro/11.webp)

Every packet is bare - no handshake, no flag fields and no teardown sequence.

Some popular and common services that run on UDP are:

| **Default Port** | **Service**                                                                                       | **Used For**                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 53               | [DNS](https://www.cloudflare.com/learning/dns/what-is-dns/)                                       | Domain name resolution                                         |
| 67 / 68          | [DHCP](https://www.geeksforgeeks.org/computer-networks/dynamic-host-configuration-protocol-dhcp/) | Automatically assigning IP addresses and network configuration |
| 69               | [TFTP](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol)                              | Simple file transfer                                           |
| 137–138          | [NetBIOS](https://wirexsystems.com/resource/protocols/netbios/)                                   | Name resolution and connectionless network communication       |
| 389              | [LDAP](https://ldap.com/)                                                                         | Directory services / directory queries                         |

---