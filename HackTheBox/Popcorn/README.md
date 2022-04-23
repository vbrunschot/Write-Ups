# Recon

We start with the usual nmap scan on the target. It shows 2 open ports and the target seems to be running Linux. 
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.6 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/1.png">

# Enumeration

Browsing to http://10.10.10.6 shows a default page, nothing interesting here. Let's start a gobuster scan on the target to enumerate directories.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/2.png">

We find several interesting directories. One seems to be running an API to rename of move files. This might be usefull later on.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/3.png">

On ```/torrent``` there's a torrent website where we can create a new user and add a new torrent. Most obvious thing to try to accomplish is to upload a reverse shell script and execute it while we have a listener ready. I tried uploading a php script but the filter blocked the upload.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/4.png">

# Initial Foothold
Next we'll try to upload a valid torrent file to see if that get's us any further.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/5.png">

We have successfully uploaded the torrent and see that we can also add a screenshot. Let's try if we can bypass the upload filtering by renaming the file and changing the content type to ```image/png```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/7.png">

We now have a reverse shell. We stabilize it using ```python -c 'import pty;pty.spawn("/bin/bash")'``` and find the user flag in the ```george``` home folder.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/11.png">

# Privilege Escalation
After some poking around we find out that the kernel version is outdated and has known vulnerabilities.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/10.png">

https://github.com/offensive-security/exploitdb/blob/master/exploits/linux/local/40839.c
We setup a http server using ```sudo python2 -m SimpleHTTPServer 80``` and use ```wget``` to download the script. After downloading we compile the code, set execution rights and run the script.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/8.png">

We can now log in as the new user ```firefart``` with the chosen password and have root privileges on the target.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Popcorn/assets/9.png">








