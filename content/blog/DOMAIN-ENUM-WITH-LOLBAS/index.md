---
title: "Domain Enumeration With LOLBas"
date: 2026-01-07
draft: false
summary: "Enumerating Active Directory Using Native Tools."
tags: ["living Off The Land", "Active Directory"]
categories: ["blog"]
series: ["lolbas"]
showToc: true
---

On a recent client engagement, I found myself in a Windows environment with nothing but `cmd.exe` - no Powershell, no imported modules, no fancy tooling. That got me exploring how far I could go with domain enumeration using just native Windows binaries and inspired me to write this blog. This blog covers what's possible using `cmd.exe` and `powershell` without the AD module.

## Enumerating Current Domain

### Identifying The Domain And SID

**COMMAND PROMPT**
- We can query the Windows environment variable:

```cmd
C:\Users\Administrator>echo %USERDOMAIN%
cs

C:\Users\Administrator>echo %USERDNSDOMAIN%
CS.ORG
```

- `systeminfo` is a windows command that gathers system data from Windows Registry, WMI an OS configs. This can also be used to get the Domain name:

```cmd
C:\Users\Administrator>systeminfo | findstr /B /C:"Domain"
Domain:                    cs.org
```

- The Domain SID can be extracted using the `whoami` command. The `SID` format is `S-Revision-IdentifierAuthority-SubAuthorities-RID`. The Last number identifies the object inside the domain and 

```cmd
C:\Users\Administrator>whoami /user

USER INFORMATION
----------------
User Name        SID
================ ============================================
cs\administrator S-1-5-21-2854037470-815493089-3575834450-500
```
> Here, the SID is `S-1-5-21-2854037470-815493089-3575834450`

**POWERSHELL**
- Using Powershell, we can enumerate the domain by connecting to the **`RootDSE`**, and getting the Domain DN

```powershell
PS C:\Users\Administrator> $root = [ADSI]"LDAP://RootDSE"
>> $domainDN = $root.defaultNamingContext
>> $domainDN
DC=cs,DC=org
```

- We can enumerate the forest info aswell from the `RootDSE`

```powershell
PS C:\Users\Administrator> $root = [ADSI]"LDAP://RootDSE"
>> $forestRoot = $root.rootDomainNamingContext
>> $forestRoot
DC=cs,DC=org
```

- We can extract the domain SID from the above domain object.

```powershell
PS C:\Users\Administrator> $root = [ADSI]"LDAP://RootDSE"
PS C:\Users\Administrator> $domainDN = $root.defaultNamingContext
PS C:\Users\Administrator> $domain = [ADSI]"LDAP://$domainDN"
PS C:\Users\Administrator> (New-Object System.Security.Principal.SecurityIdentifier(
>>     $domain.objectSid[0], 0
>> )).Value
S-1-5-21-2854037470-815493089-3575834450
```

### Identifying Domain Controllers

**COMMAND PROMPT**
- `nltest` is a Windows domain diagnostic tool used to query domain infrastructure. The `%USERDOMAIN%` is the environment variable containing the NetBIOS domain name. The `dclist` gets all domain controllers for our domain.

```cmd
C:\Users\Administrator>nltest /dclist:%USERDOMAIN%
Get list of DCs in domain 'cs' from '\\WIN-O8VCKD4SEGV'.
    WIN-O8VCKD4SEGV.cs.org [PDC]  [DS] Site: Default-First-Site-Name
The command completed successfully
```

- alternatively, we can use `dsgetdc` to get which domain controller is used for authentication. This queries DNS SRV records like `_ldap_._tcp.dc._msdcs.cs.org`

```cmd
C:\Users\Administrator>nltest /dsgetdc:%USERDOMAIN%
           DC: \\WIN-O8VCKD4SEGV
      Address: \\192.168.200.179
     Dom Guid: fb7b1327-e6cc-4b61-8218-6b5a31539239
     Dom Name: cs
  Forest Name: cs.org
 Dc Site Name: Default-First-Site-Name
Our Site Name: Default-First-Site-Name
        Flags: PDC GC DS LDAP KDC TIMESERV GTIMESERV WRITABLE DNS_FOREST CLOSE_SITE FULL_SECRET WS DS_8 DS_9 DS_10 KEYLIST
The command completed successfully
```

**POWERSHELL**
- We can create an LDAP search object to query AD. We can then find objects that are computers using LDAP filters by matching the `userAccountControl` flag.

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.Filter = "(&(objectClass=computer)(userAccountControl:1.2.840.113556.1.4.803:=8192))"
>> $searcher.PropertiesToLoad.Add("dnshostname") | Out-Null
>>
>> $searcher.FindAll() | ForEach-Object {
>>     $_.Properties.dnshostname
>> }
WIN-O8VCKD4SEGV.cs.org
```

- Alternatively, we can query AD replication objects. We can use the `RootDSE` to query the configuration container to find domain controllers.

```powershell
PS C:\Users\Administrator> $root = [ADSI]"LDAP://RootDSE"
>> $configDN = $root.configurationNamingContext
>> $domainDN = $root.defaultNamingContext
>>
>> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.SearchRoot = [ADSI]"LDAP://$configDN"
>> $searcher.Filter = "(objectClass=nTDSDSA)"
>> $searcher.PropertiesToLoad.Add("distinguishedName") | Out-Null
>>
>> $searcher.FindAll() | ForEach-Object {
>>     $_.Properties.distinguishedname -replace '^CN=NTDS Settings,CN=',''
>> }
WIN-O8VCKD4SEGV,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=cs,DC=org
```

### Enumerating The Password Policy

- To enumerate the password policy, we can use the `net` command utility. This interacts with the Windows networking services and domain infrastructure.

```cmd
C:\Users\Administrator>net accounts /domain
Force user logoff how long after time expires?:       Never
Minimum password age (days):                          1
Maximum password age (days):                          42
Minimum password length:                              4
Length of password history maintained:                24
Lockout threshold:                                    Never
Lockout duration (minutes):                           1
Lockout observation window (minutes):                 1
Computer role:                                        PRIMARY
The command completed successfully.
```

### Enumerating Domain Computers

**COMMAND PROMPT**
- We can use the ADDS administration tools to perform an LDAP query to list computer objects in the domain:

```cmd
C:\Users\Administrator>dsquery computer
"CN=WIN-O8VCKD4SEGV,OU=Domain Controllers,DC=cs,DC=org"
```

**POWERSHELL**
- Similar to CMD, we can use LDAP queries via `.NET` directory services to get computer account names, OS installed, description and their last auths:

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.Filter = "(objectClass=computer)"
>> $searcher.PropertiesToLoad.AddRange(@(
>>     "samaccountname",
>>     "operatingSystem",
>>     "description",
>>     "lastLogon"
>> )) | Out-Null
>>
>> $searcher.FindAll() | ForEach-Object {
>>     [PSCustomObject]@{
>>         Name        = $_.Properties.samaccountname
>>         OS          = $_.Properties.operatingsystem
>>         Description = $_.Properties.description
>>         LastLogon   = $_.Properties.lastlogon
>>     }
>> }

Name               OS                                        Description LastLogon
----               --                                        ----------- ---------
{WIN-O8VCKD4SEGV$} {Windows Server 2022 Standard Evaluation}             {134172015865651952}
{http_svc$}                                                              {0}
{mssql_svc$}                                                             {0}
{exchange_svc$}                                                          {0}
```

## Finding Users And Groups

### Domain Groups

**COMMAND PROMPT**
- We can use `dsquery` that uses directory service command-line utilities

