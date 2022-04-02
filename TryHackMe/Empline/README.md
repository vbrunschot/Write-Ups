# Summary
We will start by enumerating the websites and gaining a reverse shell by abusing the file upload page. After that we will use a binary with capabilities set to escalate our privileges. 

# Recon
As always we start with a nmap scan on the target. As it is blocking our ICMP probes we'll use -Pn

```sh
sudo nmap -sV -sC -p- 10.10.135.237 -T4 -Pn
```

This reveals the following open ports:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/1.png">

# Port 80
We'll run gobuster to discover directories

```sh
gobuster dir -u http://10.10.135.237/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/2.png">

We find some folders but nothing of interest here. Let's try to find subdomains. I first added empline.thm to the /etc/hosts file.

```sh
wfuzz -c -u http://empline.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H 'Host: FUZZ.empline.thm' --hw 914

```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/3.png">

# job.empline.thm

We found a subdomain: job. After changing the /etc/hosts file again we are welcomed by a login form.
I tried several simple usernames and password but without any luck. SQLi didn't work either.
I opened up BurpSuite and send the login request to repeater. I found a username and password:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/4.png">

I tried to login with found credentials but it didn't work. Tried it also with SSH but with the same results.
Let's run gobuster again on the newly found subdomain:

```sh
gobuster dir -u http://job.empline.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/5.png">

We found lot's of new directories to investigate. I looked around the folders to see what files i could find. I found some interesting things in the /test/ directory:

securityTestData.sh

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/6.png">


# Initial foothold
After browsing to /careers i found a page where i could upload a file.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/8.png">

>Method:
>This hopefully allows us to upload a reverse php script, load the script and have a netcat listener ready for the incoming connectiong thus giving us a shell on the server.

I used the reverse PHP shell from PentestMonkey
https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

Saved the file as shell.php and tried to upload it. That didn't work as there were no files added to the /upload directory. Maybe .php files are blocked. I tried several different extensions:
```sh
.php3
.php4
.php5
.php7
.phps
.php-s
.pht
.phar
.php
.phtm
.phtml
```
None of them worked. I then tried to change the Content-Type with BurpSuite. This will hopefully fool the application into uploading the shell.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/9.png">

That worked! We now got a low privilege shell on the host:


<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/10.png">


After looking around for usefull files i discovered a config file for opencats which contains a username and password.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/11.png">


# Port 3306
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/7.png">

I first used the credentials i found earlier in the /test folder. But that didn't seem to work.

Then i tried it with the credentials i found in the config fle which belongs to opencats (/var/www/opencats/config.php). I can login and i start iterating the database for usefull information.
I find 3 hashes in the user table.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/12.png">

I checked with hashid to confirm that they are MD5 hashes and loaded up hashcat in an attempt to crack the hashes:
```sh
hashcat -a 0 -m 0 hashes /usr/share/wordlists/rockyou.txt -w 3 -O  
```

We got a result:
```sh
86d0dfda99dbebc424eb4407947356ac:pretonnevippasempre
```

# Privilege escalation
I logged in using SSH and tried several things for privilege escalation. I finally spot a binary with capabilities set. 
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/13.png">

After reading https://rubyreferences.github.io/rubyref/builtin/system-cli/filesystem.html i discovered i can use Ruby to change ownership and hereby give george root privileges. This allows me to get the root flag.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Empline/assets/14.png">
















