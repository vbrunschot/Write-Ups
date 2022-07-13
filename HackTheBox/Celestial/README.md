# Recon

As usual we start with a port scan on the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.85 -T5
```

That only reveals an open port on 3000 running the Node.js Express Framework. 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/1.png">

# Enumeration

After brwosing to the website we run into a 404. But after refreshing we get diffenrent message. BurpSuite shows that on the first visit a cookie is set and after revisiting this is passed.

Removing the following from the get request resulted in an error. This tells us that the data appears to be unserialized. We also spot the username ```sun```.
```
Upgrade-Insecure-Requests: 1
If-None-Match: W/"c-8lfvj2TmiRRvB7K+JPws1w9h6aY"
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/2.png">

# Initial Foothold