```cmd
C:\Users\Administrator>dsquery group
"CN=Administrators,CN=Builtin,DC=cs,DC=org"
"CN=Users,CN=Builtin,DC=cs,DC=org"
"CN=Guests,CN=Builtin,DC=cs,DC=org"
"CN=Print Operators,CN=Builtin,DC=cs,DC=org"
"CN=Backup Operators,CN=Builtin,DC=cs,DC=org"
"CN=Replicator,CN=Builtin,DC=cs,DC=org"
"CN=Remote Desktop Users,CN=Builtin,DC=cs,DC=org"
"CN=Network Configuration Operators,CN=Builtin,DC=cs,DC=org"
"CN=Performance Monitor Users,CN=Builtin,DC=cs,DC=org"
"CN=Performance Log Users,CN=Builtin,DC=cs,DC=org"
"CN=Distributed COM Users,CN=Builtin,DC=cs,DC=org"
"CN=IIS_IUSRS,CN=Builtin,DC=cs,DC=org"
"CN=Cryptographic Operators,CN=Builtin,DC=cs,DC=org"
"CN=Event Log Readers,CN=Builtin,DC=cs,DC=org"
"CN=Certificate Service DCOM Access,CN=Builtin,DC=cs,DC=org"
"CN=RDS Remote Access Servers,CN=Builtin,DC=cs,DC=org"
"CN=RDS Endpoint Servers,CN=Builtin,DC=cs,DC=org"
"CN=RDS Management Servers,CN=Builtin,DC=cs,DC=org"
"CN=Hyper-V Administrators,CN=Builtin,DC=cs,DC=org"
"CN=Access Control Assistance Operators,CN=Builtin,DC=cs,DC=org"
"CN=Remote Management Users,CN=Builtin,DC=cs,DC=org"
"CN=Storage Replica Administrators,CN=Builtin,DC=cs,DC=org"
"CN=Domain Computers,CN=Users,DC=cs,DC=org"
"CN=Domain Controllers,CN=Users,DC=cs,DC=org"
"CN=Schema Admins,CN=Users,DC=cs,DC=org"
"CN=Enterprise Admins,CN=Users,DC=cs,DC=org"
"CN=Cert Publishers,CN=Users,DC=cs,DC=org"
"CN=Domain Admins,CN=Users,DC=cs,DC=org"
"CN=Domain Users,CN=Users,DC=cs,DC=org"
"CN=Domain Guests,CN=Users,DC=cs,DC=org"
"CN=Group Policy Creator Owners,CN=Users,DC=cs,DC=org"
"CN=RAS and IAS Servers,CN=Users,DC=cs,DC=org"
"CN=Server Operators,CN=Builtin,DC=cs,DC=org"
"CN=Account Operators,CN=Builtin,DC=cs,DC=org"
"CN=Pre-Windows 2000 Compatible Access,CN=Builtin,DC=cs,DC=org"
"CN=Incoming Forest Trust Builders,CN=Builtin,DC=cs,DC=org"
"CN=Windows Authorization Access Group,CN=Builtin,DC=cs,DC=org"
"CN=Terminal Server License Servers,CN=Builtin,DC=cs,DC=org"
"CN=Allowed RODC Password Replication Group,CN=Users,DC=cs,DC=org"
"CN=Denied RODC Password Replication Group,CN=Users,DC=cs,DC=org"
"CN=Read-only Domain Controllers,CN=Users,DC=cs,DC=org"
"CN=Enterprise Read-only Domain Controllers,CN=Users,DC=cs,DC=org"
"CN=Cloneable Domain Controllers,CN=Users,DC=cs,DC=org"
"CN=Protected Users,CN=Users,DC=cs,DC=org"
"CN=Key Admins,CN=Users,DC=cs,DC=org"
"CN=Enterprise Key Admins,CN=Users,DC=cs,DC=org"
"CN=DnsAdmins,CN=Users,DC=cs,DC=org"
"CN=DnsUpdateProxy,CN=Users,DC=cs,DC=org"
"CN=Office Admin,CN=Users,DC=cs,DC=org"
"CN=IT Admins,CN=Users,DC=cs,DC=org"
"CN=Executives,CN=Users,DC=cs,DC=org"
"CN=Senior management,CN=Users,DC=cs,DC=org"
"CN=Project management,CN=Users,DC=cs,DC=org"
"CN=marketing,CN=Users,DC=cs,DC=org"
"CN=sales,CN=Users,DC=cs,DC=org"
"CN=accounting,CN=Users,DC=cs,DC=org"
```

- Alternatively, we can use the Windows `NetAPI` using the `net` command to get the groups present in the domain

```cmd 
C:\Users\Administrator>net group /domain

Group Accounts for \\WIN-O8VCKD4SEGV
-------------------------------------------------------------------------------
*accounting
*Cloneable Domain Controllers
*DnsUpdateProxy
*Domain Admins
*Domain Computers
*Domain Controllers
*Domain Guests
*Domain Users
*Enterprise Admins
*Enterprise Key Admins
*Enterprise Read-only Domain Controllers
*Executives
*Group Policy Creator Owners
*IT Admins
*Key Admins
*marketing
*Office Admin
*Project management
*Protected Users
*Read-only Domain Controllers
*sales
*Schema Admins
*Senior management
The command completed successfully.
```

- To find users part of a particular group, we can use the `net`  utility

```cmd
C:\Users\Administrator>net group "sales" /domain
Group name     sales
Comment

Members

-------------------------------------------------------------------------------
adriaens.nanni           babita.berget            bobette.henrietta
brenda.edeline           corabel.malia            cynthie.linus
katey.malvina
The command completed successfully.
```

**POWERSHELL**
- We can use an LDAP search object with a filter to get the groups and print the `samaccountname` and `distinguishedname`.

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
PS C:\Users\Administrator> $searcher.Filter = "(objectClass=group)"
PS C:\Users\Administrator> $searcher.PropertiesToLoad.AddRange(@(
>>     "samaccountname",
>>     "distinguishedname"
>> )) | Out-Null
PS C:\Users\Administrator> $searcher.FindAll() | ForEach-Object {
>>     [PSCustomObject]@{
>>         Name = $_.Properties.samaccountname
>>         DN   = $_.Properties.distinguishedname
>>     }
>> }

