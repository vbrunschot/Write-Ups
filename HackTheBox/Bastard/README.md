# Recon

As usual we start with a port scan on the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.9 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Bastard/assets/1.png">

We find a few open ports with a webservice and a RPC service running on a Windows host. We also get some version information of the IIS service and Drupal. In the ```robots.txt``` file we find interesting locations but we are not allowed to view all of them.

# Enumeration
Although we have some info from the ```robots.txt``` file we will conduct a directory scan on the target using ```gobuster```:
```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.9
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Bastard/assets/4.png">

Nothing interesting here. Let's move on to the Drupal CMS. We are unable to create a new account because of problems with sending e-mail:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Bastard/assets/2.png">

But we can check for usernames to see if it's already in use:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Bastard/assets/3.png">

# Initial Foothold
We can also spot the version by browsing to ```http://10.10.10.9/changelog.txt```. It appears to run on Drupal version 7.54 which has a remote code execution vulnerability. We can use ```drupalgeddon2``` to gain a reverse shell. https://github.com/dreadlocked/Drupalgeddon2

You'll have to install the ```highline gem``` if you haven't already:
```
sudo gem install highline
```
And then run with ```ruby drupalgeddon2.rb http://10.10.10.9/``` to get a shell:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Bastard/assets/5.png">

With the ```systeminfo``` command we get more information about the target. It's running Microsoft Windows Server 2008 R2 Datacenter 6.1.7600. At this point we can't see if the system is patched.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Bastard/assets/6.png">

[TO BE CONTINUED]






