# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.150 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/1.png">

The scan shows two open ports and the target is running on Linux. On port 80 there's a webservice running a CMS. We spot a administator username in a message: ```Floris```. Next we'll try to find out the type and version of the CMS.

# Enumeration
Running ```whatweb``` on the target shows us that the webservice is running Joomla.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/2.png">

```Wappalyzer``` tells us the same thing:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/4.png">

Next we'll run ```joomscan``` on the target only to find out it didn't return anything useful. We do however find the administrator login page at ```http://10.10.10.150/administrator/```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/3.png">

# Initial Foothold
We can use ```cewl``` to create a wordlist in order to bruteforce the ```Floris``` administrator account.
```
cewl -m 3 http://10.10.10.150 -d 3 -w words.txt 
```
```Cewl``` removes trailing numbers so we'll add ```curling2018``` to our wordlist. We also use ```john the ripper``` to add numbers and symbols:
```
john --wordlist=words.txt --rules --stdout > words-modified.txt 
```

We intercept a login attempt with ```BurpSuite``` and use the information to craft the request for ```thc-hydra```:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/5.png">
```
hydra -l Floris -P words-modified.txt 10.10.10.150 http-post-form "/:username=^USER^&passwd=^PASS^&option=com_login&task=login&return=aW5kZXgucGhw&9abbf60416dedcbdf4f5530a3ddea49d=1:F=Username and password do not match" -V 
```
This didn't work either..

Reviewing the code from the website reveals a secret.txt file. This file contains a base64 encoded string which decodes to: ```Curling2018!```.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/6.png">

We can now login at the administrator page using ```Floris``` and ```Curling2018!```.

Next thing we can try is to upload a reverse php shell (from Pentestmonkey). A filter is blocking this attempt. We can try to bypass these filters by altering the request but it didn't seem to work. Another method is to change the content of a page. We'll do this for the ```index.php``` page after browsing to ```Extensions >  Templates > Templates > Beez3 Details and Files```.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/8.png">

As soon as we preview the template we have a reverse shell:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/7.png">

# Privilege Escalation
Trying to upgrade to a PTY shell didn't work as there's no ```python``` installed. We can however try to upgrade to a TTY shell:
```
/usr/bin/script -qc /bin/bash /dev/null	
export TERM=xterm
Press ctrl+z key combination 
stty raw -echo; fg
```










