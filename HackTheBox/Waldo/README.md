# Recon

As usual we start with a port scan against the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.87 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/1.png">


# Enumeration
Trying to run ```gobuster``` with the usual parameters results in an error. 

```
Error: the server returns a status code that matches the provided options for non existing urls. http://10.10.10.87/489ed9d8-b292-401f-aa42-a70621925b38 => 302 (Length: 0). To continue please exclude the status code, the length or use the --wildcard switch
```
This is caused by the webserver returning us a 302 status code. We can confirm this by browsing to a nonexistent page and ctach the return:
```
curl -vvv http://10.10.10.87/test
```

pic2

We can tell ```gobuster``` to ignore status codes by marking them as bad using ```-b```. With these small adjustments we can now start our scan for files and directories:
```
gobuster dir -u http://10.10.10.87 -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -b "302,404" -x php,html,txt
```

After half an hour we only got ```list.html``` which we already knew.

There is a "List Manager" running on the webserver. We can use ```BurpSuite``` to intercept the request to take a closer look.

pic3

We see that the ```/dirRead.php``` is requested with the ```path``` parameter set. We can change this to show all files in the directory.

pic4

The response shows us interesting php files. We also try if we can use ```../``` for directory traversal but it doesn't seem to work. Let's try to use ```fileRead.php``` to read the source code of ```fileRead.php```. 

pic5

Saving the result and reading through it using ```strings``` tells us that ```../``` is replaced and we can't view ```user.txt```. That's why earlier we were unable to use directory traversal.
```
str_replace( array("../", "..\"")
```
(Removed the escape character ```\```)

```dirRead.php``` also has the same protectiong. But we can evade this by altering our request to ```....//```. Note that we add two extra periods and one slash which will be replaced. That leaves us with the remaining ```../``` thus enabling directory traversal. We can add a few more in order to browse to the highest level.

pic 6

# Initial Foothold
After browsing around we spot the ```.ssh``` folder in the home directory of user ```nobody```. We can use ```fileRead.php``` to retrieve it's content.

pic 7

# Lateral Movement
# Privilege Escalation
