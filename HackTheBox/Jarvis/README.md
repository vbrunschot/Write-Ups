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

When we browse to the webservice on port ```64999``` we are immediately confronted with yhe following message:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/4.png">
