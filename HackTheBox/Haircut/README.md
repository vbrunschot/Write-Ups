# Recon

As usual we start with a port scan on the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.85 -T5
```

That only reveals an open port on 3000 running the Node.js Express Framework. 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/1.png">

# Enumeration
# Initial Foothold
# Privilege Escalation
