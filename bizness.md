
# Machine Info

Buziness form Hackthebox involved exploiting **CVE-2023-49070** an pre-authentication Remote Code Execution (RCE) & **CVE-2023-51467** an Authentication Bypass to gain initial access on box. For root we have to crack the password hash of root.

<img alt="Pasted image 20240525125115" src="https://github.com/iammR0OT/HTB/assets/74102381/75f5da65-b6a2-4d95-b525-f9b8ead36e0b">

# User
## Scanning through nmap

First of all we will go with nmap to scan the whole network and check for services running on the network. To scan the whole network and find all the open ports I use **-p-** with **--min-rate 10000** to scan network faster from **nmap** and I found a list of open ports on the network and get only the open ports using different terminal tools like **cut**, **tr** etc. The whole command will be 

```bash
nmap -p- --min-rate 10000 10.10.11.252 -oN nmap.txt && cat nmap.txt | cut  -d ' ' -f1 | tr -d '/tcp' | tr '\n' ','
```

<img alt="Pasted image 20240107173135" src="https://github.com/iammR0OT/HTB/assets/74102381/b855cacf-2e70-4227-a7a6-c8420dff1074">

<img alt="Pasted image 20240107173203" src="https://github.com/iammR0OT/HTB/assets/74102381/75f37862-0f8f-454d-b5e5-aadb0b82ee89">

Now Let's run the depth scan on these specific ports using 

``` bash
nmap -p22,80,443,2367,4965,5461,5988,17259,21902,24500,34917,36118,39410,41968,44908,45062,53037,57225,62219,62668 -sC -sV -A -T4 10.10.11.252 -oN scan.txt
```

-  **-sC** is to run all the default scripts, 
- **-sV** for service and version detection, 
- **-A** for Enable OS detection, version detection, script scanning, and traceroute,
- **-T4** for aggressive scan 
- **-oN** to write the result into a specified file.

```nmap
PORT      STATE  SERVICE      VERSION
22/tcp    open   ssh          OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 3e:21:d5:dc:2e:61:eb:8f:a6:3b:24:2a:b7:1c:05:d3 (RSA)
|   256 39:11:42:3f:0c:25:00:08:d7:2f:1b:51:e0:43:9d:85 (ECDSA)
|_  256 b0:6f:a0:0a:9e:df:b1:7a:49:78:86:b2:35:40:ec:95 (ED25519)
80/tcp    open   http         nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Did not follow redirect to https://bizness.htb/
443/tcp   open   ssl/http     nginx 1.18.0
| tls-alpn: 
|_  http/1.1
| tls-nextprotoneg: 
|_  http/1.1
|_http-title: Did not follow redirect to https://bizness.htb/
|_ssl-date: TLS randomness does not represent time
|_http-server-header: nginx/1.18.0
| ssl-cert: Subject: organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=UK
| Not valid before: 2023-12-14T20:03:40
|_Not valid after:  2328-11-10T20:03:40
34917/tcp open   tcpwrapped
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
## Information gathering

Here we found four ports open, **22 ssh** "which can be used to if we get any kind of valid credentials to login to to the machine", **80 http** "which is a hyper text transfer protocol used for web and **ngnix1,18.0** web server is running on the backend and our requests are redirecting to **bizness.htb** which means we need to add it to our DNS file located at `/etc/hosts`", **443 https** whish is also an secure web port and the port **34917 tcpwrapped** is running. 

```bash
$ echo "10.10.11.252  bizness.htb" | sudo tee -a /etc/hosts

10.10.11.230  cozyhosting.htb
```

### Port 443 HTTPs

Port 80 redirecting us to port 443. On 443 there is a some kind of business developement website template is running.

<img  alt="Pasted image 20240525125849" src="https://github.com/iammR0OT/HTB/assets/74102381/612b8a60-c447-4b18-b8ce-4fb6dd918017">

#### Tech Stack

In tech, nothing special except the cookie value `Set-Cookie: JSESSIONID=28C4181E9E11369BEE446E6CA6A54C48.jvm1` (**JAVA virtual Machine**) which is indicating that there is some kind of JAVA based application is ruuning on backend.

```http
HTTP/1.1 200
Server: nginx/1.18.0
Date: Sat, 25 May 2024 08:10:04 GMT
Content-Type: text/html
Content-Length: 27200
Connection: keep-alive
Set-Cookie: JSESSIONID=28C4181E9E11369BEE446E6CA6A54C48.jvm1; Path=/; Secure; HttpOnly; SameSite=strict
Accept-Ranges: bytes
ETag: W/"27200-1702887508516"
Last-Modified: Mon, 18 Dec 2023 08:18:28 GMT
vary: accept-encoding
```

#### Directory Fuzzing

To find hidden directories I used **ffuf** which discover **content** directory as **200** ok.

```bash
$ ffuf -u https://bizness.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -fs 0 -s

