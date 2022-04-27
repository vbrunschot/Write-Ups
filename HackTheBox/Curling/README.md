# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.150 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/1.png">

Scan shows two open ports and target is running on Linux. On port 80 there's a webservice running a CMS. Next we'll try to find out which type and version.

# Enumeration

Running ```whatweb``` on the target shows us that the webservice is running Joomla.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/2.png">

```Wappalyzer``` tells us the same thing:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/4.png">

Next we'll run ```joomscan``` on the target only to find out it didn't return anything useful. We do however find the administrator login page at ```http://10.10.10.150/administrator/```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Curling/assets/3.png">







