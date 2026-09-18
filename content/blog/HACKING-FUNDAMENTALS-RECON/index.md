---
title: "Hacking Fundamentals - Recon"
date: 2026-09-18
draft: false
summary: "Understanding Reconnaissance"
tags: ["Hacking Fundamentals"]
categories: ["blog"]
series: ["Hacking Fundamentals"]
showToc: true
author: "Moiz Bootwala"
---

Let's understand the first step in any cybersecurity engagement - Reconnaissance.
<!--more-->
Reconnaissance is a topic that deserves an entire series of its own, but I'm going to cram the essentials into this one post. Instead of covering every technique in depth, I'll give you a working idea of the process and the different ways it's done. The same core concepts apply across disciplines.

In this blog, we'll cover:
- Introduction to reconnaissance and footprinting
- footprinting using search engines
- finding domains and subdomains
- finding info about individuals and orgs
- finding info about a website
- email, whois, dns footprinting
- OSINT and steganography
- automated tools

## What Is Reconnaissance & Footprinting

Reconnaissance (also called footprinting) is the initial phase in a cyber attack where an attacker gathers information about a target network to identify potential entry points. This step is very important for building a clear map or blueprint of the security landscape before attempting any form of intrusion.

This process can be broadly categorized into 2 types:
- **Passive**: collecting information without directly interacting with the target. This could include searching publicly available resources such as websites, social media, or DNS records.
- **Active**: Collecting information through direct interaction with the target, such as using a network scanning tool or contacting the target servers.

> *Irrespective of the tools, techniques used for footprinting, the goal always remains the same: gather as much relevant information as possible about the target organization before moving to the actual hacking phase.*

Some of the data that are targeted to be recovered during this phase are:
- IP addresses and DNS records
- Accessible systems
- OS versions and listening services
- Firewalls and security devices
- Phone systems
- Organizational structure
- Phone numbers and email addresses
- Office locations
- Company history

This information provides a detailed understanding of the target, which attackers can use to develop a more effective plan for compromising the target infrastructure.

> On a real engagement, everything above is bound by **scope**. The specific IP ranges, hosts and applications that are agreed to be accessed by us, as opposed to anything explicitly out-of-bounds.

## Footprinting Using Search Engines

Search engines use something known as **crawlers** to scan active websites and index their content, including web pages, videos, images, and files. This indexed data is stored in a large database, which search engines query when a user submits a request. The results, known as **Search Engine Results Pages (SERPs)**, are ranked based on relevance.

Beyond the mainstream ones, a few lesser known engines can occasionally give us things that Google won't.