Name                                      DN
----                                      --
{Administrators}                          {CN=Administrators,CN=Builtin,DC=cs,DC=org}
{Users}                                   {CN=Users,CN=Builtin,DC=cs,DC=org}
{Guests}                                  {CN=Guests,CN=Builtin,DC=cs,DC=org}
{Print Operators}                         {CN=Print Operators,CN=Builtin,DC=cs,DC=org}
{Backup Operators}                        {CN=Backup Operators,CN=Builtin,DC=cs,DC=org}
{Replicator}                              {CN=Replicator,CN=Builtin,DC=cs,DC=org}
{Remote Desktop Users}                    {CN=Remote Desktop Users,CN=Builtin,DC=cs,DC=org}
{Network Configuration Operators}         {CN=Network Configuration Operators,CN=Builtin,DC=cs,DC=org}
{Performance Monitor Users}               {CN=Performance Monitor Users,CN=Builtin,DC=cs,DC=org}
{Performance Log Users}                   {CN=Performance Log Users,CN=Builtin,DC=cs,DC=org}
{Distributed COM Users}                   {CN=Distributed COM Users,CN=Builtin,DC=cs,DC=org}
{IIS_IUSRS}                               {CN=IIS_IUSRS,CN=Builtin,DC=cs,DC=org}
{Cryptographic Operators}                 {CN=Cryptographic Operators,CN=Builtin,DC=cs,DC=org}
{Event Log Readers}                       {CN=Event Log Readers,CN=Builtin,DC=cs,DC=org}
{Certificate Service DCOM Access}         {CN=Certificate Service DCOM Access,CN=Builtin,DC=cs,DC=org}
{RDS Remote Access Servers}               {CN=RDS Remote Access Servers,CN=Builtin,DC=cs,DC=org}
{RDS Endpoint Servers}                    {CN=RDS Endpoint Servers,CN=Builtin,DC=cs,DC=org}
{RDS Management Servers}                  {CN=RDS Management Servers,CN=Builtin,DC=cs,DC=org}
{Hyper-V Administrators}                  {CN=Hyper-V Administrators,CN=Builtin,DC=cs,DC=org}
{Access Control Assistance Operators}     {CN=Access Control Assistance Operators,CN=Builtin,DC=cs,DC=org}
{Remote Management Users}                 {CN=Remote Management Users,CN=Builtin,DC=cs,DC=org}
{Storage Replica Administrators}          {CN=Storage Replica Administrators,CN=Builtin,DC=cs,DC=org}
{Domain Computers}                        {CN=Domain Computers,CN=Users,DC=cs,DC=org}
{Domain Controllers}                      {CN=Domain Controllers,CN=Users,DC=cs,DC=org}
{Schema Admins}                           {CN=Schema Admins,CN=Users,DC=cs,DC=org}
{Enterprise Admins}                       {CN=Enterprise Admins,CN=Users,DC=cs,DC=org}
{Cert Publishers}                         {CN=Cert Publishers,CN=Users,DC=cs,DC=org}
{Domain Admins}                           {CN=Domain Admins,CN=Users,DC=cs,DC=org}
{Domain Users}                            {CN=Domain Users,CN=Users,DC=cs,DC=org}
{Domain Guests}                           {CN=Domain Guests,CN=Users,DC=cs,DC=org}
{Group Policy Creator Owners}             {CN=Group Policy Creator Owners,CN=Users,DC=cs,DC=org}
{RAS and IAS Servers}                     {CN=RAS and IAS Servers,CN=Users,DC=cs,DC=org}
{Server Operators}                        {CN=Server Operators,CN=Builtin,DC=cs,DC=org}
{Account Operators}                       {CN=Account Operators,CN=Builtin,DC=cs,DC=org}
{Pre-Windows 2000 Compatible Access}      {CN=Pre-Windows 2000 Compatible Access,CN=Builtin,DC=cs,DC=org}
{Incoming Forest Trust Builders}          {CN=Incoming Forest Trust Builders,CN=Builtin,DC=cs,DC=org}
{Windows Authorization Access Group}      {CN=Windows Authorization Access Group,CN=Builtin,DC=cs,DC=org}
{Terminal Server License Servers}         {CN=Terminal Server License Servers,CN=Builtin,DC=cs,DC=org}
{Allowed RODC Password Replication Group} {CN=Allowed RODC Password Replication Group,CN=Users,DC=cs,DC=org}
{Denied RODC Password Replication Group}  {CN=Denied RODC Password Replication Group,CN=Users,DC=cs,DC=org}
{Read-only Domain Controllers}            {CN=Read-only Domain Controllers,CN=Users,DC=cs,DC=org}
{Enterprise Read-only Domain Controllers} {CN=Enterprise Read-only Domain Controllers,CN=Users,DC=cs,DC=org}
{Cloneable Domain Controllers}            {CN=Cloneable Domain Controllers,CN=Users,DC=cs,DC=org}
{Protected Users}                         {CN=Protected Users,CN=Users,DC=cs,DC=org}
{Key Admins}                              {CN=Key Admins,CN=Users,DC=cs,DC=org}
{Enterprise Key Admins}                   {CN=Enterprise Key Admins,CN=Users,DC=cs,DC=org}
{DnsAdmins}                               {CN=DnsAdmins,CN=Users,DC=cs,DC=org}
{DnsUpdateProxy}                          {CN=DnsUpdateProxy,CN=Users,DC=cs,DC=org}
{Office Admin}                            {CN=Office Admin,CN=Users,DC=cs,DC=org}
{IT Admins}                               {CN=IT Admins,CN=Users,DC=cs,DC=org}
{Executives}                              {CN=Executives,CN=Users,DC=cs,DC=org}
{Senior management}                       {CN=Senior management,CN=Users,DC=cs,DC=org}
{Project management}                      {CN=Project management,CN=Users,DC=cs,DC=org}
{marketing}                               {CN=marketing,CN=Users,DC=cs,DC=org}
{sales}                                   {CN=sales,CN=Users,DC=cs,DC=org}
{accounting}                              {CN=accounting,CN=Users,DC=cs,DC=org}
```

- Similarly, we can use the `LDAP` object to list members part of a particular group. In the below example, I listed members part of the `sales` group

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.Filter = "(&(objectClass=group)(name=sales))"
>> $searcher.PropertiesToLoad.Add("member") | Out-Null
>> $group = $searcher.FindOne()
>> $group.Properties.member

CN=Bobette Henrietta,CN=Users,DC=cs,DC=org
CN=Katey Malvina,CN=Users,DC=cs,DC=org
CN=Corabel Malia,CN=Users,DC=cs,DC=org
CN=Adriaens Nanni,CN=Users,DC=cs,DC=org
CN=Babita Berget,CN=Users,DC=cs,DC=org
CN=Cynthie Linus,CN=Users,DC=cs,DC=org
CN=Brenda Edeline,CN=Users,DC=cs,DC=org
```

### Domain Users

**COMMAND PROMPT**
- We can use `dsquery` to perform an `LDAP` search against the domain controller to return all user objects.

```cmd
C:\Users\Administrator>dsquery user
"CN=Administrator,CN=Users,DC=cs,DC=org"
"CN=Guest,CN=Users,DC=cs,DC=org"
"CN=krbtgt,CN=Users,DC=cs,DC=org"
"CN=Brenda Edeline,CN=Users,DC=cs,DC=org"
"CN=Allina Florencia,CN=Users,DC=cs,DC=org"
"CN=Carlee Risa,CN=Users,DC=cs,DC=org"
"CN=Ferne Elyse,CN=Users,DC=cs,DC=org"
"CN=Elsbeth Etty,CN=Users,DC=cs,DC=org"
"CN=Jill Avie,CN=Users,DC=cs,DC=org"
"CN=Robby Minni,CN=Users,DC=cs,DC=org"
"CN=Katrinka Florance,CN=Users,DC=cs,DC=org"
"CN=Norma Antonia,CN=Users,DC=cs,DC=org"
"CN=Aileen Pippy,CN=Users,DC=cs,DC=org"
"CN=Letta Gertrud,CN=Users,DC=cs,DC=org"
"CN=Cynthie Linus,CN=Users,DC=cs,DC=org"
"CN=Britteny Guenevere,CN=Users,DC=cs,DC=org"
"CN=Joannes Eugenia,CN=Users,DC=cs,DC=org"
"CN=Cecily Josee,CN=Users,DC=cs,DC=org"
"CN=Leena Aviva,CN=Users,DC=cs,DC=org"
"CN=Deonne Sabrina,CN=Users,DC=cs,DC=org"
"CN=Angelia Luz,CN=Users,DC=cs,DC=org"
"CN=Natka Alene,CN=Users,DC=cs,DC=org"
"CN=Malynda Meredithe,CN=Users,DC=cs,DC=org"
"CN=Belvia Sigrid,CN=Users,DC=cs,DC=org"
"CN=Babita Berget,CN=Users,DC=cs,DC=org"
"CN=Lori Lottie,CN=Users,DC=cs,DC=org"
...
...
...
"CN=Gwendolin Emogene,CN=Users,DC=cs,DC=org"
"CN=Cherlyn Amabel,CN=Users,DC=cs,DC=org"
"CN=Carma Lowe,CN=Users,DC=cs,DC=org"
"CN=Felisha Orly,CN=Users,DC=cs,DC=org"
"CN=Aila Livvy,CN=Users,DC=cs,DC=org"
"CN=Dione Lay,CN=Users,DC=cs,DC=org"
Dsquery has reached the default limit of 100 results to display; use the -limit option to display more results.
```

