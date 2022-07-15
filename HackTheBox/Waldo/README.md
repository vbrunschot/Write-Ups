# Recon

As usual we start with a port scan against the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.87 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/1.png">


# Enumeration
Trying to run ```gobuster``` with the usual parameters results in an error. 

```
Error: the server returns a status code that matches the provided options for non existing urls. http://10.10.10.87/489ed9d8-b292-401f-aa42-a70621925b38 => 302 (Length: 0). To continue please exclude the status code, the length or use the --wildcard switch
```
This is caused by the webserver returning us a 302 status code. We can confirm this by browsing to a nonexistent page and catch the return:
```
curl -vvv http://10.10.10.87/test
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/2.png">

We can tell ```gobuster``` to ignore status codes by marking them as bad using ```-b```. With these small adjustments we can now start our scan for files and directories:
```
gobuster dir -u http://10.10.10.87 -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -b "302,404" -x php,html,txt
```

After half an hour we only got ```list.html``` which we already knew.

There is a "List Manager" running on the webserver. We can use ```BurpSuite``` to intercept the request to take a closer look.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/3.png">

We see that the ```/dirRead.php``` is requested with the ```path``` parameter set. We can change this to show all files in the directory.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/4.png">

The response shows us interesting php files. We also try if we can use ```../``` for directory traversal but it doesn't seem to work. Let's try to use ```fileRead.php``` to read the source code of ```fileRead.php```. 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/5.png">

Saving the result and reading through it using ```strings``` tells us that ```../``` is replaced and we can't view ```user.txt```. That's why earlier we were unable to use directory traversal.
```
str_replace( array("../", "..\"")
```
> (Removed the escape character ```\```)

```dirRead.php``` also has the same protection. But we can evade this by altering our request to ```....//```. Note that we add two extra periods and one slash which will be replaced by an empty char. That leaves us with the remaining ```../``` thus enabling directory traversal. We can add a few more in order to browse the highest level.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/6.png">

# Initial Foothold
After browsing around we spot the ```.ssh``` folder in the home directory of user ```nobody```. It contains ```.monitor``` and we can use ```fileRead.php``` to retrieve it's content:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/7.png">

We have to do some formatting on the key before we can use it. Change ```\/``` to ```/``` and change ```\n``` to a new line. U can also use an online json decoder. Save it as ```id_rsa``` and change it's permission to 600 by running ```chmod 600 id_rsa```.

We can now use ```ssh``` to connect to the target.
```
ssh nobody@10.10.10.87 -i id_rsa
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/8.png">

# Lateral Movement
We download and run ```linpeas.sh``` on the target which tells us our current shell is situated within a Docker container.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Waldo/assets/9.png">

Another way is to use the following commands to confirm this:
```
grep -i docker /proc/self/cgroup 2>/dev/null
find / -name "*dockerenv*" -exec ls -la {} \; 2>/dev/null
```

We can try to use the earlier found key to create a session as ```monitor```
```
ssh -i /home/nobody/.ssh/.monitor monitor@localhost
```

# Escaping Restricted Environment Method 1
We are in a restricted environment and can only run a few binaries.

pic10

By using the ```-l``` prefix we can see the links that are used to the actual binaries. Sometimes a ```r``` is added in front of the name to imply restrictions. GTFOBins tells us that ```ed``` can be used to spawn a shell which breaks us out of the restricted environment.

Just run ```red``` followed by ```!/bin/sh```.

# Escaping Restricted Environment Method 2



# Privilege Escalation



[TO BE CONTINUED]