| **Search Engine** | **URL**                                                        |
| ----------------- | -------------------------------------------------------------- |
| WolframAlpha      | [https://www.wolframalpha.com/](https://www.wolframalpha.com/) |
| StartPage         | [https://www.startpage.com/](https://www.startpage.com/)       |
| MetaGer           | [https://metager.org/](https://metager.org/)                   |
| eTools            | [https://www.etools.ch/](https://www.etools.ch/)               |

### Google Dorks

**Google Dorks** are special search operators that let us refine a search with a lot more precision than a plain keyword query. The basic syntax is `operator:search_term`, and a large catalog of useful queries is maintained in the **[Google Hacking Database (GHDB)](https://www.exploit-db.com/google-hacking-database)**.

| **Dork**      | **Function**                                                                   |
| ------------- | ------------------------------------------------------------------------------ |
| `site:`       | Restricts search results to a specific site or domain.                         |
| `allinurl:`   | Limits results to pages with all query terms in the URL.                       |
| `inurl:`      | Restricts results to only pages containing the specified word in the URL.      |
| `allintitle:` | Limits results to pages containing all the specified query terms in the title. |
| `intitle:`    | Restricts results to only pages containing a specified word in the title.      |
| `location:`   | Finds information from or about a specific location.                           |
| `cache:`      | Displays the cached version of a web page stored by Google.                    |
| `filetype:`   | Limits the search to specific file types like PDF, DOC, etc.                   |

We can negate a dork with a minus sign (`-`). For example, `-site:google.com` will exclude results from google.com.

A few practical exercises that you can try:
1. Search a book
```
intitle:"complete guide to shodan" filetype:pdf
```

![searching for a book pdf](https://cdn.ziomsec.com/hacking-fundamentals-recon/1.webp)

2. Search for a specific document type on a domain
```
filetype:txt site:nasa.gov
```

![searching for a filetype on a domain](https://cdn.ziomsec.com/hacking-fundamentals-recon/2.webp)

3. Search for login portal
```
intitle:"login" inurl:admin
```

![searching for admin login portals](https://cdn.ziomsec.com/hacking-fundamentals-recon/3.webp)

The **Google Hacking Database** catalogs dorks by category: footholds, files containing usernames, sensitive directories, vulnerable servers, error messages, and more. It's worth browsing through even outside of an active engagement, just to get a feel for what kind of things end up indexed by accident. 

> **[Google Advanced Search](https://www.google.com/advanced_search)** gives you a graphical form for building more complex queries if you don't want to memorize operators.

## Hacker Search Engines

Beyond web content, there's a class of search engines built specifically to index *devices* rather than pages. This includes every internet-facing camera, router, database or any other device that responds to a probe.

- **[Shodan](https://www.shodan.io/)** is the best known of these. It's useful for both attackers looking for exposed systems and defenders wanting to know what of theirs is publicly visible. It also ships a CLI (`shodan init API_KEY` to configure, then query from the terminal), and community tools like **[ShonyDanza](https://github.com/fierceoj/ShonyDanza)** wrap it into a menu-driven interface for common lookups (host profiles, domain profiles, honeyscore, on-demand scans).
- **[Censys](https://censys.com/)** serves a similar purpose — discovering and monitoring every device on the internet — and is often used alongside Shodan since the two don't always index the same things.
- **[Netcraft](https://www.netcraft.com/tools/)** leans more toward website investigation — what tech stack a site runs, its domain history, hosting provider, and related infrastructure.
- **[Pentest-Tools](https://pentest-tools.com/)** bundles a bunch of these lookups (subdomain discovery, DNS lookup, vulnerability scanning) into one browser-based dashboard.
- **[ZoomEye](https://www.zoomeye.ai/)** is a Shodan-alike with its own operator syntax worth knowing: `port:22`, `os:linux`, `service:webcam`, `hostname:google.com`, `country:US`, `app:Apache`, `ip:8.8.8.8`, `cidr:8.8.8.8/24`. Shodan supports the same idea with operators like `city`, `country`, `geo`, `hostname`, `net`, `os`, `port` and `before`/`after` for filtering by date.

![shodan](https://cdn.ziomsec.com/hacking-fundamentals-recon/4.webp)

## Finding Domains And Subdomains

A subdomain like `admin.example.com` or `staging.example.com` is often forgotten, less maintained, and less scrutinized than the main site — which makes subdomain enumeration one of the highest-value activities in recon. More subdomains discovered means more attack surface to work with.

### Browser Tools For Subdomain Discovery

| **Tool**     | **Description**                                                                        | **Link**                                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Netcraft** | Discovers DNS information, domains, and subdomains.                                     | [https://searchdns.netcraft.com/](https://searchdns.netcraft.com/)                                             |
| **crt.sh**   | Certificate transparency log search — SSL certificates leak the hostnames they cover. | [https://crt.sh/](https://crt.sh/)                                                                             |
| **Nmmapper** | Web-based subdomain finder.                                                             | [https://www.nmmapper.com/sys/tools/subdomainfinder/](https://www.nmmapper.com/sys/tools/subdomainfinder/)   |

Certificate transparency is worth calling out specifically — every publicly trusted TLS certificate gets logged, and that log includes every hostname the certificate covers. A wildcard cert issued for `*.example.com` won't help, but plenty of orgs issue individual certs per subdomain, which crt.sh happily surfaces.

![crtsh](https://cdn.ziomsec.com/hacking-fundamentals-recon/5.webp)

### Command Line Tools For Subdomain Discovery

**[Sublist3r](https://github.com/aboul3la/Sublist3r)** is a Python tool that leans on OSINT — search engines, certificate transparency logs, and a handful of other sources — to enumerate subdomains for a given domain:

```bash
python3 sublist3r.py -d target.com
```

### ASN (Autonomous System Number)

An **ASN** is a unique identifier assigned to a network, used to group IP addresses into blocks. Pulling the ASN for a target organization can widen your view of their infrastructure considerably. You go from "one domain" to "every IP block this org owns."

| **Tool** | **Link**                                       |
| -------- | ----------------------------------------------- |
| **BGP**  | [https://bgp.he.net/](https://bgp.he.net/)     |
| **ARIN** | [https://www.arin.net/](https://www.arin.net/) |

## Finding Info About Individuals And Organizations

People are frequently the weakest link, and they tend to overshare — professional bios, past employers, hobbies, even office badge photos end up public without anyone intending it. A few tools make it easy to pull that together.

### Browser Based Tools

| **Tool**            | **Description**                                                    | **Link**                                                             |
| ------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **Spokeo**          | People search engine aggregating public sources.                   | [https://www.spokeo.com/](https://www.spokeo.com/)                   |
| **PeekYou**         | People search focused on social media profiles and public records. | [https://www.peekyou.com/](https://www.peekyou.com/)                 |
| **Followerwonk**    | Twitter analytics - find, analyze and compare profiles.            | [https://followerwonk.com/](https://followerwonk.com/)               |
| **Social Searcher** | Tracks public social media mentions in real time.                  | [https://www.social-searcher.com/](https://www.social-searcher.com/) |
| **Phonebook**       | Finds emails belonging to a particular domain.                     | [https://phonebook.cz](https://phonebook.cz)                         |
| **Webmii**          | Finds info about the person behind an email address.               | [https://webmii.com/](https://webmii.com/)                           |
| **checkusernames**  | Checks a username across many platforms at once.                   | [https://www.checkusernamess.com/](https://www.checkusernamess.com/) |

### Command Line Tools

**[theHarvester](https://github.com/laramies/theHarvester)** pulls emails, subdomains, and hosts from public sources like search engines, PGP key servers and Shodan, all in one pass:

```bash
theHarvester -d target.com -l 5 -b bing
```

![theHarvester tool](https://cdn.ziomsec.com/hacking-fundamentals-recon/6.webp)

**[Sherlock](https://github.com/sherlock-project/sherlock)** checks whether a given username exists across a huge range of social platforms — handy once you have a name or handle to pivot on:

```bash
sherlock 'target_username'
```

![sherlock tool](https://cdn.ziomsec.com/hacking-fundamentals-recon/7.webp)

> LinkedIn deserves a special mention here. It's often the single richest source for organizational structure, since employees tend to keep job titles, teams, and tenure current and public. Combined with job postings (which frequently leak technology stack details in the "requirements" section), you can build a decent picture of both the people and the systems behind a company without sending a single packet their way.

## Finding Info About A Website

Once you know a site exists, the next question is what it's built on and how it's put together.

| **Tool**                | **Description**                                           | **Link**                                                                                              |
| ----------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Burp Suite**          | Full web app security testing suite.                      | [https://portswigger.net/burp](https://portswigger.net/burp)                                          |
| **Zaproxy**             | Open-source web application security scanner.             | [https://www.zaproxy.org/](https://www.zaproxy.org/)                                                  |
| **WhatWeb**             | Identifies websites and the web technologies behind them. | https://github.com/urbanadventurer/whatweb                                                            |
| **Wappalyzer**          | Browser extension that detects a site's tech stack.       | [https://www.wappalyzer.com/](https://www.wappalyzer.com/)                                            |
| **Web Archive**         | Archived snapshots of a site over time.                   | [http://web.archive.org/](http://web.archive.org/)                                                    |
| **FoxyProxy**           | Quickly switch proxy servers while browsing the target.   | [Firefox add-on](https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/)                  |
| **User-Agent Switcher** | Pretend to browse from a different OS or browser.         | [Firefox add-on](https://addons.mozilla.org/en-US/firefox/addon/user-agent-string-switcher/versions/) |
| **Builtwith**           | Technology profiler for a site's stack.                   | [Firefox add-on](https://addons.mozilla.org/en-US/firefox/addon/builtwith/)                           |

![whatweb tool](https://cdn.ziomsec.com/hacking-fundamentals-recon/8.webp)

Some of this doesn't even need a dedicated tool; the **network tab** in your browser's dev console will usually show server headers on any response, which alone can reveal the web server and version.

**Burp Proxy** intercepts every HTTP/S request and response between your browser and the target, letting you pick through the **HTML source** (hidden links, comments, structural hints) and **cookies** (which can leak details about session handling and the software behind it).

![burp suite traffic intercept](https://cdn.ziomsec.com/hacking-fundamentals-recon/9.webp)

### Web Spiders And Wordlists

Web spiders crawl a site to collect employee names, email addresses, and other data useful for social engineering, though a `robots.txt` file can block them from certain directories.

| **Tool**      | **Description**                | **Link**                                                                             |
| ------------- | --------------------------------- | ------------------------------------------------------------------------------------ |
| **WebScarab** | Web spidering for data collection. | [https://github.com/OWASP/OWASP-WebScarab](https://github.com/OWASP/OWASP-WebScarab) |
| **Photon**    | Fast, customizable web crawler.   | [https://github.com/s0md3v/Photon](https://github.com/s0md3v/Photon)                 |

> A site's `sitemap.xml` is worth checking directly too. It's meant for search engine crawlers, but it doubles as a free directory listing.

To pull every URL a domain has ever had indexed (including from old, possibly forgotten versions of the site), **[waybackurls](https://github.com/tomnomnom/waybackurls)** queries the Wayback Machine's archive directly.

And once you've mapped a site's content, **[Cewl](https://github.com/digininja/CeWL)** can turn that content into a custom wordlist — useful later for password attacks against the same organization, since people tend to reuse company-specific jargon in their passwords:

```bash
cewl -w list.txt -d 5 -m 5 http://target.com
```
- `-w`: write the output to a file.
- `-d 5`: crawl to a depth of 5.
- `-m 5`: only keep words 5 characters or longer.

### WAF Detection

Before you go further, it's worth knowing whether a **Web Application Firewall** sits in front of the target — it changes what kind of testing is even worth attempting. **[wafw00f](https://github.com/EnableSecurity/wafw00f)** handles this detection:

```bash
wafw00f -l                     # list all WAFs it can detect
wafw00f https://target.com     # check a single target
wafw00f https://target.com -a  # check for all possible WAF matches, not just the first
```

## Email, Whois And DNS Footprinting

### Email Footprinting

Emails are one of the more directly useful things to gather as they double as usernames for a huge range of services. **[Infoga](https://github.com/The404Hacking/Infoga)** automates gathering and verifying email addresses tied to a domain:

```bash
infoga --target site.com --source all
```

**[Hunter.io](https://hunter.io/email-finder)** does something similar from the browser, searching for organizational email addresses by domain.

### Whois Footprinting

**Whois** is a protocol used to retrieve registration information for a domain or IP address — registrant name, organization, contact details, creation and expiry dates, and the authoritative name servers. A Whois server listens on **TCP port 43**.

Registration data is stored under one of two models:
- **Thick Whois**: the registry itself stores the complete record for every domain.
- **Thin Whois**: the registry only stores which registrar's Whois server holds the full details.

Regional Internet Registries (RIRs) each maintain Whois data for their region:
- **ARIN** (Americas)
- **AFRINIC** (Africa)
- **RIPE NCC** (Europe)
- **LACNIC** (Latin America & Caribbean)
- **APNIC** (Asia-Pacific)

From the CLI:

```bash
whois example.com
```

Browser-based lookups work the same way. **[DomainTools Whois](https://whois.domaintools.com/)** is a common option.

> If a registrant didn't use privacy protection when registering, **reverse Whois** (e.g. via **[Whoxy](https://www.whoxy.com/)**) lets you go the other direction - find every other domain registered under the same name or email address.

### DNS Footprinting

DNS records reveal a domain's IP addresses, mail servers, subdomains, and infrastructure choices. Two terms worth knowing here: **DNS poisoning** (tampering with a DNS cache to redirect legitimate traffic) and **DNSSEC** (a security extension designed to prevent exactly that).

| **Record Type** | **Description**                                                     |
| ---------------- | ---------------------------------------------------------------------- |
| **A**            | Maps a domain to its IPv4 address.                                  |
| **AAAA**         | Maps a domain to its IPv6 address.                                  |
| **MX**           | Specifies the mail server for a domain.                             |
| **NS**           | Points to the domain's name server.                                 |
| **CNAME**        | Alias for a domain or subdomain.                                    |
| **SOA**          | Indicates the primary authoritative name server.                    |
| **SRV**          | Points to a specific service for a domain.                          |
| **PTR**          | Maps an IP back to its domain (reverse DNS).                        |
| **RP**           | Responsible person associated with the domain.                      |
| **HINFO**        | Hardware and OS information for a host.                              |
| **TXT**          | Unstructured text — often used for SPF/DKIM verification.           |

Two of the more common lookup tools:

```bash
nslookup
> set type=A
> example.com
```

```bash
host www.target.com          # retrieves its IP address
host -t mx target.com        # retrieves MX records
```

> If a lookup returns multiple IP addresses for one hostname, that's often a sign the site sits behind Cloudflare or a similar reverse proxy/CDN.

**dig** is the more flexible of the two, and can also be pointed at a specific server:

```bash
dig example.com
dig @1.1.1.1 example.com
```

Occasionally a misconfigured name server will allow a full **zone transfer**, handing over every record it holds for a domain in one request — effectively a map of the entire internal DNS structure:

```bash
dig axfr @nameserver.com target.com
```

A handful of web tools and CLI utilities specialize in pulling this all together automatically:

| **Web Tool**       | **URL**                                                        |
| ------------------- | -------------------------------------------------------------- |
| **SecurityTrails**  | [https://securitytrails.com/](https://securitytrails.com/)     |
| **DNSDumpster**     | [https://dnsdumpster.com/](https://dnsdumpster.com/)           |
| **YouGetSignal**    | [https://www.yougetsignal.com/](https://www.yougetsignal.com/) |
| **crt.sh**          | [https://crt.sh/](https://crt.sh/)                             |
| **RapidDNS**        | [https://rapiddns.io/](https://rapiddns.io/)                   |

| **CLI Tool**   | **Description**                                                                     |
| -------------- | ---------------------------------------------------------------------------------------- |
| **DNSRecon**   | Enumerates standard DNS records and attempts zone transfers.                             |
| **Fierce**     | Looks for non-contiguous IP space and misconfigured DNS.                                 |
| **DNSenum**    | Detailed DNS recon covering records, subdomains, and more: `dnsenum target.com`         |

## OSINT And Steganography

**OSINT** — Open Source Intelligence — is the umbrella term for everything we've covered so far: gathering and analyzing information from publicly accessible sources, and turning that raw data into something you can actually act on. The **[OSINT Framework](https://osintframework.com/)** is worth bookmarking — it's a categorized index of nearly every tool mentioned in this post and many more, organized by what kind of information you're after.

![osint framework website](https://cdn.ziomsec.com/hacking-fundamentals-recon/10.webp)

### Github Footprinting

Public code repositories are a surprisingly consistent source of leaks — API keys, credentials, and connection strings, often committed by accident and sometimes still recoverable from commit history even after being "removed." `.env` files are a favorite target since they're specifically meant to hold sensitive configuration:

```plaintext
path:**/.env
path:**/.env MAIL_HOST=smtp.gmail.com
```

Purpose-built tools go further than a manual search:
- **[gitrob](https://github.com/michenriksen/gitrob)** and **[gitleaks](https://github.com/gitleaks/gitleaks)** scan repositories for exposed secrets.
- **[GitTools](https://github.com/internetwache/GitTools)** and **[GitHack](https://github.com/lijiejie/GitHack)** can even reconstruct source code from an exposed `.git` directory on a live web server.

### Deep And Dark Web

The **Deep Web** is simply everything not indexed by standard search engines — login-gated pages, dynamically generated content, private databases. The **Dark Web** is a subset of that, built specifically around anonymity, and reachable only through dedicated networks like **Tor**, **Freenet**, **GNUnet**, **I2P** and **Retroshare**. **TAILS** is an operating system built specifically for operating in this space safely.

| **Tool**                    | **Link**                                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| **The Hidden Wiki**         | [https://thehiddenwiki.org/](https://thehiddenwiki.org/)                                       |
| **ExoneraTor**              | [https://metrics.torproject.org/exonerator.html](https://metrics.torproject.org/exonerator.html) |
| **OnionLand Search Engine** | [https://onionlandsearchengine.net/](https://onionlandsearchengine.net/)                        |
| **DarkNet Search**          | [https://darknetsearch.com/](https://darknetsearch.com/)                                        |

### Steganography

Steganography is the practice of hiding data inside otherwise unremarkable files like an image, an audio clip; and it comes up in recon whenever a target's publicly shared media might be worth a closer look than a glance.

- **Binwalk** analyzes and extracts embedded data from binary files: `binwalk -e <filename>`
- **Steghide** hides or extracts data from image/audio files: `steghide extract -sf image.png`
- **ExifTool** reads (and strips) file metadata: `exiftool image.png` to view, `exiftool -all= <file>` to strip it all
- **ZSteg** detects hidden data specifically in PNG and BMP files: `zsteg -a image.png`
- **zbarimg** scans and decodes barcodes from an image

> **[WiGLE.net](https://wigle.net/)** is worth a mention here too. It maps WiFi networks and cell towers globally, and can occasionally be used to pin down a physical location from nothing more than a network name.

## Automated Recon Tools

Doing all of the above by hand, one tool at a time, doesn't scale; which is why most of it eventually gets wrapped into frameworks that chain multiple sources together in a single run.

| **Tool**                                                    | **Description**                                                                           |
| ----------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **[Recon-ng](https://github.com/lanmaster53/recon-ng)**     | Full reconnaissance framework, modular like Metasploit but focused purely on recon.       |
| **[SpiderFoot](https://github.com/smicallef/spiderfoot)**   | Browser-based automation of OSINT and recon across many data sources.                     |
| **[Metagoofil](https://www.kali.org/tools/metagoofil/)**    | Extracts metadata from public documents (PDF, DOC, etc.) found on a target's site.        |
| **[ReconDog](https://github.com/s0md3v/ReconDog)**          | Lightweight recon with minimal input required.                                            |
| **[OSRFramework](https://github.com/i3visio/osrframework)** | Automates checking usernames, domains and emails across platforms.                        |
| **[Reconftw](https://github.com/six2dez/reconftw)**         | Combines multiple subdomain enumeration and scanning tools into one pipeline.             |
| **[Amass](https://github.com/owasp-amass/amass)**           | OWASP project for network mapping and attack surface discovery via subdomain enumeration. |
| **[Raccoon](https://github.com/evyatarmeged/Raccoon)**      | DNS records, subdomains, open ports and website details in one pass.                      |
| **[Trickest](https://trickest.io/)**                        | Browser-based automation of scanning, enumeration and info gathering.                     |
| **[Black Widow](https://github.com/1N3/BlackWidow)**        | Recon and web app scraping/attack tool.                                                   |

![recon-ng tool](https://cdn.ziomsec.com/hacking-fundamentals-recon/11.webp)

## Conclusion

That covers reconnaissance and footprinting at a practical level: search-engine and dork-based OSINT, hacker search engines like Shodan and Censys, finding domains and subdomains, digging up information on individuals and organizations, footprinting a website's tech stack and structure, email/whois/DNS footprinting, OSINT sources including the dark web and steganography, and the automated frameworks that tie all of it together.

Everything in this phase is passive or, at most, lightly interactive. In the next post, I'll talk about **Phase 2: Scanning**, where we start actively probing the target to confirm which hosts are alive, which ports are open, and what's actually listening on them.

---