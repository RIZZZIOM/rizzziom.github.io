---
title: "Hacking Fundamentals - Enumeration"
date: 2026-10-01
draft: false
summary: "Understanding Enumeration"
tags: ["Hacking Fundamentals"]
categories: ["blog"]
series: ["Hacking Fundamentals"]
showToc: true
author: "Moiz Bootwala"
---

Let's turn the ports we found into information we can actually use.
<!--more-->
This is **Part 2 of Phase 2: Scanning, Enumeration and Vulnerability Analysis**. Part 1 ended with a list of live hosts, open ports and possible services. Enumeration takes the next step: we establish active connections to those services, ask them targeted questions and record what they reveal. Towards the end, we will analyze those findings for vulnerabilities before moving into exploitation.

Enumeration is a huge topic because every protocol works differently. Covering every service in one post isn't really possible, so we'll concentrate on five examples that teach the underlying process: FTP, MySQL, SMB, HTTP and NFS.

## What Is Enumeration

**Enumeration** is the process of actively gathering detailed information about a target's users, machines, network shares and services. It usually involves establishing a connection and sending directed queries to a service.

The easiest way to separate it from scanning is by the question being asked. For example, let's say there's an SMB service running on the target:

| **Stage**       | **Question**                     | **Example Result**                                  |
| --------------- | -------------------------------- | --------------------------------------------------- |
| **Scanning**    | What is reachable and listening? | TCP port 445 is open and appears to be SMB.         |
| **Enumeration** | What does that service expose?   | A guest session works and reveals a readable share. |
| **Vulnerability analysis** | Does that exposure create a weakness? | The guest-accessible share exposes sensitive files. |

We ask similar questions for other services as well:
- FTP (TCP/21) - is anonymous login enabled? Which files are readable or writable?
- MySQL (TCP/3306) - does it accept weak or empty credentials? Which databases, tables, users and privileges are exposed?
- SMB (TCP/445) - which shares exist? Do null or guest sessions work? Which files are inside?
- HTTP (TCP/80) - which technologies, directories, files and virtual hosts are present?
- NFS (TCP/UDP 2049) - which directories are exported? Can we mount and read them?

Enumeration is the natural next step we take after scanning. Vulnerability analysis then helps us separate ordinary service information from weaknesses that may provide a realistic attack path. Together, these activities determine which service to target and what may be worth exploiting.

Let's understand this further using the same lab from the previous blog - **[metasploitable-2](https://vulnhub.com/entry/metasploitable-2,29/)**

The important thing is that we do not run every enumeration command we know against every port. We start from the scan result and follow the protocol:

| **Ports**            | **Service** | **What We Want To Learn**                                                   |
| -------------------- | ----------- | --------------------------------------------------------------------------- |
| TCP 21               | FTP         | Banner, supported commands, anonymous access, files and possible usernames. |
| TCP 3306             | MySQL       | Version, authentication, databases, tables, users and account privileges.   |
| TCP 139/445          | SMB         | Host information, shares, users, groups and anonymous or guest access.      |
| TCP 80               | HTTP        | Server technology, headers, methods, directories, files and virtual hosts.  |
| TCP/UDP 111 and 2049 | NFS         | Exported directories and whether they can be mounted and read.              |

## Enumerating FTP

**FTP (File Transfer Protocol)** uses TCP ports 20 and 21. One of the first checks is whether the server permits **anonymous login**, where the username is `anonymous` and the server accepts any password.

FTP separates the connection used for commands from the connection used to transfer data. Port 21 is the control side: the client connects, authenticates and sends commands such as `USER`, `PASS` and `PASV`. Port 20 is traditionally associated with the data side, which carries directory listings and files. This matters during enumeration because a working connection to port 21 only proves that we can speak to the FTP server. We still need to learn how it is configured and what the account can access.

### Confirm The FTP Service

Start by confirming the service and its banner:

```bash
nmap -sV -p 21 TARGET_IP
```

![probing ftp service with nmap](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/1.webp)

We can also connect directly with Netcat:

```bash
nc TARGET_IP 21
```

An FTP server normally responds before authentication, which can reveal the product or version behind the port. We can then issue the protocol commands manually:

```plaintext
USER anonymous
PASS anonymous
PASV
```