- Instead of `LDAP`, we can also utilize Windows `NetAPI` to get the `samaccountnames` of the users.

```cmd
C:\Users\Administrator>net users /domain

User accounts for \\WIN-O8VCKD4SEGV

-------------------------------------------------------------------------------
Administrator            adriaens.nanni           aigneis.chrysa
aila.livvy               aileen.pippy             aliza.rene
allegra.ginni            allina.florencia         amargo.elizabeth
andriana.mella           angelia.luz              anitra.delia
ansley.kenton            aprilette.pauly          ariadne.kelci
babita.berget            belvia.sigrid            bernice.amara
bobette.henrietta        brenda.edeline           britt.aeriela
britte.hephzibah         britteny.guenevere       britteny.rosina
cammie.roslyn            carlee.risa              carma.lowe
carmina.imelda           cecily.josee             cherlyn.amabel
clio.adelaida            consuelo.fania           corabel.malia
corri.brandice           cynthie.linus            dale.annecorinne
darya.clarette           denni.demetra            deonne.sabrina
dione.lay                dynah.gigi               elsbeth.etty
etty.lita                evie.murial              felisha.orly
ferne.elyse              fidelia.aliza            florence.mariquilla
gabriela.eleanora        george.annissa           Guest
gunilla.annelise         gwendolin.emogene        haleigh.katrinka
hayley.kirk              heidie.belva             hollyanne.salome
ivonne.karel             jeanne.charity           jeri.faye
jerrie.cindelyn          jill.avie                joana.honoria
joannes.eugenia          joelie.lenora            joella.christine
katey.malvina            kati.adel                katrinka.florance
krbtgt                   lanie.juieta             leanna.marice
leena.aviva              letitia.meagan           letta.gertrud
lilia.koenraad           linzy.gene               lisa.agnes
loren.matelda            lori.lottie              lorne.ricki
lorri.damaris            lydia.rebe               malynda.meredithe
martelle.emeline         maryanne.josy            maryellen.silvia
merissa.gerladina        mimi.dorothy             mirabel.honor
modestia.judye           natka.alene              nessi.jeanne
nicolina.dulcea          nisse.louie              nolie.elisabet
norma.antonia            quintilla.bea            rennie.gale
robby.minni              sapphira.fanny           susana.angel
The command completed successfully.
```


**POWERSHELL**
- With powershell, we can use a `.NET` LDAP search object and perform custom LDAP queries on it. In the below command, I listed objects with class `user` and category `person`. This filter avoids computer objects.

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.Filter = "(&(objectClass=user)(objectCategory=person))"
>> $searcher.PropertiesToLoad.Add("samaccountname") | Out-Null
>>
>> $searcher.FindAll() | ForEach-Object {
>>     $_.Properties.samaccountname
>> }
Administrator
Guest
krbtgt
brenda.edeline
allina.florencia
carlee.risa
ferne.elyse
elsbeth.etty
jill.avie
robby.minni
katrinka.florance
norma.antonia
aileen.pippy
letta.gertrud
cynthie.linus
britteny.guenevere
joannes.eugenia
cecily.josee
leena.aviva
deonne.sabrina
angelia.luz
natka.alene
malynda.meredithe
belvia.sigrid
babita.berget
lori.lottie
adriaens.nanni
bernice.amara
florence.mariquilla
...
...
...
martelle.emeline
jerrie.cindelyn
nisse.louie
andriana.mella
carmina.imelda
joella.christine
modestia.judye
linzy.gene
katey.malvina
lorne.ricki
bobette.henrietta
clio.adelaida
george.annissa
merissa.gerladina
aigneis.chrysa
mimi.dorothy
nolie.elisabet
heidie.belva
gwendolin.emogene
cherlyn.amabel
carma.lowe
felisha.orly
aila.livvy
dione.lay
joelie.lenora
allegra.ginni
```

- similarly, we can also list privileged accounts

```powershell
PS C:\Users\Administrator> $searcher = [ADSISearcher]"(&(objectCategory=person)(objectClass=user)(adminCount=1))"
>> $searcher.FindAll() | ForEach-Object {
>>     $_.Properties.samaccountname
>> }
Administrator
krbtgt
```

### Enumerate As-Rep Roastable Accounts

We can enumerate users that have Kerberos pre-authentication disabled (as-rep roastable) using an ldap filter: `(userAccountControl:1.2.840.113556.1.4.803:=4194304)`

**COMMAND PROMPT**
- we can use `csvde` to export the result of the `ldap` filter to find as-rep roastable users and save them in a file called `nopreauth.csv`:

```cmd
C:\Users\Administrator>csvde -f nopreauth.csv -r "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" -l "sAMAccountName"
Connecting to "(null)"
Logging in as current user using SSPI
Exporting directory to file nopreauth.csv
Searching for entries...
Writing out entries
..
Export Completed. Post-processing in progress...
2 entries exported

The command has completed successfully

C:\Users\Administrator>type nopreauth.csv
DN,sAMAccountName
"CN=Bernice Amara,CN=Users,DC=cs,DC=org",bernice.amara
"CN=Britte Hephzibah,CN=Users,DC=cs,DC=org",britte.hephzibah
```

- Alternatively, we can also use the Directory Services command line tool to query using LDAP filters.

```cmd
C:\Users\Administrator>dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" -attr sAMAccountName
  sAMAccountName
  bernice.amara
  britte.hephzibah
```

**POWERSHELL**
- The same ldap search is now performed using `.NET`

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.Filter = "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))"
>> $searcher.PropertiesToLoad.Add("samaccountname") | Out-Null
>>
>> $searcher.FindAll() | ForEach-Object {
>>     $_.Properties.samaccountname
>> }
bernice.amara
britte.hephzibah
```

### Enumerate Kerberoastable Accounts

**COMMAND PROMPT**
- The simplest way to query SPNs through cmd is using `setspn`. This is a tool used to register, delete and query SPNs.