control
images
```

- **-u** for url
- **-w** for wordlist
- **-fs** to filter size of page
- **-s** to run ffuf in silent mode, not to print banners.

#### OFBiz

Apache OFBiz is **an open source enterprise resource planning (ERP) system**. It provides a suite of enterprise applications that integrate and automate many of the business processes of an enterprise. OFBiz is an Apache Software Foundation top level project.
**Content** directory redirect us to **OFBiz** login page, also leaking it's version **18.12** at the very left bottom

<img alt="Pasted image 20240525130514" src="https://github.com/iammR0OT/HTB/assets/74102381/684f8b60-24ec-4845-9b9c-abe7db3f54c3">

With a quick google search for **apache ofbiz 18.12** exploit's, I managed to discover Two Vuln's, **CVE-2023-49070** an pre-authentication Remote Code Execution (RCE) & **CVE-2023-51467** an Authentication Bypass Vuln with there POC's. 

<img  alt="Pasted image 20240525132727" src="https://github.com/iammR0OT/HTB/assets/74102381/f2470e5a-40a9-4e30-9002-b47629363434">

We will be using POC written by **jakobakos** can be found [here](https://github.com/jakabakos/Apache-OFBiz-Authentication-Bypass?source=post_page-----60728bcde635--------------------------------). We need two things, URL for vulnerable version of Apache Ofbiz and second command we want to execute on victim server. 

<img  alt="Pasted image 20240525153428" src="https://github.com/iammR0OT/HTB/assets/74102381/7b5480e0-79a7-4356-9b4c-5278ab532491">

We will be getting reverseshell on the box, for that start a **nc** on listeneing mode, so that it can catch reverseshell. 

<img alt="Pasted image 20240525153455" src="https://github.com/iammR0OT/HTB/assets/74102381/d192b8cf-45fc-4e82-bce0-edb841c2bff9">

After running poc we got reverseshell on our attacker machine within no time.

<img  alt="Pasted image 20240525153545" src="https://github.com/iammR0OT/HTB/assets/74102381/1c518a68-626a-4cc5-b15c-69f60e4af540">

To make our shell stable, we will be using below script and grab our flag from **ofbiz** home directory.

```python
$ python3 -c "import pty;pty.spawn('/bin/bash')"
$ export TERM=xterm
```

<img alt="Pasted image 20240525154005" src="https://github.com/iammR0OT/HTB/assets/74102381/ccc8c076-da77-44ba-a320-74579bba7aa2">

# Privilege Escalation

**Linpeas** didn't find any useful thing on machine, also there is no other user on which we can pivot to proceed. So I decided to enumerate file system manually. In `/opt/ofbiz/runtime/data/derby/ofbiz/seg0` I found a list of data files. 

#### What is derby

Apache Derby is a relational database management system developed by the Apache Software Foundation that can be embedded in Java programs and used for online transaction processing. It has a 3.5 MB disk-space footprint. Apache Derby is developed as an open source project under the Apache 2.0 license.


<img alt="Pasted image 20240525155436" src="https://github.com/iammR0OT/HTB/assets/74102381/01baf6a8-bdb5-41e4-a5cb-9980e468b598">

So all these files are database files, which means we can look for password entries in all these database files. To find Passwords we will be using **grep** command.

```bash
$ grep -ri "password"
```

- **-r** for recursion
- **-i** for case in-sensitive

We find 14 files with the keyword password in them. Let's enumerate all of them individually.

<img  alt="Pasted image 20240525160340" src="https://github.com/iammR0OT/HTB/assets/74102381/44d09c96-dacf-4af7-a232-1aa7d5f0d30a">

In **c54do.dat** I found a password hash for a admin user. 

<img  alt="Pasted image 20240525164056" src="https://github.com/iammR0OT/HTB/assets/74102381/dba4d1ab-5e52-47e5-a5d1-ca8bb4274ea9">

`$SHA$d$uP0_QaVBpDWFeo8-dRzDqRwXQ2IYNN`

#### HASH crack

Apache **ofbiz** stores the user password in hashes format and use salt to make it difficult to crack. In **ofbiz** GitHub I discover that the second value in the hash is the salt of hash. You can find it on [this Link](https://github.com/apache/ofbiz/blob/trunk/framework/base/src/main/java/org/apache/ofbiz/base/crypto/HashCrypt.java) on line number **167**.

```java
 public static String pbkdf2HashCrypt(String hashType, String salt, String value){
        char[] chars = value.toCharArray();
        if (UtilValidate.isEmpty(salt)) {
            salt = getSalt();
        }
from this snippet we can see that in ofbiz first hash type comes, then the salt vale, then the hash. So in our case the salt will be d.
```

To crack this hash we will be using a python script written by **duck-sec**. You can download it form [this link](https://github.com/duck-sec/Apache-OFBiz-SHA1-Cracker).
This script needs two flags, **--hash-string** and **--wordlist** to run successfully.

<img alt="Pasted image 20240525163819" src="https://github.com/iammR0OT/HTB/assets/74102381/244917a9-5c6e-4acd-a316-2f80dacec0bb">

The password for the root user is **monkeybizness** cracked by the script.

We can with ssh on root user using **monkeybizness** password or can use **su** (switch user) to gain access to root user.
The root flag can be found on `/root/root.txt`

<img alt="Pasted image 20240525164211" src="https://github.com/iammR0OT/HTB/assets/74102381/02c44701-53fd-4d32-9fe1-45963cfcc91c">

# Flags

User : 75ce03cdff487.....9b62a535830abd

Root : 55f6f1ac53b.....1e93b658f4fc6c6c0
# Happy Hacking ❤
