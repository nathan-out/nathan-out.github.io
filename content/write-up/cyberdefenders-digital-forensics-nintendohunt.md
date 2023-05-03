---
title: "Cyberdefenders · Digital Forensics · NintendoHunt"
date: 2023-05-03T13:20:01+02:00
draft: false
---

This is my proposed solution for the cyberdefenders’s challenge [NintendoHunt](https://cyberdefenders.org/blueteam-ctf-challenges/102#nav-questions) (difficult) published on April 27, 2023.

The scenario : 

> You have been hired as a digital forensics investigator to investigate a potential security breach at a small company. The company has recently noticed unusual network activity and suspects that there may be a malicious process running on one of their computers. Your task is identifying the malicious process and gathering information about its activity.

Tools:

- Volatility 2

---

## Q1 What is the process ID of the currently running malicious process?

First of all, I have to retrieve the profile to use with this dump :

```bash
$ vol.py -f memdump.mem imageinfo
Instantiating KDBG using: Unnamed AS Win10x64_17134 (6.4.17134 64bit)
```

Note that this command takes severals minutes, maybe because of the dump size (5.4Go)? One of the profile found is `Win10x64_1713`.

My first approach was to look the processes :

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 (pslist and psscan)
```

We could see a lot of processes and `svchost.exe`. But none obvious malicious file. The second part of the question was a little bit tricky for me, as I'm not familiar with Windows native process tree (svchost, explorer, dllhost and stuff). I got stuck for a while and I solved the question without follow the hint. A Cyberdefender's member gave me a tips:

> Search about the most abusable process by the hackers

I made several challenges with `svchost.exe` abuse so let's investigate on it.  According to this site : [Windows Processes - HackTricks](https://book.hacktricks.xyz/generic-methodologies-and-resources/basic-forensic-methodology/windows-forensics/windows-processes):

> There will be several processes of svchost.exe. If any of them is not using the -k flag, then that's very suspicious. If you find that services.exe is not the parent, that's also very suspicious.

I investigate on command lines with `cmdline` plugin: 

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 cmdline
...
svchost.exe pid:   7260
Command line : C:\Windows\System32\svchost.exe -k wsappx -p -s ClipSVC
************************************************************************
svchost.exe pid:   8560
Command line : "C:\Windows\svchost.exe" 
...
```

This plugin shows that process #8560 was launched in a very unusual way. This could be enough to tag it as malicious, but let's dig again:

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 psscan|grep svchost

0x0000a780001d6080 svchost.exe        5048    804 0x000000003c400002 2018-08-01 19:21:00 UTC+0000                                 
0x0000c20c6a5514c0 svchost.exe        8808    804 0x0000000079000002 2018-08-06 18:12:05 UTC+0000                                 
0x0000c20c6aa0d580 svchost.exe        8052    804 0x00000000a5c09002 2018-08-06 18:12:40 UTC+0000                                 
...
0x0000c20c6d5ac340 svchost.exe.ex     5528   4824 0x0000000119400002 2018-08-01 19:52:20 UTC+0000   2018-08-01 19:52:20 UTC+0000  
0x0000c20c6d63e080 svchost.exe        9388    804 0x000000002be00002 2018-08-06 18:11:49 UTC+0000                                 
0x0000c20c6d6fc580 svchost.exe       10012   4824 0x0000000136200002 2018-08-01 19:49:19 UTC+0000   2018-08-01 19:49:19 UTC+0000  
0x0000c20c6d82e080 svchost.exe        1404   4824 0x00000000a0f00002 2018-08-01 19:54:55 UTC+0000   2018-08-01 19:56:35 UTC+0000  
0x0000c20c6d99b580 svchost.exe.ex     8140   4824 0x00000000b8600002 2018-08-01 19:52:16 UTC+0000   2018-08-01 19:52:16 UTC+0000  
0x0000c20c6dbc5340 svchost.exe        7852   4824 0x000000003ff00002 2018-08-01 19:49:21 UTC+0000   2018-08-01 19:49:22 UTC+0000  
...
```

Some of `svchost` process are linked with PPID 804 (`services.exe`) or PPID #4824 (`explorer.exe`). A `svchost` in Windows must have a `services` process as parent. Let's filter on theses processes:

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 psscan|grep -E 'svchost.*4824'

0x0000c20c6ab2b580 svchost.exe.ex     6176   4824 0x000000004d100002 2018-08-01 19:52:19 UTC+0000   2018-08-01 19:52:19 UTC+0000  
0x0000c20c6ab70080 svchost.exe        8852   4824 0x0000000096f00002 2018-08-01 19:59:49 UTC+0000   2018-08-01 20:00:08 UTC+0000  
0x0000c20c6d5ac340 svchost.exe.ex     5528   4824 0x0000000119400002 2018-08-01 19:52:20 UTC+0000   2018-08-01 19:52:20 UTC+0000  
0x0000c20c6d6fc580 svchost.exe       10012   4824 0x0000000136200002 2018-08-01 19:49:19 UTC+0000   2018-08-01 19:49:19 UTC+0000  
0x0000c20c6d82e080 svchost.exe        1404   4824 0x00000000a0f00002 2018-08-01 19:54:55 UTC+0000   2018-08-01 19:56:35 UTC+0000  
0x0000c20c6d99b580 svchost.exe.ex     8140   4824 0x00000000b8600002 2018-08-01 19:52:16 UTC+0000   2018-08-01 19:52:16 UTC+0000  
0x0000c20c6dbc5340 svchost.exe        7852   4824 0x000000003ff00002 2018-08-01 19:49:21 UTC+0000   2018-08-01 19:49:22 UTC+0000  
0x0000c20c6ddad580 svchost.exe        8560   4824 0x00000000b2200002 2018-08-01 20:13:10 UTC+0000                                                       
```

We were able to see that PID #8560 is running, which is an important point in relation to the question, **and** that it is the same suspiciously launched process (see above). Here it is our malicious process.

## Q2 What is the md5 hash hidden in the malicious process memory?

Dump the malicious process in order to explore its memory. As the process seems hidden you have to put the `offset` (available in `pslist`, `pscan` or `psxview`):

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 memdump -p 8560 --dump-dir 8560 --offset=0x0000000084551580

Writing svchost.exe [  8560] to 8560.dmp

# let's search for md5 hash
$ strings 8560/8560.dmp|grep -Ei '(hash|md5)|[a-fA-F0-9]{31}'
# too much results!!!
```

There is too much strings and none of them seems to be the flag. Maybe the approach is not the good one so let's try to extract the raw strings and look for something interesting:

```bash
$ strings 8560/8560.dmp > 8560_raw_strings
$ cat 8560_raw_strings
...
s":
			"auto_complete":
			{
				"selected_items":
				[
				]
			},
			"buffers":
			[
				{
					"contents": "da391kdasdaadsssssss    t.h.e. fl.ag.is. M2ExOTY5N2YyOTA5NWJjMjg5YTk2ZTQ1MDQ2Nzk2ODA=",
					"settings":
...
```

This is pretty obvious right now; the flag is encoded in base64 and when the string `M2ExOTY5N2YyOTA5NWJjMjg5YTk2ZTQ1MDQ2Nzk2ODA=` is decoded we got a md5 hash: `3a19697f29095bc289a96e4504679680`.

## Q3 What is the process name of the malicious process parent?

Easy question as we found the malicious process, let's look at `psscan` and I get the answer: `explorer.exe`.

## Q4 What is the MAC address of this machine's default gateway?

You have to know where default gateway MAC address are located. By chance, I found this write-up... which is the original challenge^^ [13Cubed Mini Memory CTF Write-up](https://www.petermstewart.net/13cubed-mini-memory-ctf-write-up/). I was looking for the location of this kind of information and I found the write-up. I don't know how to find the good Windows registry key without the WU... I think this is a problem when you are a beginner, Windows registry is huge and it's hard to find accurate information.

As my research suggested, the hive to investigate is `SOFTWARE`. First, you have to obtain its offset (`virtual` column):

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 hivelist

Virtual            Physical           Name
------------------ ------------------ ----
...
0xffffd38985eb3000 0x0000000105738000 \SystemRoot\System32\Config\SOFTWARE
```

The offset to provide for the following commands is: `0xffffd38985eb3000`. The information is located under `Microsoft\Windows NT\CurrentVersion\NetworkList\Signatures\Unmanaged`. There is another randomly (I guess) generated key under it, so you have to find this value first:

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 printkey -o 0xffffd38985eb3000 -K "Microsoft\Windows NT\CurrentVersion\NetworkList\Signatures\Unmanaged"

Legend: (S) = Stable   (V) = Volatile

----------------------------
Registry: \SystemRoot\System32\Config\SOFTWARE
Key name: Unmanaged (S)
Last updated: 2018-08-01 18:50:26 UTC+0000

Subkeys:
  (S) 010103000F0000F0080000000F0000F0E3E937A4D0CD0A314266D2986CB7DED5D8B43B828FEEDCEFFD6DE7141DC1D15D

Values:
```

You have the randomly (?) generated name, so let's dig into it:

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 printkey -o 0xffffd38985eb3000 -K "Microsoft\Windows NT\CurrentVersion\NetworkList\Signatures\Unmanaged\010103000F0000F0080000000F0000F0E3E937A4D0CD0A314266D2986CB7DED5D8B43B828FEEDCEFFD6DE7141DC1D15D"

Legend: (S) = Stable   (V) = Volatile

----------------------------
Registry: \SystemRoot\System32\Config\SOFTWARE
Key name: 010103000F0000F0080000000F0000F0E3E937A4D0CD0A314266D2986CB7DED5D8B43B828FEEDCEFFD6DE7141DC1D15D (S)
Last updated: 2018-08-01 18:50:26 UTC+0000

Subkeys:

Values:
REG_SZ        ProfileGuid     : (S) {596B8D0F-BFBC-4B67-9ED8-237BD3DDABF3}
REG_SZ        Description     : (S) Network
REG_DWORD     Source          : (S) 8
REG_SZ        DnsSuffix       : (S) localdomain
REG_SZ        FirstNetwork    : (S) Network
REG_BINARY    DefaultGatewayMac : (S) 
0x00000000  00 50 56 fe d8 07
```

We found `DefaultGatewayMac` key: `00:50:56:fe:d8:07`.

## Q5 What is the name of the file that is hidden in the alternative data stream

Very interesting question as I didn't know what is ADS (alternate data stream) before. From what I understand, it's a way for hackers to hide some malicious file "behind" an innocent one (more informations here: [SANS - ADS](https://www.sans.org/blog/alternate-data-streams-overview/)). To detect this with Volatility you have to use `mftparser` plugin, **but in verbose mode** (`-v`). As the output is large (~50MB), you have to filter on it. According to this website: [Executable and Data Files](https://www.tophertimzen.com/resources/cs407/slides/week08-diskArtifacts.html), a way to detect an ADS is to filter on `$DATA ADS`:

```bash
$ vol.py -f memdump.mem --profile=Win10x64_17134 mftparser -v|grep '$DATA ADS'

$DATA ADS Name: $Bad
$DATA ADS Name: $Max
$DATA ADS Name: Zone.Identifier
$DATA ADS Name: yes.txt
```

The hidden one is `yes.txt`.

---

If you want to go further, the displayed one is `Users\CTF\Desktop\test.txt` and it's linked with the hiden file `yes.txt`, there is the MFT entry below. You can either see the contents of both files:

```bash
MFT entry found at offset 0x126515800
Attribute: In Use & File
Record Number: 117434
Link count: 1


$STANDARD_INFORMATION
Creation                       Modified                       MFT Altered                    Access Date                    Type
------------------------------ ------------------------------ ------------------------------ ------------------------------ ----
2018-08-01 19:40:27 UTC+0000 2018-08-01 19:40:56 UTC+0000   2018-08-06 18:12:15 UTC+0000   2018-08-01 19:40:56 UTC+0000   Archive

$FILE_NAME
Creation                       Modified                       MFT Altered                    Access Date                    Name/Path
------------------------------ ------------------------------ ------------------------------ ------------------------------ ---------
2018-08-01 19:40:27 UTC+0000 2018-08-01 19:40:27 UTC+0000   2018-08-01 19:40:27 UTC+0000   2018-08-01 19:40:27 UTC+0000   Users\CTF\Desktop\test.txt

$OBJECT_ID
Object ID: 618163e8-bf95-e811-8415-784f437cb8f9
Birth Volume ID: 80000000-2800-0000-0000-000000000100
Birth Object ID: 09000000-1800-0000-6869-207468657265
Birth Domain ID: 2e071800-0000-0200-8000-000048000000

$DATA
0000000000: 68 69 20 74 68 65 72 65 2e                        hi.there.

$DATA ADS Name: yes.txt
0000000000: 4f 6f 6f 68 2e 2e 2e 20 63 6f 75 6c 64 20 74 68   Oooh....could.th
0000000010: 69 73 20 62 65 20 61 20 66 6c 61 67 3f            is.be.a.flag?
```

## Q6 What is the full path of the browser cache created when the user visited "www.13cubed.com" ?

My first idea was to look at the internet history using `iehistory` plugin. Unfortunately, the tool does not return anything. But the cache is a manipulated file and we could see its path in the MFT. Using the `mftparser -v` output of the previous question, I investigate on it. I don't know where is located the browser cache so I was stuck on the question for a while. I searched the occurences of "cache", "cache\" but the number of results discouraged me.

I finally searched for the domain name, `13cubed`... and found one result!

```
$ cat mftparser_verbose|grep '13cubed'

2018-08-01 19:29:27 UTC+0000 2018-08-01 19:29:27 UTC+0000   2018-08-01 19:29:27 UTC+0000   2018-08-01 19:29:27 UTC+0000   Users\CTF\AppData\Local\Packages\MICROS~1.MIC\AC\#!001\MICROS~1\Cache\AHF2COV9\13cubed[1].htm
```

The output shows the file location which is the flag: 

```
C:\Users\CTF\AppData\Local\Packages\MICROS~1.MIC\AC\#!001\MICROS~1\Cache\AHF2COV9\13cubed[1].htm
```

**Good to know:**

1. you can retrieve a part of internet history using the MFT

2. the cache filename contains the domain's website, which is very usefull for sorting the results

Hope you enjoyed the WU :)