```cmd
C:\Users\Administrator>setspn -Q */*
Checking domain DC=cs,DC=org
CN=WIN-O8VCKD4SEGV,OU=Domain Controllers,DC=cs,DC=org
        Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/WIN-O8VCKD4SEGV.cs.org
        ldap/WIN-O8VCKD4SEGV.cs.org/ForestDnsZones.cs.org
        ldap/WIN-O8VCKD4SEGV.cs.org/DomainDnsZones.cs.org
        DNS/WIN-O8VCKD4SEGV.cs.org
        GC/WIN-O8VCKD4SEGV.cs.org/cs.org
        RestrictedKrbHost/WIN-O8VCKD4SEGV.cs.org
        RestrictedKrbHost/WIN-O8VCKD4SEGV
        RPC/3bf8aa9c-d5fa-4be9-a063-95b2a7ba495b._msdcs.cs.org
        HOST/WIN-O8VCKD4SEGV/cs
        HOST/WIN-O8VCKD4SEGV.cs.org/cs
        HOST/WIN-O8VCKD4SEGV
        HOST/WIN-O8VCKD4SEGV.cs.org
        HOST/WIN-O8VCKD4SEGV.cs.org/cs.org
        E3514235-4B06-11D1-AB04-00C04FC2DCD2/3bf8aa9c-d5fa-4be9-a063-95b2a7ba495b/cs.org
        ldap/WIN-O8VCKD4SEGV/cs
        ldap/3bf8aa9c-d5fa-4be9-a063-95b2a7ba495b._msdcs.cs.org
        ldap/WIN-O8VCKD4SEGV.cs.org/cs
        ldap/WIN-O8VCKD4SEGV
        ldap/WIN-O8VCKD4SEGV.cs.org
        ldap/WIN-O8VCKD4SEGV.cs.org/cs.org
CN=krbtgt,CN=Users,DC=cs,DC=org
        kadmin/changepw
CN=http_svc,CN=Managed Service Accounts,DC=cs,DC=org
        http_svc/httpserver.cs.org
CN=mssql_svc,CN=Managed Service Accounts,DC=cs,DC=org
        mssql_svc/mssqlserver.cs.org
CN=exchange_svc,CN=Managed Service Accounts,DC=cs,DC=org
        exchange_svc/exserver.cs.org

Existing SPN found!
```

- Similar to `asreproastable` account enumeration, we can also use `csvde` to run ldap queries to get kerberoastable accounts and save the output in a csv file.

```cmd
C:\Users\Administrator>csvde -f out.csv -r "(servicePrincipalName=*)" -l sAMAccountName,servicePrincipalName
Connecting to "(null)"
Logging in as current user using SSPI
Exporting directory to file out.csv
Searching for entries...
Writing out entries
.....
Export Completed. Post-processing in progress...
5 entries exported

The command has completed successfully

C:\Users\Administrator>type out.csv
DN,sAMAccountName,servicePrincipalName
"CN=WIN-O8VCKD4SEGV,OU=Domain Controllers,DC=cs,DC=org",WIN-O8VCKD4SEGV$,Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/WIN-O8VCKD4SEGV.cs.org;ldap/WIN-O8VCKD4SEGV.cs.org/ForestDnsZones.cs.org;ldap/WIN-O8VCKD4SEGV.cs.org/DomainDnsZones.cs.org;DNS/WIN-O8VCKD4SEGV.cs.org;GC/WIN-O8VCKD4SEGV.cs.org/cs.org;RestrictedKrbHost/WIN-O8VCKD4SEGV.cs.org;RestrictedKrbHost/WIN-O8VCKD4SEGV;RPC/3bf8aa9c-d5fa-4be9-a063-95b2a7ba495b._msdcs.cs.org;HOST/WIN-O8VCKD4SEGV/cs;HOST/WIN-O8VCKD4SEGV.cs.org/cs;HOST/WIN-O8VCKD4SEGV;HOST/WIN-O8VCKD4SEGV.cs.org;HOST/WIN-O8VCKD4SEGV.cs.org/cs.org;E3514235-4B06-11D1-AB04-00C04FC2DCD2/3bf8aa9c-d5fa-4be9-a063-95b2a7ba495b/cs.org;ldap/WIN-O8VCKD4SEGV/cs;ldap/3bf8aa9c-d5fa-4be9-a063-95b2a7ba495b._msdcs.cs.org;ldap/WIN-O8VCKD4SEGV.cs.org/cs;ldap/WIN-O8VCKD4SEGV;ldap/WIN-O8VCKD4SEGV.cs.org;ldap/WIN-O8VCKD4SEGV.cs.org/cs.org
"CN=exchange_svc,CN=Managed Service Accounts,DC=cs,DC=org",exchange_svc$,exchange_svc/exserver.cs.org
"CN=http_svc,CN=Managed Service Accounts,DC=cs,DC=org",http_svc$,http_svc/httpserver.cs.org
"CN=krbtgt,CN=Users,DC=cs,DC=org",krbtgt,kadmin/changepw
"CN=mssql_svc,CN=Managed Service Accounts,DC=cs,DC=org",mssql_svc$,mssql_svc/mssqlserver.cs.org
```

- Another tool that allows running `ldap` queries is `dsquery`.

```cmd
C:\Users\Administrator>dsquery * -filter "(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))" -attr sAMAccountName servicePrincipalName
  sAMAccountName    servicePrincipalName
  krbtgt            kadmin/changepw
```

**POWERSHELL**
- With powershell, we can create an LDAP search object and use the same filter as `dsquery` to find accounts that have SPNs.

```powershell
PS C:\Users\Administrator> $searcher = New-Object DirectoryServices.DirectorySearcher
>> $searcher.Filter = "(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))"
>> $searcher.PropertiesToLoad.AddRange(@("samaccountname","serviceprincipalname")) | Out-Null
>>
>> $searcher.FindAll() | ForEach-Object {
>>     [PSCustomObject]@{
>>         User = $_.Properties.samaccountname
>>         SPNs = $_.Properties.serviceprincipalname
>>     }
>> }

User     SPNs
----     ----
{krbtgt} {kadmin/changepw}
```

## Enumerating ACLs

> This one's a bit of a hassle and analyzing these results will take more time. Instead, these outputs can be formatted in a way that they can be used by Bloodhound.

- We can enumerate `GenericWrite` perms using Powershell. Here, the `IdentityReference` has these permissions over the objects in in `ObjectDN`.

```powershell
PS C:\Users\Administrator> ([ADSISearcher]"(objectClass=*)").FindAll() | ForEach-Object {
>>     $Object = $_.GetDirectoryEntry()
>>     $ACLs = $Object.ObjectSecurity.Access
>>
>>     foreach ($ACE in $ACLs) {
>>         if ($ACE.ActiveDirectoryRights -match "GenericWrite") {
>>             [PSCustomObject]@{
>>                 ObjectDN  = $Object.distinguishedName
>>                 Identity  = $ACE.IdentityReference
>>                 Inherited = $ACE.IsInherited
>>             }
>>         }
>>     }
>> }

ObjectDN                                                                         Identity                                   Inherited
--------                                                                         --------                                   ---------
{CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                                         NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS     False
{CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                                         cs\DnsAdmins                                   False
{DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                       NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS     False
{DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                       cs\DnsAdmins                                    True
{DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                       NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=@,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                  NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=@,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org}                  cs\DnsAdmins                                    True
{DC=a.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=a.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=b.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=b.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=c.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=c.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=d.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=d.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=e.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=e.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=f.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=f.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=g.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=g.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=h.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=h.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=i.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=i.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=j.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=j.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=k.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=k.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=l.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=l.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{DC=m.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS      True
{DC=m.root-servers.net,DC=RootDNSServers,CN=MicrosoftDNS,CN=System,DC=cs,DC=org} cs\DnsAdmins                                    True
{CN=Executives,CN=Users,DC=cs,DC=org}                                            cs\Project management                          False
{CN=Project management,CN=Users,DC=cs,DC=org}                                    cs\marketing                                   False
```

- Enumerating `WriteProperty` where the `IdentityReference` has the permission over the `object DN`

