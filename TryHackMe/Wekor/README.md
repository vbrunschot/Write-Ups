# Recon
```
 sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.6.227 -T5 
 ```
 <img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/1.png">

 # Enumeration
 We discover two open ports and see the content of the robots.txt. The folders don't seem to exist on the host. Except for ```comingreallysoon``` in which we find a reference to ```/it-next```.

We browse around and try to do a purchase. There's a possibility to add a coupon code. We'll catch the request in BurpSuite and pass it to sqlmap to see if it's vulnerable to SQL injection.

The target appears to be vulnerable and after some digging around we get the content of a WordPress user table. It contains multiple usernames and hashes. 
```
 sqlmap -r request.txt --columns -T wp_users -D wordpress
```
 <img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/2.png">

 We also spot ```http://site.wekor.thm/wordpress```. We'll add that to our ```/etc/hosts``` so we can then visit the WordPress page.


We will try to crack the hashes using hashcat.
First we determine the algorithm with ```hashid``` en we look up which method to use.

 <img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/3.png">

 > Visit https://hashcat.net/wiki/doku.php?id=example_hashes for available methods.
 
 ```
 hashcat -m 400 -a 0 hash /usr/share/wordlists/rockyou.txt
 ```
 Hashcat cracked multiple passwords. Let's see if we can put them to use at the WordPress site. 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/4.png">


We start a scan to discover directories to see if we can find the admin login page.
```
gobuster dir -u http://site.wekor.thm/wordpress/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/5.png">

# Intitial Foothold
We can now login and find out that ```wp_yura``` is a WordPress admin account. With these permissions we can change the content of the 404 page from a theme, resulting in a reverse shell on the target by browsing to:

```
http://site.wekor.thm/wordpress/wp-content/themes/twentytwentyone/404.php
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/6.png">

We now have a low privilege shell as ```www-data``` on the target.  As the host has python we upgrade the shell by entering:
```
python -c 'import pty;pty.spawn("/bin/bash")'
```

We spot database credentials in a config file (```/var/www/html/it-next/config.pphp```). We try using them with SSH but that didn't work.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/7.png">

Tried looking suid files, checked cronjobs and finally found something interesting with ```netstat -an```: 
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/8.png">

There appears to be running a service on port 3306 (probably MySQL) and 11211 (probably memcached).

Let's try to connect with supposedly memcached:
```
telnet localhost 11211
```
We can enumerate the service and discover a username and password:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/9.png">

# Privilege Escalation
We'll use ``su Orka`` and enter the password to login as Orka. Running ```sudo -l``` tells us we can run ```/home/Orka/Desktop/bitcoin``` as root:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/10.png">

When running the binary we are asked to enter a password. We will download the binary using netcat and decompile it with ghidra. There we find a password for the binary.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/11.png">


The binary is using an absolute path for calling the script, and an absolute path was used for the binary in the sudoers file, but, it is using a relative path for calling the python binary. And remember, we can write to ```usr/sbin``` which has a higher precedence (on this machine) over ```/usr/bin```, where the actual python binary resides.

What this means, is that the system will look for the python binary in ```/usr/sbin``` before looking somewhere else, which means we can place our own program, give it the same name, and the system will execute it instead of the actual python binary.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Wekor/assets/12.png">

We'll create bash file, which will spawn a shell, name it ```python``` and place it in ```/usr/bin```:
```sh
#!/bin/bash
/bin/bash
```
Now we'll get a shell as root as soon as we execute the following command:
```
sudo -u root /home/Orka/Desktop/bitcoin
```















