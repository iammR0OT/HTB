# Network
We have one domain and one subdomain

	keeper.htb
	tickets.keeper.htb

Two Ports are open

	22
	80

# Web

Tickets service is running on port 80 and **4.4.4+dfsg-2ubuntu1** is running
found default credentials 

![Pasted image 20230908111836](https://github.com/iammR0OT/HTB/assets/74102381/d7c63fe9-031c-49f0-a6fb-301dacba0c9c)


	username = root
	password = password
		
While surfing the website found another username and Email address

	Email = lnorgaard@keeper.htb
	username = lnorgaard
	
 ![Pasted image 20230908112205](https://github.com/iammR0OT/HTB/assets/74102381/4eb388d1-f431-4578-a3a1-2d4ddf718ffb)


Found his password 

	**Welcome2023!**
	
![Screenshot from 2023-09-08 11-22-55](https://github.com/iammR0OT/HTB/assets/74102381/a2521176-b942-430a-b60c-9d81be78e631)


Lets try to SSH in the account

We got a shell, lets try to esclate privileges

![Screenshot from 2023-09-08 11-26-09](https://github.com/iammR0OT/HTB/assets/74102381/e1b96dc2-4432-41b2-b2f0-c2d950a239e8)

	user flag = c2caffa87a5980ed5ad8181d85da825c

# Privilege Escalation
In home folder i found some files , **RT3000.zip, passcode.kdbx , KeePassDumpFull.dmp**

	passcode.kdbx : file contains key and passwords in it
	KeePassDumpFull.dmp : file contains dump data of key and passwords 

Now we have a .dmp file lets try to dump it and extract master key from it. after some googling found a **CVE-2023-32784** lets try to escalate it

After running that POC against KeePassFullDump file we got master key 

![Screenshot from 2023-09-08 15-28-55](https://github.com/iammR0OT/HTB/assets/74102381/9c50d426-1480-4ba0-b808-8546ea8c811e)

According to POC the first two characters are misspelled or change so we need to find out that two characters to find exact master key

After some googling i found that it is a sweet dish name which start with ro not M} 

So, Lets change it and login

		**rødgrød med fløde**
After putting that master key in kepassx we got a putty key file

![Screenshot from 2023-09-08 15-32-56](https://github.com/iammR0OT/HTB/assets/74102381/6c10954c-f6b0-4883-a640-da1537486320)

After putting master key we found putty key in the passcode.kdbx file

	putty key file are stored as .ppk 

Lets convert that ssh.ppk to ssh.pem file

	.pem file is permission called **Privacy Enhanced Mail** file used in certificate installation. we will use it here to login as root

We will use tool called puttygen to convert that ppk file into pem file
	`puttygen ssh.ppk -O private-openssh -o  ssh.pem`
Boom we are now root

![Screenshot from 2023-09-08 15-23-01](https://github.com/iammR0OT/HTB/assets/74102381/f31b2cb5-3ec4-4699-9f25-cc7399cb2a10)

![Screenshot from 2023-09-08 15-23-47](https://github.com/iammR0OT/HTB/assets/74102381/32da4b9c-24a2-4f52-a5c4-6fe435803a75)

`root = 7a5556fe52434fc5ddd2a4c149b24fba`

## Key Points

Always try to login with default credentials when you see a login page

For privilege Escalation if there is a software provided always try to find its Exploit Like here we found CVE-2023-32784

If you found any .ppk file convert it into .pem file and use it to gain shell