```powershell
PS C:\Users\Administrator> ([ADSISearcher]"(objectClass=*)").FindAll() | ForEach-Object {
>>     $Object = $_.GetDirectoryEntry()
>>     $ACLs = $Object.ObjectSecurity.Access
>>
>>     foreach ($ACE in $ACLs) {
>>         if ($ACE.ActiveDirectoryRights -match "WriteProperty") {
>>             [PSCustomObject]@{
>>                 ObjectDN  = $Object.distinguishedName
>>                 Identity  = $ACE.IdentityReference
>>                 Inherited = $ACE.IsInherited
>>             }
>>         }
>>     }
>> }

ObjectDN                                    Identity               Inherited
--------                                    --------               ---------
{DC=cs,DC=org}                              BUILTIN\Administrators     False
{DC=cs,DC=org}                              cs\Domain Admins           False
{DC=cs,DC=org}                              NT AUTHORITY\SELF          False
{DC=cs,DC=org}                              NT AUTHORITY\SELF          False
{DC=cs,DC=org}                              NT AUTHORITY\SELF          False
{CN=Users,DC=cs,DC=org}                     cs\Domain Admins           False
{CN=Users,DC=cs,DC=org}                     NT AUTHORITY\SELF           True
{CN=Users,DC=cs,DC=org}                     NT AUTHORITY\SELF           True
{CN=Users,DC=cs,DC=org}                     NT AUTHORITY\SELF           True
{CN=Users,DC=cs,DC=org}                     BUILTIN\Administrators      True
{CN=Computers,DC=cs,DC=org}                 cs\Domain Admins           False
{CN=Computers,DC=cs,DC=org}                 NT AUTHORITY\SELF           True
{CN=Computers,DC=cs,DC=org}                 NT AUTHORITY\SELF           True
{CN=Computers,DC=cs,DC=org}                 NT AUTHORITY\SELF           True
{CN=Computers,DC=cs,DC=org}                 BUILTIN\Administrators      True
{OU=Domain Controllers,DC=cs,DC=org}        cs\Domain Admins           False
{OU=Domain Controllers,DC=cs,DC=org}        NT AUTHORITY\SELF           True
{OU=Domain Controllers,DC=cs,DC=org}        NT AUTHORITY\SELF           True
{OU=Domain Controllers,DC=cs,DC=org}        NT AUTHORITY\SELF           True
{OU=Domain Controllers,DC=cs,DC=org}        BUILTIN\Administrators      True
{CN=System,DC=cs,DC=org}                    cs\Domain Admins           False
{CN=System,DC=cs,DC=org}                    NT AUTHORITY\SELF           True
{CN=System,DC=cs,DC=org}                    NT AUTHORITY\SELF           True
{CN=System,DC=cs,DC=org}                    NT AUTHORITY\SELF           True
{CN=System,DC=cs,DC=org}                    BUILTIN\Administrators      True
{CN=LostAndFound,DC=cs,DC=org}              cs\Domain Admins           False
{CN=LostAndFound,DC=cs,DC=org}              NT AUTHORITY\SELF           True
{CN=LostAndFound,DC=cs,DC=org}              NT AUTHORITY\SELF           True
{CN=LostAndFound,DC=cs,DC=org}              NT AUTHORITY\SELF           True
...
```

- Enumerating `WriteDACL`

```powershell
PS C:\Users\Administrator> ([ADSISearcher]"(objectClass=*)").FindAll() | ForEach-Object {
>>     $Object = $_.GetDirectoryEntry()
>>     $ACLs = $Object.ObjectSecurity.Access
>>
>>     foreach ($ACE in $ACLs) {
>>         if ($ACE.ActiveDirectoryRights -match "WriteDacl") {
>>             [PSCustomObject]@{
>>                 ObjectDN  = $Object.distinguishedName
>>                 Identity  = $ACE.IdentityReference
>>                 Inherited = $ACE.IsInherited
>>             }
>>         }
>>     }
>> }

ObjectDN                                    Identity               Inherited
--------                                    --------               ---------
{DC=cs,DC=org}                              BUILTIN\Administrators     False
{DC=cs,DC=org}                              cs\Domain Admins           False
{CN=Users,DC=cs,DC=org}                     cs\Domain Admins           False
{CN=Users,DC=cs,DC=org}                     BUILTIN\Administrators      True
{CN=Computers,DC=cs,DC=org}                 cs\Domain Admins           False
{CN=Computers,DC=cs,DC=org}                 BUILTIN\Administrators      True
{OU=Domain Controllers,DC=cs,DC=org}        cs\Domain Admins           False
{OU=Domain Controllers,DC=cs,DC=org}        BUILTIN\Administrators      True
{CN=System,DC=cs,DC=org}                    cs\Domain Admins           False
{CN=System,DC=cs,DC=org}                    BUILTIN\Administrators      True
{CN=LostAndFound,DC=cs,DC=org}              cs\Domain Admins           False
{CN=LostAndFound,DC=cs,DC=org}              BUILTIN\Administrators      True
{CN=Infrastructure,DC=cs,DC=org}            cs\Domain Admins           False
{CN=Infrastructure,DC=cs,DC=org}            BUILTIN\Administrators      True
{CN=ForeignSecurityPrincipals,DC=cs,DC=org} BUILTIN\Administrators      True
{CN=Program Data,DC=cs,DC=org}              BUILTIN\Administrators      True
{CN=Microsoft,CN=Program Data,DC=cs,DC=org} BUILTIN\Administrators      True
{CN=NTDS Quotas,DC=cs,DC=org}               BUILTIN\Administrators      True
{CN=Managed Service Accounts,DC=cs,DC=org}  cs\Domain Admins           False
{CN=Managed Service Accounts,DC=cs,DC=org}  BUILTIN\Administrators      True
{CN=WinsockServices,CN=System,DC=cs,DC=org} cs\Domain Admins           False
{CN=WinsockServices,CN=System,DC=cs,DC=org} BUILTIN\Administrators      True
{CN=RpcServices,CN=System,DC=cs,DC=org}     cs\Domain Admins           False
{CN=RpcServices,CN=System,DC=cs,DC=org}     BUILTIN\Administrators      True
{CN=FileLinks,CN=System,DC=cs,DC=org}       cs\Domain Admins           False
{CN=FileLinks,CN=System,DC=cs,DC=org}       BUILTIN\Administrators      True
{CN=VolumeTable,CN=FileLinks,CN=System,D... BUILTIN\Administrators     False
{CN=VolumeTable,CN=FileLinks,CN=System,D... cs\Domain Admins           False
{CN=ObjectMoveTable,CN=FileLinks,CN=Syst... cs\Domain Admins           False
{CN=ObjectMoveTable,CN=FileLinks,CN=Syst... BUILTIN\Administrators      True
{CN=Default Domain Policy,CN=System,DC=c... cs\Domain Admins           False
{CN=Default Domain Policy,CN=System,DC=c... BUILTIN\Administrators      True
{CN=AppCategories,CN=Default Domain Poli... cs\Domain Admins           False
{CN=AppCategories,CN=Default Domain Poli... BUILTIN\Administrators      True
{CN=Meetings,CN=System,DC=cs,DC=org}        cs\Domain Admins           False
{CN=Meetings,CN=System,DC=cs,DC=org}        BUILTIN\Administrators      True
{CN=Policies,CN=System,DC=cs,DC=org}        cs\Domain Admins           False
{CN=Policies,CN=System,DC=cs,DC=org}        BUILTIN\Administrators      True
{CN={31B2F340-016D-11D2-945F-00C04FB984F... CREATOR OWNER              False
{CN={31B2F340-016D-11D2-945F-00C04FB984F... NT AUTHORITY\SYSTEM        False
{CN={31B2F340-016D-11D2-945F-00C04FB984F... cs\Domain Admins           False
{CN={31B2F340-016D-11D2-945F-00C04FB984F... cs\Domain Admins           False
{CN={31B2F340-016D-11D2-945F-00C04FB984F... cs\Domain Admins           False
{CN={31B2F340-016D-11D2-945F-00C04FB984F... cs\Enterprise Admins       False
{CN={31B2F340-016D-11D2-945F-00C04FB984F... cs\Enterprise Admins       False
{CN=User,CN={31B2F340-016D-11D2-945F-00C... cs\Domain Admins           False
{CN=User,CN={31B2F340-016D-11D2-945F-00C... cs\Domain Admins            True
{CN=User,CN={31B2F340-016D-11D2-945F-00C... cs\Enterprise Admins        True
{CN=User,CN={31B2F340-016D-11D2-945F-00C... CREATOR OWNER               True
{CN=User,CN={31B2F340-016D-11D2-945F-00C... NT AUTHORITY\SYSTEM         True
{CN=Machine,CN={31B2F340-016D-11D2-945F-... cs\Domain Admins           False
{CN=Machine,CN={31B2F340-016D-11D2-945F-... cs\Domain Admins            True
{CN=Machine,CN={31B2F340-016D-11D2-945F-... cs\Enterprise Admins        True
```