![authenticating to FTP server with netcat](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/2.webp)

`USER` supplies the account name, `PASS` attempts authentication and `PASV` asks the server to use passive mode for the data connection. Doing this once by hand makes the later tool output easier to understand.

### Check Anonymous Access

Nmap can test this with the `ftp-anon` script:

```
nmap -p 21 --script=ftp-anon.nse TARGET_IP
```

![ftp anonymous access nmap scan](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/3.webp)

We can also connect with the FTP client:

```
ftp TARGET_IP
```

When prompted, we enter `anonymous` as the username and supply a password. If the login works, our next question is not simply “did we get in?” We check what the account can **read** and what it can **write**. Anonymous access to an empty, read-only location is very different from anonymous access to sensitive files or a writable directory.

Inside the FTP client, start with a directory listing. Walk through any returned directories and record whether files can be downloaded. If the server exposes a writable directory, that is a separate and more important finding than read-only access.

```plaintext
ls
get FILE_NAME
put TEST_FILE
```

`ls` lists the current directory, `get` downloads a selected file and `put` tests whether the current directory accepts an upload. Only use a harmless test file for the write check.

We can also run all available ftp scripts provided by nmap for a broader coverage:

```
nmap -p 21 --script ftp-* TARGET_IP
```

![running all nmap ftp related scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/4.webp)

![running all nmap ftp related scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/5.webp)

The wildcard is useful for broad coverage, but the output should still be read script by script. One result may describe anonymous access, another may return the banner, and so on.

### Enumerate Possible FTP Users And Passwords

