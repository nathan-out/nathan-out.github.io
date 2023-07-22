---
title: "Cyberdefenders · Digital Forensics · Redline"
date: 2023-07-22T16:23:52+03:00
draft: false
---

The Scenario:

> "As a member of the Security Blue team, your assignment is to analyze a memory dump using Redline and Volatility tools. Your goal is to trace the steps taken by the attacker on the compromised machine and determine how they managed to bypass the Network Intrusion Detection System "NIDS". Your investigation will involve identifying the specific malware family employed in the attack, along with its characteristics. Additionally, your task is to identify and mitigate any traces or footprints left by the attacker."

Tools:

- Volatility2 & 3

## Abstract

This Blue Team challenge writeup involves analyzing a suspicious process found in memory using the volatility framework. The process name is "oneetx.exe" and exhibits a suspicious behavior with the memory protection set to "PAGE_EXECUTE_READWRITE," which can evade antivirus detection. 

Further investigation reveals a child process named "rundll32.exe" associated with the suspicious process. The investigation also identifies the presence of a VPN program named "Outline.exe," which might be responsible for the attacker's VPN connection. However, the attacker's IP address cannot be retrieved using Volatility2, but Volatility3 successfully extracts the IP address as "77.91.124.20." By cross-referencing this IP address on VirusTotal, it is determined that the malware belongs to the "RedLine Stealer" family. 

Additionally, the URL of the PHP file visited by the attacker is found on VirusTotal, as well as by analyzing the process memory, resulting in the complete URL. Finally, the full path of the malicious executable, "oneetx.exe," is obtained through process memory analysis.

## 1. What is the name of the suspicious process? 

```
vol.py -f <file> imageinfo
```

```
vol.py -f <file> --profile=<profile> malfind

Process: oneetx.exe Pid: 5896 Address: 0x400000
Vad Tag: VadS Protection: PAGE_EXECUTE_READWRITE
Flags: PrivateMemory: 1, Protection: 6

0x0000000000400000  4d 5a 90 00 03 00 00 00 04 00 00 00 ff ff 00 00   MZ..............
0x0000000000400010  b8 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00   ........@.......
0x0000000000400020  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0x0000000000400030  00 00 00 00 00 00 00 00 00 00 00 00 00 01 00 00   ................
```

PAGE_EXECUTE_READWRITE allows the program to modify its code or be modified by another process during execution. This could help to evade antivirus detection. When `malfind` found something, it's likely a bad new as it scan the image to find injected code blocks.

## 2. What is the child process name of the suspicious process?

```
pstree

 0xffffad8189b41080:oneetx.exe                       5896   8844      5      0 2023-05-21 22:30:56 UTC+0000
. 0xffffad818d1912c0:rundll32.exe                    7732   5896      1      0 2023-05-21 22:31:53 UTC+0000
```

## 3. What is the memory protection applied to the suspicious process memory region? 

Fount with `malfind`.

## 4. What is the name of the process responsible for the VPN connection?

A program looks different as it's not a Windows default: Outline.exe. Searching this name on internet allows you to know that this a VPN program.

## 5. What is the attacker's IP address? 

Surprisingly, Volatility2 was not able to retrieve this information. When I tried to perform `netscan`, the connection was marked as close and I can't get the remode IP address.

With vol3, you can retrieve this information easily:

```
vol.py -f "MemoryDump.mem" windows.netscan|grep -i 5896

Offset	Proto	LocalAddr	LocalPort	ForeignAddr	ForeignPort	State	PID	Owner	Created
0xad818de4aa20.0TCPv4	10.0.85.2DB scan55462fin 77.91.124.20    80             CLOSED	5896	oneetx.exe	2023-05-21 23:01:22.000000
```

## 6. Based on the previous artifacts. What is the name of the malware family? 

We have few IOCs, the most usefull one is probably the IP address. Using VirusTotal, in the details section, on the Google results part, you will see that is a **RedLine Stealer**.

## 7. What is the full URL of the PHP file that the attacker visited? 

You can retrieve this information on VirusTotal too. But, you can also dump the process memory and search on it:

```
vol.py -f "MemoryDump.mem" -o <output_dir> windows.memmap --dump --pid 5896

# extracts strings
strings -el <output_dir>/pid.5896.dmp strings_5896

# search on the IOC
grep -i 'http://77.91.124.20' strings_5896
```

You will find the complete path.

## 8. What is the full path of the malicious executable? 

Using the same file, you can look up for the file name and retrieve is location:

```
grep -ie 'oneetx\.exe' strings_5896
```