- Enumerating `GenericAll`

```powershell
PS C:\Users\Administrator> ([ADSISearcher]"(objectClass=*)").FindAll() | ForEach-Object {
>>     $Object = $_.GetDirectoryEntry()
>>     $ACLs = $Object.ObjectSecurity.Access
>>
>>     foreach ($ACE in $ACLs) {
>>         if ($ACE.ActiveDirectoryRights -match "GenericAll") {
>>             [PSCustomObject]@{
>>                 ObjectDN  = $Object.distinguishedName
>>                 Identity  = $ACE.IdentityReference
>>                 Inherited = $ACE.IsInherited
>>             }
>>         }
>>     }
>> }

ObjectDN                                    Identity             Inherited
--------                                    --------             ---------
{DC=cs,DC=org}                              NT AUTHORITY\SYSTEM      False
{DC=cs,DC=org}                              cs\Enterprise Admins     False
{CN=Users,DC=cs,DC=org}                     NT AUTHORITY\SYSTEM      False
{CN=Users,DC=cs,DC=org}                     cs\Enterprise Admins      True
{CN=Computers,DC=cs,DC=org}                 NT AUTHORITY\SYSTEM      False
{CN=Computers,DC=cs,DC=org}                 cs\Enterprise Admins      True
{OU=Domain Controllers,DC=cs,DC=org}        NT AUTHORITY\SYSTEM      False
{OU=Domain Controllers,DC=cs,DC=org}        cs\Enterprise Admins      True
{CN=System,DC=cs,DC=org}                    NT AUTHORITY\SYSTEM      False
{CN=System,DC=cs,DC=org}                    cs\Enterprise Admins      True
{CN=LostAndFound,DC=cs,DC=org}              NT AUTHORITY\SYSTEM      False
{CN=LostAndFound,DC=cs,DC=org}              cs\Enterprise Admins      True
{CN=Infrastructure,DC=cs,DC=org}            NT AUTHORITY\SYSTEM      False
{CN=Infrastructure,DC=cs,DC=org}            cs\Enterprise Admins      True
{CN=ForeignSecurityPrincipals,DC=cs,DC=org} NT AUTHORITY\SYSTEM      False
{CN=ForeignSecurityPrincipals,DC=cs,DC=org} cs\Domain Admins         False
{CN=ForeignSecurityPrincipals,DC=cs,DC=org} cs\Enterprise Admins      True
{CN=Program Data,DC=cs,DC=org}              NT AUTHORITY\SYSTEM      False
{CN=Program Data,DC=cs,DC=org}              cs\Domain Admins         False
{CN=Program Data,DC=cs,DC=org}              cs\Enterprise Admins      True
{CN=Microsoft,CN=Program Data,DC=cs,DC=org} NT AUTHORITY\SYSTEM      False
{CN=Microsoft,CN=Program Data,DC=cs,DC=org} cs\Domain Admins         False
{CN=Microsoft,CN=Program Data,DC=cs,DC=org} cs\Enterprise Admins      True
{CN=NTDS Quotas,DC=cs,DC=org}               cs\Domain Admins         False
{CN=NTDS Quotas,DC=cs,DC=org}               cs\Enterprise Admins      True
{CN=Managed Service Accounts,DC=cs,DC=org}  NT AUTHORITY\SYSTEM      False
{CN=Managed Service Accounts,DC=cs,DC=org}  cs\Enterprise Admins      True
{CN=Keys,DC=cs,DC=org}                      NT AUTHORITY\ENTE...     False
{CN=Keys,DC=cs,DC=org}                      NT AUTHORITY\SYSTEM      False
{CN=Keys,DC=cs,DC=org}                      cs\Domain Admins         False
{CN=Keys,DC=cs,DC=org}                      cs\Domain Control...     False
{CN=Keys,DC=cs,DC=org}                      cs\Enterprise Admins     False
{CN=Keys,DC=cs,DC=org}                      cs\Key Admins            False
{CN=Keys,DC=cs,DC=org}                      cs\Enterprise Key...     False
{CN=WinsockServices,CN=System,DC=cs,DC=org} NT AUTHORITY\SYSTEM      False
{CN=WinsockServices,CN=System,DC=cs,DC=org} cs\Enterprise Admins      True
{CN=RpcServices,CN=System,DC=cs,DC=org}     NT AUTHORITY\SYSTEM      False
{CN=RpcServices,CN=System,DC=cs,DC=org}     cs\Enterprise Admins      True
{CN=FileLinks,CN=System,DC=cs,DC=org}       NT AUTHORITY\SYSTEM      False
{CN=FileLinks,CN=System,DC=cs,DC=org}       cs\Enterprise Admins      True
{CN=VolumeTable,CN=FileLinks,CN=System,D... NT AUTHORITY\SYSTEM      False
{CN=VolumeTable,CN=FileLinks,CN=System,D... cs\Enterprise Admins     False
{CN=ObjectMoveTable,CN=FileLinks,CN=Syst... NT AUTHORITY\SYSTEM      False
{CN=ObjectMoveTable,CN=FileLinks,CN=Syst... cs\Enterprise Admins      True
{CN=Default Domain Policy,CN=System,DC=c... NT AUTHORITY\SYSTEM      False
{CN=Default Domain Policy,CN=System,DC=c... cs\Enterprise Admins      True
{CN=AppCategories,CN=Default Domain Poli... NT AUTHORITY\SYSTEM      False
```

- We can also print all the ACLs regarding all objects in the domain. Here, similar to above, the `IdentityReference` is the object that has the ACL and the `ObjectDN` is the object it has access over. The relationship is: `IdentityReference  → has permission → ObjectDN`