FTP responses can also be used to test whether account names exist. The [`ftp-user-enum`](https://pentestmonkey.net/tools/user-enumeration/ftp-user-enum) script takes a username wordlist and checks each entry against the service. **NMAP**'s `ftp-brute` script can also do the same:

```bash
nmap --script ftp-brute --script-args userdb=usernames.txt -p 21 TARGET_IP
```

This is useful when usernames have already been gathered during reconnaissance or exposed by another service. At this stage, the goal is to produce a smaller list of possible accounts, not to start guessing passwords.

Once a candidate list of usernames exists, the same `ftp-brute` script can pair it with a password wordlist to attempt authentication:

```bash
nmap --script ftp-brute --script-args userdb=usernames.txt,passdb=passwords.txt -p 21 TARGET_IP
```

Other common tools for this stage include `hydra` and `medusa`:

```bash
hydra -L usernames.txt -P passwords.txt ftp://TARGET_IP
```

![bruteforcing ftp creds with hydra](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/6.webp)

```bash
medusa -h TARGET_IP -U usernames.txt -P passwords.txt -M ftp
```

By the end of FTP enumeration, we should be able to answer:
- Which FTP product and version is responding?
- Is anonymous authentication enabled?
- Which directories and files can the anonymous account read?
- Can the account write anywhere?
- Which FTP features are enabled?
- Did the service reveal any valid usernames?

To learn more about FTP:
- https://en.wikipedia.org/wiki/File_Transfer_Protocol
- https://www.geeksforgeeks.org/computer-networks/file-transfer-protocol-ftp-in-application-layer/

## Enumerating MySQL

**MySQL** is a relational database management system that commonly listens on TCP port 3306. Applications connect to the database server, authenticate with a MySQL account and issue queries. The server organizes information into databases, each database contains tables, and those tables contain rows and columns.

This makes MySQL especially useful to understand as a beginner. Web applications frequently depend on a database for accounts, application settings and other stored information. During enumeration, we want to learn:

- Which MySQL version is running?
- Does the server accept remote connections?
- Are any accounts using an empty password?
- Which usernames can be identified?
- Which databases and tables can an available account access?
- What privileges does that account have?

The TCP scan already confirmed that port 3306 is open on our lab VM, so we can move directly into MySQL-specific checks.

### Confirm The MySQL Service

Start by confirming the service and version:

```bash
nmap -sV -p 3306 TARGET_IP
```

Nmap's `mysql-info` script requests information from the initial MySQL handshake:

```bash
nmap -p 3306 --script=mysql-info TARGET_IP
```

![probing mysql service with nmap](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/7.webp)

The result can confirm that the service really is MySQL and identify details such as the protocol and server version. This is more useful than relying on the port number alone because any application can technically listen on any port.

### Check Accounts And Empty Passwords

Nmap provides scripts for two useful authentication checks:

```bash
nmap -p 3306 --script=mysql-empty-password,mysql-enum TARGET_IP
```

- `mysql-empty-password` checks whether the server accepts an account with an empty password.
- `mysql-enum` attempts to identify valid MySQL usernames from differences in the server's responses.

![running account related nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/8.webp)

An empty password is a configuration weakness rather than a software exploit, but the effect can be just as serious. If a privileged database account accepts remote authentication without a password, anyone who can reach the port may be able to access the data allowed to that account.

### Connect With The MySQL Client

If the empty-password check succeeds for the root account, connect without `-p`:

```bash
mysql -h TARGET_IP -u root
```

For an account that requires a password, add `-p` so the client prompts for it:

```bash
mysql -h TARGET_IP -u USERNAME -p
```

- `-h` specifies the database server.
- `-u` specifies the MySQL username.
- `-p` asks for the account password.

![connecting to the sql server](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/9.webp)
> The previous Nmap scan revealed that the root account had an empty password, so we omit the `-p` flag.

A successful connection changes the type of enumeration we can perform. Before authentication, we are limited to information exposed by the handshake and login behavior. After authentication, our visibility depends on the privileges assigned to the account.

### Identify The Server And Available Databases

MySQL statements end with a semicolon. Start with a few queries that describe the current session and server:

```sql
status;
SELECT VERSION();
SELECT @@hostname;
SHOW DATABASES;
```

- `status` displays information about the current client connection.
- `SELECT VERSION()` returns the database version.
- `SELECT @@hostname` returns the server hostname.
- `SHOW DATABASES` lists the databases visible to the current account.

The database list is permission-dependent. An account may see every database, only the application database it needs, or almost nothing.

![running sql commands](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/10.webp)

### Inspect Databases And Tables

Choose one database from the result:

```sql
USE DATABASE_NAME;
SHOW TABLES;
```

`USE` selects the database, while `SHOW TABLES` lists the tables visible inside it. Once an interesting table is found, inspect its structure before dumping its contents:

```sql
DESCRIBE TABLE_NAME;
SELECT * FROM TABLE_NAME LIMIT 10;
```

`DESCRIBE` reveals the column names and types. The `LIMIT 10` clause keeps the first inspection small instead of returning every row. In the lab, table names and columns help us understand which application uses the database and what kind of information it stores.

### Enumerate MySQL Users And Privileges

If the current account is allowed to read the MySQL system database, list the configured accounts and the hosts from which they are permitted to connect:

```sql
SELECT user, host FROM mysql.user;
```

![enumerating users](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/11.webp)

MySQL treats the username and allowed host together as an account. This is why an account may work locally but fail remotely, or be accepted only from a particular system.

Next, check the privileges assigned to the current session:

```sql
SHOW GRANTS;
```

![enumerating permissions](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/12.webp)

Privileges determine which databases, tables and operations the account can access. If either query returns an access-denied error, that result is still useful because it identifies the current account's boundary.

When finished, leave the MySQL client with:

```sql
exit;
```

By the end of MySQL enumeration, we should know:

- The MySQL product and version running on port 3306.
- Whether remote authentication is possible.
- Whether an account accepts an empty password.
- Which usernames, databases and tables are exposed.
- Which hostname the database reports.
- Which privileges belong to the connected account.

MySQL is a good example of why enumeration must go beyond reading a banner. The port and version tell us what service exists, but authentication and SQL queries show what the service actually exposes.

To learn more about MySQL:
- https://dev.mysql.com/doc/

## Enumerating SMB

**SMB** commonly runs on TCP ports 445 and 139 and is used for network communication and file sharing. SMB and NetBIOS are independent protocols, but NetBIOS over TCP is often enabled for compatibility, so their enumeration is frequently combined.

An SMB server publishes named resources called **shares**. A client connects to the server, establishes a session and asks which shares or other information it is allowed to access. Some servers require a username and password immediately, while others expose information through null or guest sessions.

That gives SMB enumeration several layers. We first identify the host and protocol, then list shares, then determine which session types work, and finally inspect the contents allowed by that session. Finding a share does not automatically mean we can read it, and being able to read it does not automatically mean we can write to it.

The information we care about first is:
- the target's hostname
- available shares
- null or guest access
- users and groups
- readable files inside a share

### Identify The Host And SMB Service

`nmblookup` can query the NetBIOS names associated with the VM:

```bash
nmblookup -A TARGET_IP
```

![querying netbios names with nmblookup](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/13.webp)

This gives us a hostname to compare with the name returned by Nmap, enum4linux or MySQL.

Nmap can then run its SMB-related scripts against both common ports:

```bash
nmap -sV --script=smb-*.nse -p 139,445 TARGET_IP
```

![running smb related nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/14.webp)

![running smb related nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/15.webp)

![running smb related nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/16.webp)

![running smb related nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/17.webp)

This keeps the workflow connected to the [scanning blog](https://ziomsec.com/blog/hacking-fundamentals-scanning/): the same scanner that found SMB now hands the service to scripts that know how to query it.

### List Shares

**smbclient** can ask the target for its share list:

```bash
smbclient -L TARGET_IP
```

![listing smb shares](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/18.webp)

**smbmap** can test a null username:

```bash
smbmap -H TARGET_IP -u ""
```

![testing for null session](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/19.webp)

A **null session** connects without credentials. A **guest session** uses the service's guest access. If either succeeds, we can try accessing the returned shares.

For a broad automated pass, **enum4linux** combines several SMB enumeration checks:

```bash
enum4linux -A TARGET_IP
```

If credentials are available, it also supports authenticated enumeration:

```bash
enum4linux -u <username> -p <password> TARGET_IP
```

The value of enum4linux is not merely that it runs several checks at once. Its output brings the host, workgroup/domain, users, groups and shares into one place, which makes it easier to compare findings from the other SMB tools.

### Query SMB Through RPC

`rpcclient` gives us an interactive way to query information exposed through RPC. We can first test a null session:

```bash
rpcclient -U "" -N TARGET_IP
```

If the session opens, a few useful queries are:

```plaintext
enumdomusers
enumdomgroups
netshareenumall
querygroupmem GROUP_RID
```

- `enumdomusers` requests the users the service exposes.
- `enumdomgroups` requests the available groups.
- `netshareenumall` requests the share list.
- `querygroupmem` uses a group's RID to request its members.

These results should be compared rather than treated independently. A username returned by RPC may also appear in a file inside a share, while a share found by enum4linux can be opened with smbclient.

### Connect To A Share

Once a share name and valid username are known, connect with:

```bash
smbclient //TARGET_IP/SHARE_NAME -U USERNAME
```

If the share allows anonymous access, try the same connection without supplying a password:

```bash
smbclient //TARGET_IP/SHARE_NAME -N
```

![connecting to an smb share](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/20.webp)

Inside the client, `ls` lists files. If there are many files to retrieve from the share, smbclient can recurse through them:

```shell
recurse on
prompt off
mget *
```

The important part is the progression:

```plaintext
Ports 139/445 are open -> SMB scripts identify the service -> List shares -> Check null, guest or authenticated access -> Inspect readable files
```

We do not need to jump straight to an exploit. A share name, a username or a carelessly exposed file may be the most useful finding on the host.

Before moving on, we should be able to answer:
- Which hostname and workgroup/domain does SMB report?
- Which SMB ports and versions are available?
- Do null or guest sessions work?
- Which shares exist, and which of them are readable or writable?
- Which users and groups were exposed?
- Do any files reveal names or information that can be used while enumerating another service?

To learn more about SMB:
- https://www.geeksforgeeks.org/computer-networks/introduction-to-microsoft-smb-a-network-file-sharing-protocol/
- https://www.ibm.com/docs/en/aix/7.3.0?topic=management-smb-protocol
- https://www.geeksforgeeks.org/ethical-hacking/smb-enumeration/

## Enumerating HTTP And HTTPS

HTTP and HTTPS commonly run on TCP ports 80 and 443. Web enumeration focuses on the technologies behind a site, hidden directories and files, virtual hosts, subdomains and server misconfigurations.

This overlaps with the reconnaissance post, but the interaction is different. Passive recon searched public sources for known web assets. Here we send requests directly to the web server to discover what it serves.

HTTP follows a request-and-response model. The client sends a request containing a method, a path and headers. The server returns a response containing headers and a body. Enumeration changes one piece of that request at a time: the path for directory discovery, the `Host` header for virtual-host discovery, or the request itself when inspecting supported behavior.

For the Metasploitable 2 VM, we begin with the IP address found during scanning. Every new directory or application we discover becomes another target to inspect instead of treating port 80 as one single page.

### Build An Initial Web Profile

Start with the response and the technology stack:

```bash
whatweb http://TARGET_IP
curl -v http://TARGET_IP
```

![viewing http response](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/21.webp)

**Nikto** can then check the web server for misconfigurations and outdated software:

```bash
nikto -h http://TARGET_IP
```

![performing nikto scan](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/22.webp)

Nmap's HTTP scripts can add more information. For example, the sitemap generator attempts to crawl the web service and return the paths it finds:

```bash
nmap -p 80 --script=http-sitemap-generator TARGET_IP
```

![generating sitemap with nmap](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/23.webp)

It is also worth requesting the two files commonly intended for crawlers:

```bash
curl http://TARGET_IP/robots.txt
curl http://TARGET_IP/sitemap.xml
```

These commands create a starting profile, but they do not tell us every path the site contains. We still need active content discovery.

### Discover Directories And Files

Web fuzzing replaces a marker with every entry in a wordlist and requests the resulting paths. With **ffuf**, the marker is `FUZZ`:

```bash
ffuf -u http://TARGET_IP/FUZZ -w <wordlist>
```

If the wordlist contains `admin`, `test` and `www`, ffuf tests:

```plaintext
http://TARGET_IP/admin
http://TARGET_IP/test
http://TARGET_IP/www
```

![discovering hidden files and directories with ffuf](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/24.webp)

**Gobuster** performs the same kind of directory discovery:

```bash
gobuster dir -u http://TARGET_IP -w <wordlist> -t 64
```

![discovering hidden files and directories with gobuster](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/25.webp)

The `-r` switch follows redirects, while `-x` lets us search for particular file types such as `.php`, `.js` or `.asp`. If generic wordlists are not finding target-specific names, **Cewl** can build one from the site's own content:

```bash
cewl -d 2 -m 5 -w words.txt http://TARGET_IP
```

That wordlist can then be passed back into ffuf or Gobuster.

### Virtual Host Enumeration

One IP can serve different websites according to the HTTP `Host` header. This branch only becomes useful when reconnaissance, MySQL, SMB or the website itself gives us a hostname or domain. If that name does not exist in public DNS, we first map it locally in `/etc/hosts`:

```plaintext
TARGET_IP    TARGET_DOMAIN
```

That entry only resolves the exact base domain. It does not automatically create mappings for names such as `admin.TARGET_DOMAIN` or `test.TARGET_DOMAIN`.

We can still test those names by fuzzing the `Host` header:

```bash
ffuf -u http://TARGET_DOMAIN \
  -H "Host: FUZZ.TARGET_DOMAIN" \
  -w WORDLIST
```

For every word, ffuf sends a request to the base target but changes the header:

```http
Host: admin.TARGET_DOMAIN
Host: test.TARGET_DOMAIN
```

The server uses that header to choose which virtual host to return. Valid vhosts would return different response which could then be filtered out.

Putting `FUZZ` directly in the URL is less useful in this setup:

```bash
ffuf -u http://FUZZ.TARGET_DOMAIN -w <wordlist>
```

That version asks the operating system to resolve every generated hostname. Unless DNS or `/etc/hosts` contains each name, the request cannot reach the server. Fuzzing the `Host` header avoids creating a separate local entry for every guess.

Before leaving HTTP, record:
- The server and technologies identified.
- Interesting response headers and any server banner.
- Paths returned by the crawler, `robots.txt`, `sitemap.xml` and the wordlist tools.
- The status of discovered directories and files.
- Any hostname or virtual host that needs its own enumeration pass.
- Target-specific words that can be turned into a wordlist.

To learn more about HTTP:
- https://developer.mozilla.org/en-US/docs/Web/HTTP
- https://en.wikipedia.org/wiki/HTTP

## Enumerating NFS

The all-port scan can reveal **NFS (Network File System)** on TCP or UDP port 2049, often alongside RPC services on port 111. NFS allows a server to make directories available across a network. A client mounts an exported directory into its own filesystem and works with it like a local path.

The terminology is important:
- An **export** is a directory the NFS server makes available.
- A **mount point** is the local directory where we attach that export.
- The export rules determine which clients are allowed to mount it.
- Normal file ownership and permissions still determine what can be read or written after it is mounted.

### Confirm NFS And RPC

Start by confirming the two ports identified by the scan:

```bash
nmap -sV -p 111,2049 TARGET_IP
sudo nmap -sU -sV -p 111,2049 TARGET_IP
```

Port 111 helps clients locate RPC-based services, while port 2049 is associated with NFS itself. Checking both TCP and UDP prevents us from overlooking the service because we only followed the TCP result.

![scanning udp and tcp nfs ports with nmap](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/26.webp)

### List Exported Directories

`showmount` asks the server which directories it exports:

```bash
showmount -e TARGET_IP
```

![listing exported directories on the target](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/27.webp)

The output gives us two pieces of information: the exported path and the clients or network ranges allowed to mount it. An export visible to our lab machine becomes the next object to verify.

### Mount And Inspect An Export

Create an empty local directory for the mount point:

```bash
mkdir /tmp/nfs-share
```

Then mount the exported path returned by `showmount`:

```bash
sudo mount -t nfs TARGET_IP:/EXPORTED_PATH /tmp/nfs-share -o nolock
```

Once mounted, list the files together with their ownership and permissions:

```bash
ls -la /tmp/nfs-share
```

![viewing contents inside the exported directory](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/28.webp)

Do not stop at the fact that the mount succeeded. Walk through the exposed directories and answer the same questions we used for FTP and SMB:
- Which files and directories are readable?
- Does the export expose user or application data?
- Which owners and groups appear in the listing?
- Is any part of the export writable?

File access may be restricted even when the export can be mounted. The directory listing helps explain those restrictions because NFS access is still affected by the ownership and permission information attached to the files.

When the check is complete, unmount it:

```bash
sudo umount /tmp/nfs-share
```

NFS gives us a useful comparison with SMB. Both can expose files over a network, but they use different protocols and tools. The scan determines which one is present; the enumeration process then lists the shared resource, tests access and inspects its contents.

## Vulnerability Analysis

At this point, we have a large amount of information about the target. Scanning told us which ports were open, while enumeration showed us how the services were configured and what they exposed. **Vulnerability analysis** is where we evaluate those findings to determine which ones represent an actual weakness.

This distinction is important. An open port is not automatically a vulnerability, an old-looking version is not automatically exploitable, and a successful tool result still needs to be understood. The goal is to turn raw observations into findings that can be explained and verified.

### What Counts As A Vulnerability

The weaknesses discovered during this phase commonly fall into a few categories:

- **Misconfiguration**: A service has been configured in an unsafe way, such as allowing anonymous access to sensitive files.
- **Default installation**: Features, sample applications or settings from the original installation remain enabled.
- **Default or empty passwords**: An account still uses a known default password or no password at all.
- **Missing patches**: The operating system or application is missing security updates.
- **Design flaws**: A weakness exists in how the software handles data, authentication or another security-sensitive operation.
- **Operating-system flaws**: The underlying operating system contains a weakness that affects the service or host.

Several enumeration results can point towards these categories:

| **Enumeration Result**                         | **Possible Weakness To Investigate**                                                       |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------ |
| FTP permits anonymous access                   | Sensitive files may be readable, or a directory may be writable without a named account.   |
| MySQL accepts an empty password                | A database account may be accessible without meaningful authentication.                    |
| SMB accepts a null or guest session            | Shares, users or files may be exposed without valid credentials.                           |
| HTTP reveals outdated software or unsafe files | The web server or application may contain known vulnerabilities or configuration mistakes. |
| NFS exports a directory to our system          | Files may be exposed with permissions that allow unintended reading or writing.            |

### Build A Findings Inventory

Go back through the saved scan output and the enumeration results from each service. For every interesting observation, record:

- the affected IP address and port
- the service and detected version
- the command or request that produced the result
- the access required to reproduce it
- the information or capability that was exposed
- the possible security impact
- any evidence needed to verify the finding later

For example, `port 21 is open` is only a scan result. `Anonymous FTP authentication succeeds and the account can download a sensitive file` is a potential vulnerability because it describes the access, the exposed resource and the impact.

### Use Vulnerability Scanners Carefully

Nmap can run NSE scripts categorized for vulnerability detection:

```bash
sudo nmap -sV -p 21,80,139,445,3306 --script vuln TARGET_IP
```

![running vulnerability scan nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/29.webp)

![running vulnerability scan nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/30.webp)

![running vulnerability scan nmap scripts](https://cdn.ziomsec.com/hacking-fundamentals-enumeration/31.webp)

This combines version detection with vulnerability-related scripts for the selected services. The result can highlight known weaknesses and service-specific checks worth investigating further.

Web services can also be checked with Nikto, which we used earlier:

```bash
nikto -h http://TARGET_IP
```

Larger vulnerability scanners such as **Nessus** and **OpenVAS** can assess multiple services and organize the results by severity. Their reports are useful for coverage, but they should not replace manual validation.

Automated scanners can produce two important types of mistakes:
- A **false positive** occurs when a scanner reports a vulnerability that is not actually present. This can happen when service detection is inaccurate or when a security patch has been backported without changing the reported software version.
- A **false negative** occurs when the scanner fails to identify a vulnerability that is present. This is more dangerous because it can create a false sense of security.

Treat scanner output as a list of claims to investigate. It is not proof by itself.

### CVE And CVSS

Publicly disclosed vulnerabilities are commonly assigned a **CVE**, which provides a standardized identifier that can be used across advisories, scanners and vulnerability databases. The **National Vulnerability Database (NVD)** adds information about these publicly disclosed vulnerabilities.

The **Common Vulnerability Scoring System (CVSS)** assigns a severity score from 0 to 10:

| **Score** | **Severity** |
| --------- | ------------ |
| 0.0       | None         |
| 0.1-3.9   | Low          |
| 4.0-6.9   | Medium       |
| 7.0-8.9   | High         |
| 9.0-10.0  | Critical     |

The exact product and version gathered during enumeration can be compared with applicable CVEs. The CVSS score then helps prioritize the results, but it does not replace validation. We still need to confirm that the target uses the affected version and configuration.

### Validate The Lab Findings

In our lab, we revisit each service with a specific question:
1. **FTP**: Does anonymous authentication work? What can the account read or write?
2. **MySQL**: Does an account accept an empty password? Which databases and privileges become available?
3. **SMB**: Does a null or guest session work? Which shares and files are accessible?
4. **HTTP**: Which server findings from Nikto can be reproduced manually? Which discovered directories expose sensitive or unsafe content?
5. **NFS**: Which directories are exported? Can they be mounted, and what do their permissions allow?

For each potential vulnerability, we reproduce the result with the smallest relevant command and save the evidence. If the result cannot be reproduced, we investigate why before carrying it forward. The target may not meet the vulnerable conditions, the scanner may have identified the service incorrectly, or a defensive control may be changing the response.

The output of vulnerability analysis should be a shortlist of **validated weaknesses**, each tied to a service, evidence and possible impact. That shortlist becomes the input for exploitation.

## Conclusion

That covers enumeration and vulnerability analysis, marking the end of Phase 2. We took five scan results from our lab and followed each one:
- FTP led to anonymous-access and read/write checks.
- MySQL led to version, authentication, database, table, user and privilege information.
- SMB led to null/guest sessions, shares and files.
- HTTP led to technology detection, hidden paths and virtual hosts.
- NFS led to exported directories, mounted shares and filesystem permissions.

The central lesson is that enumeration is **service-driven**. Scanning gives us ports; enumeration turns those ports into hostnames, users, shares, files, directories and other details that can shape an attack path. Vulnerability analysis then separates ordinary information from weaknesses that can be validated and prioritized. Just as importantly, the services feed into one another: a MySQL hostname or database name can guide web enumeration, an SMB username can guide FTP checks, and files exposed through SMB or NFS may reveal where to look next.

In the next post, we will move into **Phase 3: Exploitation**, where we use the validated weaknesses discovered during Phase 2 to gain access.

---