```powershell
PS C:\Users\Administrator> ([ADSISearcher]"(objectClass=*)").FindAll() | ForEach-Object {
>>     $Object = $_.GetDirectoryEntry()
>>
>>     foreach ($ACE in $Object.ObjectSecurity.Access) {
>>         [PSCustomObject]@{
>>             ObjectName = $Object.Name
>>             ObjectDN   = $Object.distinguishedName
>>             Identity   = $ACE.IdentityReference
>>             Rights     = $ACE.ActiveDirectoryRights
>>             Type       = $ACE.AccessControlType
>>             Inherited  = $ACE.IsInherited
>>         }
>>     }
>> }


ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : Everyone
Rights     : ReadProperty
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS
Rights     : GenericRead
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : NT AUTHORITY\Authenticated Users
Rights     : GenericRead
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : NT AUTHORITY\SYSTEM
Rights     : GenericAll
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : BUILTIN\Administrators
Rights     : CreateChild, Self, WriteProperty, ExtendedRight, Delete, GenericRead, WriteDacl, WriteOwner
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : BUILTIN\Pre-Windows 2000 Compatible Access
Rights     : ListChildren
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : BUILTIN\Pre-Windows 2000 Compatible Access
Rights     : ReadProperty, ReadControl
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : cs\Domain Admins
Rights     : CreateChild, Self, WriteProperty, ExtendedRight, GenericRead, WriteDacl, WriteOwner
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : cs\Enterprise Admins
Rights     : GenericAll
Type       : Allow
Inherited  : False

ObjectName : {cs}
ObjectDN   : {DC=cs,DC=org}
Identity   : CREATOR OWNER
Rights     : Self
Type       : Allow
Inherited  : False
```

### Delegation Enumeration

- This command searches Active Directory for user accounts and computers that have unconstrained delegation enabled and prints their usernames.

```powershell
C:\Users\Administrator>dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=524288))" -attr sAMAccountName

C:\Users\Administrator>dsquery * -filter "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))" -attr dNSHostName
  dNSHostName
  WIN-O8VCKD4SEGV.cs.org
```

- The same can be done with powershell

```powershell
PS C:\Users\Administrator> ([ADSISearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=524288))").FindAll() | % { $_.Properties.samaccountname }

PS C:\Users\Administrator> ([ADSISearcher]"(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))").FindAll() | % { $_.Properties.dnshostname }
WIN-O8VCKD4SEGV.cs.org
```

- This command searches for user accounts and computers configured for constrained delegation and lists the services they can delegate to.

```powershell
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(msDS-AllowedToDelegateTo=*))" -attr sAMAccountName msDS-AllowedToDelegateTo

dsquery * -filter "(&(objectCategory=computer)(msDS-AllowedToDelegateTo=*))" -attr dNSHostName msDS-AllowedToDelegateTo
```

- The powershell alternative to find accounts with constrained delegation is:

```powershell
([ADSISearcher]"(&(objectCategory=person)(objectClass=user)(msDS-AllowedToDelegateTo=*))").FindAll() |
% {
  [PSCustomObject]@{
    User  = $_.Properties.samaccountname
    SPNs = $_.Properties.'msds-allowedtodelegateto'
  }
}

([ADSISearcher]"(&(objectCategory=computer)(msDS-AllowedToDelegateTo=*))").FindAll() |
% {
  [PSCustomObject]@{
    Computer = $_.Properties.dnshostname
    SPNs     = $_.Properties.'msds-allowedtodelegateto'
  }
}
```

- Finally, we can find RBCD. This command finds computers that allow other machines to impersonate users to them.

```powershell
dsquery * -filter "(&(objectCategory=computer)(msDS-AllowedToActOnBehalfOfOtherIdentity=*))" -attr dNSHostName msDS-AllowedToActOnBehalfOfOtherIdentity
```

- This command finds computers with RBCD enabled and extracts their delegation security descriptor.

```powershell
([ADSISearcher]"(&(objectCategory=computer)(msDS-AllowedToActOnBehalfOfOtherIdentity=*))").FindAll() |
% {
  [PSCustomObject]@{
    Computer = $_.Properties.dnshostname
    RBCD_SD  = $_.Properties.'msds-allowedtoactonbehalfofotheridentity'
  }
}

$sd = New-Object System.DirectoryServices.ActiveDirectorySecurity
$sd.SetSecurityDescriptorBinaryForm($RBCDbytes)
$sd.GetAccessRules($true,$true,[System.Security.Principal.SecurityIdentifier])
```

## Enumerating GPO & OU

**COMMAND PROMPT**
- This command lists all Organizational Units in the domain.

```cmd
C:\Users\Administrator>dsquery ou -limit 0
"OU=Domain Controllers,DC=cs,DC=org"
```

- We can list computers belonging to this OU by

```cmd
C:\Users\Administrator>dsquery computer "OU=Domain Controllers,DC=cs,DC=org"
"CN=WIN-O8VCKD4SEGV,OU=Domain Controllers,DC=cs,DC=org"
```
> Similarly, we can also list users.

- This command shows which Group Policies are applied to the current computer.

```cmd
C:\Users\Administrator>gpresult /R /SCOPE COMPUTER

Microsoft (R) Windows (R) Operating System Group Policy Result tool v2.0
© Microsoft Corporation. All rights reserved.

Created on ‎3/‎7/‎2026 at 8:04:26 AM


RSOP data for  on WIN-O8VCKD4SEGV : Logging Mode
-------------------------------------------------

OS Configuration:            Primary Domain Controller
OS Version:                  10.0.20348
Site Name:                   Default-First-Site-Name
Roaming Profile:
Local Profile:
Connected over a slow link?: No


COMPUTER SETTINGS
------------------
    CN=WIN-O8VCKD4SEGV,OU=Domain Controllers,DC=cs,DC=org
    Last time Group Policy was applied: 3/7/2026 at 8:01:53 AM
    Group Policy was applied from:      WIN-O8VCKD4SEGV.cs.org
    Group Policy slow link threshold:   500 kbps
    Domain Name:                        cs
    Domain Type:                        Windows 2008 or later

    Applied Group Policy Objects
    -----------------------------
        Default Domain Controllers Policy
        Default Domain Policy

    The following GPOs were not applied because they were filtered out
    -------------------------------------------------------------------
        Local Group Policy
            Filtering:  Not Applied (Empty)

    The computer is a part of the following security groups
    -------------------------------------------------------
        BUILTIN\Administrators
        Everyone
        BUILTIN\Users
        BUILTIN\Pre-Windows 2000 Compatible Access
        Windows Authorization Access Group
        NT AUTHORITY\NETWORK
        NT AUTHORITY\Authenticated Users
        This Organization
        WIN-O8VCKD4SEGV$
        Domain Controllers
        NT AUTHORITY\ENTERPRISE DOMAIN CONTROLLERS
        Authentication authority asserted identity
        Denied RODC Password Replication Group
        System Mandatory Level
```

**POWERSHELL**
- This PowerShell command finds all Organizational Units in Active Directory.

```powershell
PS C:\Users\Administrator> $ouSearcher = [ADSISearcher]"(&(objectClass=organizationalUnit))"
>> $ouSearcher.FindAll() | ForEach-Object { $_.Properties.name }
Domain Controllers
```

- We can find the computers belonging to the OU by

```powershell
PS C:\Users\Administrator> $ouDN = "OU=Domain Controllers,DC=cs,DC=org"
>> $searcher = [ADSISearcher]"(&(objectCategory=computer))"
>> $searcher.SearchRoot = [ADSI]"LDAP://$ouDN"
>> $searcher.FindAll() | ForEach-Object { $_.Properties.dnshostname }
WIN-O8VCKD4SEGV.cs.org
```
> To get the users, we can use the filter: `(objectCategory=person)(objectClass=user)`

## Conclusion

Enumeration is often the most critical phase of an engagement, and as we've seen, native Windows tools can get us surprisingly far (even without the AD module).  Checkout the **[lolad](https://lolad-project.github.io/)** project for more such commands and enumeration techniques along these lines.

That's it from my end, until next time!

---