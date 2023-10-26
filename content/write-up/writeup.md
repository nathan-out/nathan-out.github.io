# Cyberdefenders · Digital Forensic · Szechuan Sauce

The scenario:

> Your bedroom door bursts open, shattering your pleasant dreams. Your mad scientist of a boss begins dragging you out of bed by the ankle. He simultaneously explains between belches that the FBI contacted him. They found his recently-developed Szechuan sauce recipe on the dark web. As you careen past the door frame you are able to grab your incident response “Go-Bag”. Inside is your trusty incident thumb drive and laptop. **Some files may be corrupted just like in the real world. If one tool does not work for you, find another one.**

## Abstract

...

![Rick from Rick and Morty, this challenge is based on this sitcom.]()

<figcaption>Rick from Rick and Morty, this challenge is a reference to this sitcom.</figcaption>

---

## Just before starting...

This challenge contains **many** files. We have a lot of information and the challenge will not help you on which file you have to investigate on. You have to understand the differences between filetypes and tools associated.

Let's take a look to each of them:

- `20200918_0417_DESKTOP-SDN1RPT.E01`, `E02`, `E03` and `E04`

Disk dump from a machine named `DESKTOP-SDN1RPT`. The reason why there are four files is because of the tool used (Encase), it cut the dump into four file for convenience, but you can consider as one file split into four.

- `20200918_0417_DESKTOP-SDN1RPT.E01.txt`

It contains various information on `20200918_0417_DESKTOP-SDN1RPT.E01` such as the software used to acquire the dump (FTK Imager), file hash, evidence number and lot of technical stuff which is not interesting for us. By the way, this file is very usefull in real case.

- `autoruns-desktop-sdn1rpt.csv`

CSV file which contains information about software configured to run when the machine is booted or a user log-in.

- `autorunsc-citadel-dc01.csv`

Same as the previous one, but for the citadel machine.

- `case001.pcap`

Network capture, contains information on network traffic.

- `citadeldc01.mem`

RAM dump of the citadel machine, containing all runtime information.

- `DESKTOP-SDN1RPT.mem`

Same as the previous one, but for the desktop machine.

- `E01-DC01/`
    - `20200918_0347_CDrive.E01`, `E02`

    - `20200918_0347_CDrive.E01.txt`

Bellow a table summarizing tool and filetype associated:

| Extension | Tool                                    |
|-----------|-----------------------------------------|
| E0x       | Autopsy, FTK Imager                     |
| mem       | Volatility                              |
| pcap      | Wireshark, Network Miner, Brim Security |
| csv, txt  | Text editor, or an Excel like           |

I will use Autopsy and not FTK Imager because Autopsy has a powerfull OS parser and FTK Imager is just to open and browse image file. Plus, I will not use Brim Security, as I'm not familiar with, instead, I will use Wireshark and Network Miner.

## Q1 What’s the Operating System version of the Server? (two words)

Using [Autopsy](https://www.autopsy.com/), create a new case and add data source (image file, add both `E01`) and waits until parsing is complete. Then, go to `Operating System Information`:

![The Program Name column contains the OS version.](img/os_info.png)

<figcaption>The Program Name column contains the OS version.</figcaption>

To fully understand where this information is stored, you can also parse hives located at `C:\Windows\System32\config\SOFTWARE`, in the key `Microsoft\Windows NT\CurrentVersion`, Autopsy provides a native parser but you can also use [Registry Explorer from Eric Zimmerman](https://www.sans.org/tools/registry-explorer/).

## Q2 What’s the Operating System of the Desktop? (four words separated by spaces)

Same as the previous one.

## Q3 What was the IP address assigned to the domain controller?

As explained in the Q1, you can dump the `SYSTEM` hive located at: `C:\Windows\System32\config`. Or, use Network Miner, you have to convert `case001.pcap` into a format accepted by the software. To do that, just open the file with Wireshark > Save As > Choose `Wireshark/tcpdump... - pcap` and open it with Network Miner. Once opened, you will see an overview of the network:

![I love this tool, as it provides a very usefull overview, especially in this challenge where there are many machines.](img/network_miner1.png)

<figcaption>I love this tool, as it provides a very usefull overview, especially in this challenge where there are many machines.</figcaption>

Search for a Citadel Windows machine and there is the IP.

## Q4 What was the timezone of the Server?

Tricky question, first, you have to find the timezone set into the server. It's located into the `SYSTEM` registry. Using Autopsy, you can easily read it:

![](img/timezone.png)

The server is set on Pacific Standard Time which is UTC-7. **But** the admin set a wrong timezone on the domain controller. You can see it on NTP packets (clock unsynchronized). To retrieve the correct timezone, you can correlate two events in the network and in the DC. To see how you can do, you can check this write-up: [https://ellisstannard.medium.com/cyberdefenders-szechuan-sauce-writeup-ab172eb7666c](https://ellisstannard.medium.com/cyberdefenders-szechuan-sauce-writeup-ab172eb7666c).

You will see that the system is set one hour after the correct timezone, so the answer is UTC-6.

## Q5 What was the initial entry vector (how did they get in)?. Provide protocol name.

Extract security logs from the DC located at: `%System32%\winevt\Logs\Security.evtx`. Then, you can use a log viewer such as Microsoft Event Viewer > click on Open saved logs > select your export.

Time to analyze, you can assume that the attacker wants to log into the machine. Each event has a given ID and the ID for failed attempt is `4625` and `4624` for a successfull one. To quickly have the list of the person who wants to access the machine, I used both `EvtxECmd` to dump all the logs (the Windows view did not satisfy me) and `EZViewer` to vizualize them. We can see that a Kali machine bruteforced the access with a successfull authentication on the DC using LogonType 10 which is RDP, we also have an IP.

![](img/bruteforce_kali.png)

Plus, we can investigate on the network dump. A good first step is to get an overview; Statistics > Protocol Hierarchy.

![](img/protocol_hierarchy.png)

I highlighted RDP over UDP, this is interesting because RDP provides a graphical interface to connect to another computer over a network connection. Sometimes, RDP is used by attacker to gain access. To investigate more in depth, search on `rdp && ip.dst == 10.42.85.10` (DC IP address). The first packet contains a Cookie filled with the value `nmap`, which is a scanning tool used by attackers. Plus, the source address is `194.61.24.102`, using VirusTotal you will see that this IP ws used by pirates.

The protocol name is **RDP**.

## Q6 What was the malicious process used by the malware? (one word)

Attacker gained access to the DC and we want to know what's happened on this machine, time to use Volatility! Usually, I start my investigation with `pstree` to get an overview:

```
PID     PPID    ImageFileName   Offset(V)       Threads Handles SessionId       Wow64   CreateTime      ExitTime

4       0       System  0xe00062c0a900  98      -       N/A     False   2020-09-19 01:22:38.000000      N/A
* 204   4       smss.exe        0xe00062c0a900  2       -       N/A     False   2020-09-19 01:22:38.000000      N/A
324     316     csrss.exe       0xe00062c0a900  8       -       0       False   2020-09-19 01:22:39.000000      N/A
404     316     wininit.exe     0xe00062c0a900  1       -       0       False   2020-09-19 01:22:40.000000      N/A
* 460   404     lsass.exe       0xe00062c0a900  31      -       0       False   2020-09-19 01:22:40.000000      N/A
* 452   404     services.exe    0xe00062c0a900  5       -       0       False   2020-09-19 01:22:40.000000      N/A
** 640  452     svchost.exe     0xe00062c0a900  8       -       0       False   2020-09-19 01:22:40.000000      N/A
*** 2056        640     WmiPrvSE.exe    0xe00062c0a900  11      -       0       False   2020-09-19 01:23:21.000000      N/A
*** 2764        640     WmiPrvSE.exe    0xe00062c0a900  6       -       0       False   2020-09-19 04:37:42.000000      N/A
** 1292 452     Microsoft.Acti  0xe00062c0a900  9       -       0       False   2020-09-19 01:22:57.000000      N/A
** 3724 452     spoolsv.exe     0xe00062c0a900  13      -       0       False   2020-09-19 03:29:40.000000      N/A
** 1556 452     VGAuthService.  0xe00062c0a900  2       -       0       False   2020-09-19 01:22:57.000000      N/A
** 796  452     vds.exe 0xe00062c0a900  11      -       0       False   2020-09-19 01:23:20.000000      N/A
** 668  452     svchost.exe     0xe00062c0a900  16      -       0       False   2020-09-19 01:22:41.000000      N/A
** 2460 452     msdtc.exe       0xe00062c0a900  9       -       0       False   2020-09-19 01:23:21.000000      N/A
** 800  452     svchost.exe     0xe00062c0a900  12      -       0       False   2020-09-19 01:22:40.000000      N/A
** 928  452     svchost.exe     0xe00062c0a900  16      -       0       False   2020-09-19 01:22:41.000000      N/A
** 1956 452     svchost.exe     0xe00062c0a900  30      -       0       False   2020-09-19 01:23:20.000000      N/A
** 2216 452     dllhost.exe     0xe00062c0a900  10      -       0       False   2020-09-19 01:23:21.000000      N/A
** 684  452     svchost.exe     0xe00062c0a900  6       -       0       False   2020-09-19 01:22:40.000000      N/A
** 1332 452     dfsrs.exe       0xe00062c0a900  16      -       0       False   2020-09-19 01:22:57.000000      N/A
** 1600 452     vmtoolsd.exe    0xe00062c0a900  9       -       0       False   2020-09-19 01:22:57.000000      N/A
** 848  452     svchost.exe     0xe00062c0a900  39      -       0       False   2020-09-19 01:22:41.000000      N/A
*** 3056        848     WMIADAP.exe     0xe00062c0a900  5       -       0       False   2020-09-19 04:37:42.000000      N/A
*** 3796        848     taskhostex.exe  0xe00062c0a900  7       -       1       False   2020-09-19 04:36:03.000000      N/A
** 1236 452     svchost.exe     0xe00062c0a900  8       -       0       False   2020-09-19 01:23:21.000000      N/A
** 1368 452     dns.exe 0xe00062c0a900  16      -       0       False   2020-09-19 01:22:57.000000      N/A
** 1000 452     svchost.exe     0xe00062c0a900  18      -       0       False   2020-09-19 01:22:41.000000      N/A
** 1644 452     wlms.exe        0xe00062c0a900  2       -       0       False   2020-09-19 01:22:57.000000      N/A
** 1392 452     ismserv.exe     0xe00062c0a900  6       -       0       False   2020-09-19 01:22:57.000000      N/A
** 1660 452     dfssvc.exe      0xe00062c0a900  11      -       0       False   2020-09-19 01:22:57.000000      N/A
412     396     csrss.exe       0xe00062c0a900  10      -       1       False   2020-09-19 01:22:40.000000      N/A
492     396     winlogon.exe    0xe00062c0a900  4       -       1       False   2020-09-19 01:22:40.000000      N/A
* 808   492     dwm.exe 0xe00062c0a900  7       -       1       False   2020-09-19 01:22:40.000000      N/A
3644    2244    coreupdater.ex  0xe00062c0a900  0       -       2       False   2020-09-19 03:56:37.000000      2020-09-19 03:56:52.000000
3472    3960    explorer.exe    0xe00062c0a900  39      -       1       False   2020-09-19 04:36:03.000000      N/A
* 2608  3472    vmtoolsd.exe    0xe00062c0a900  8       -       1       False   2020-09-19 04:36:14.000000      N/A
* 2840  3472    FTK Imager.exe  0xe00062c0a900  9       -       1       False   2020-09-19 04:37:04.000000      N/A
* 3260  3472    vm3dservice.ex  0xe00062c0a900  1       -       1       False   2020-09-19 04:36:14.000000      N/A
400     1904    ServerManager.  0xe00062c0a900  10      -       1       False   2020-09-19 04:36:03.000000      N/A
```

There is no obvious malicious file, but correlating `pstree` with `netscan` we will have a **lot** of results because it's a DC. Let's try to find a C2 evidence by highlighting the network traffic:

```
Offset	Proto	LocalAddr	LocalPort	ForeignAddr	ForeignPort	State	PID	Owner	Created

0x3148500	TCPv4	0.0.0.0	636	0.0.0.0	0	LISTENING	460	lsass.exe	-
0x314ac30	TCPv4	0.0.0.0	389	0.0.0.0	0	LISTENING	460	lsass.exe	-
0x314ac30	TCPv6	::	389	::	0	LISTENING	460	lsass.exe	-
0x31fa880	UDPv4	0.0.0.0	0	*	0		460	lsass.exe	2020-09-19 01:22:50.000000 
0x3f2b560	UDPv4	0.0.0.0	0	*	0		1368	dns.exe	2020-09-19 01:22:57.000000 
0x5ea7910	TCPv4	0.0.0.0	49152	0.0.0.0	0	LISTENING	404	wininit.exe	-
0x5ea7910	TCPv6	::	49152	::	0	LISTENING	404	wininit.exe	-
0x5eabed0	TCPv4	0.0.0.0	49152	0.0.0.0	0	LISTENING	404	wininit.exe	-
0x5eaf250	TCPv4	0.0.0.0	135	0.0.0.0	0	LISTENING	684	svchost.exe	-
0x5eb0540	TCPv4	0.0.0.0	135	0.0.0.0	0	LISTENING	684	svchost.exe	-
0x5eb0540	TCPv6	::	135	::	0	LISTENING	684	svchost.exe	-
0x5ee0b90	UDPv4	0.0.0.0	0	*	0		1368	dns.exe	2020-09-19 01:22:57.000000 
0x5ee9d20	UDPv4	0.0.0.0	0	*	0		1368	dns.exe	2020-09-19 01:22:57.000000 
0x5f25300	TCPv4	0.0.0.0	49153	0.0.0.0	0	LISTENING	800	svchost.exe	-
0x5f25300	TCPv6	::	49153	::	0	LISTENING	800	svchost.exe	-
0x5f284a0	TCPv4	0.0.0.0	49153	0.0.0.0	0	LISTENING	800	svchost.exe	-
0x5f9bec0	UDPv4	0.0.0.0	0	*	0		928	svchost.exe	2020-09-19 01:23:09.000000 
0x5f9bec0	UDPv6	::	0	*	0		928	svchost.exe	2020-09-19 01:23:09.000000 
0x5fa53b0	UDPv4	10.42.85.10	41136	*	0		4	System	2020-09-19 01:22:42.000000 
0x5fb76c0	UDPv4	10.42.85.10	41136	*	0		4	System	2020-09-19 01:22:42.000000 
0x5fd1880	TCPv4	0.0.0.0	49154	0.0.0.0	0	LISTENING	848	svchost.exe	-
0x5fd1880	TCPv6	::	49154	::	0	LISTENING	848	svchost.exe	-
0x5fdcdd0	TCPv4	0.0.0.0	49154	0.0.0.0	0	LISTENING	848	svchost.exe	-
...
0x20df6ba0	UDPv4	0.0.0.0	0	*	0		1368	dns.exe	2020-09-19 01:22:57.000000 
0x20e28d10	TCPv6	fe80::2dcf:e660:be73:d220	49155	fe80::2dcf:e660:be73:d220	62777	CLOSED	460	lsass.exe	-
0x20f52a00	TCPv6	fe80::2dcf:e660:be73:d220	135	fe80::2dcf:e660:be73:d220	62779	CLOSED	684	svchost.exe	N/A
0x20fc7590	TCPv4	10.42.85.10	62613	203.78.103.109	443	ESTABLISHED	3644	coreupdater.ex	N/A
0x20fffe50	TCPv4	0.0.0.0	62475	0.0.0.0	0	LISTENING	3724	spoolsv.exe	-
0x20fffe50	TCPv6	::	62475	::	0	LISTENING	3724	spoolsv.exe	-
0x211a9560	UDPv4	0.0.0.0	0	*	0		1368	dns.exe	2020-09-19 01:22:57.000000 
```

`coreupdater.exe` is suspicious because it uses port number 443 and its IP is tagged as malicious by Virus Total. It's surrely our malware.

## Q7 Which process did malware migrate to after the initial compromise? (one word)

A **very usefull** Volatility plugin is `malfind`, it helps to identify injected code inside process. A Windows executable always start with `MZ`. Finding a `MZ` header inside a process is very suspicious because is a strong sign of code injection.

```
3724    spoolsv.exe     0x4afc1f0000    0x4afc25afff    VadS    PAGE_EXECUTE_READWRITE  107     1       Disabled
4d 5a 90 00 03 00 00 00 MZ......
04 00 00 00 ff ff 00 00 ........
b8 00 00 00 00 00 00 00 ........
40 00 00 00 00 00 00 00 @.......
00 00 00 00 00 00 00 00 ........
00 00 00 00 00 00 00 00 ........
00 00 00 00 00 00 00 00 ........
00 00 00 00 00 01 00 00 ........
0x4afc1f0000:   pop     r10
0x4afc1f0002:   nop
0x4afc1f0003:   add     byte ptr [rbx], al
0x4afc1f0005:   add     byte ptr [rax], al
0x4afc1f0007:   add     byte ptr [rax + rax], al
0x4afc1f000a:   add     byte ptr [rax], al
3724    spoolsv.exe     0x4afc070000    0x4afc0a8fff    VadS    PAGE_EXECUTE_READWRITE  57      1       Disabled
4d 5a 41 52 55 48 89 e5 MZARUH..
48 83 ec 20 48 83 e4 f0 H...H...
e8 00 00 00 00 5b 48 81 .....[H.
c3 b7 57 00 00 ff d3 48 ..W....H
81 c3 34 b6 02 00 48 89 ..4...H.
3b 49 89 d8 6a 04 5a ff ;I..j.Z.
d0 00 00 00 00 00 00 00 ........
00 00 00 00 f0 00 00 00 ........
0x4afc070000:   pop     r10
0x4afc070002:   push    r10
0x4afc070004:   push    rbp
0x4afc070005:   mov     rbp, rsp
0x4afc070008:   sub     rsp, 0x20
0x4afc07000c:   and     rsp, 0xfffffffffffffff0
0x4afc070010:   call    0x4afc070015
0x4afc070015:   pop     rbx
0x4afc070016:   add     rbx, 0x57b7
0x4afc07001d:   call    rbx
0x4afc07001f:   add     rbx, 0x2b634
0x4afc070026:   mov     qword ptr [rbx], rdi
0x4afc070029:   mov     r8, rbx
0x4afc07002c:   push    4
0x4afc07002e:   pop     rdx
0x4afc07002f:   call    rax
0x4afc070031:   add     byte ptr [rax], al
0x4afc070033:   add     byte ptr [rax], al
0x4afc070035:   add     byte ptr [rax], al
0x4afc070037:   add     byte ptr [rax], al
0x4afc070039:   add     byte ptr [rax], al
0x4afc07003b:   add     al, dh
0x4afc07003d:   add     byte ptr [rax], al
3724    spoolsv.exe     0x4afc260000    0x4afc283fff    VadS    PAGE_EXECUTE_READWRITE  36      1       Disabled
4d 5a 90 00 03 00 00 00 MZ......
04 00 00 00 ff ff 00 00 ........
b8 00 00 00 00 00 00 00 ........
40 00 00 00 00 00 00 00 @.......
00 00 00 00 00 00 00 00 ........
00 00 00 00 00 00 00 00 ........
00 00 00 00 00 00 00 00 ........
00 00 00 00 e0 00 00 00 ........
0x4afc260000:   pop     r10
0x4afc260002:   nop
0x4afc260003:   add     byte ptr [rbx], al
0x4afc260005:   add     byte ptr [rax], al
0x4afc260007:   add     byte ptr [rax + rax], al
0x4afc26000a:   add     byte ptr [rax], al
```

## Q8 Identify the IP Address that delivered the payload.

Refer to Q5 answer. Using this Wireshark filter : `rdp && ip.dst == 10.42.85.10` we can see that the source IP is always `194.61.24.102`. Plus, VirusTotal flag this IP as potentially malicious. Using Network Miner, there is another clue, this machine uses `nmap`:

![](img/nmap_clue.png)

## Q9 What IP Address was the malware calling to?

Since Q6, we already know that `203.78.103.109` is the C2 IP.

## Q10 Where did the malware reside on the disk?

We are looking for disk evidence, let's use Autopsy again. To search for a filename, go to Tools > File Search by Attributes:

![](img/autopsy_search.png)

Here are the results:

![](img/coreupdater_location.png)

## Q11 What's the name of the attack tool you think this malware belongs to? (one word)

Extract `coreupdater.exe`, calculate its footprint and go to VirusTotal, you will find that this program is built on Metasploit. I suppose that is you try to reverse it, you will find the tool name too.

## Q12 One of the involved malicious IP's is based in Thailand. What was the IP?

Use an online tool like [https://www.hostip.fr/](https://www.hostip.fr/) to locate one of the previous two IP address.

## Q13 Another malicious IP once resolved to klient-293.xyz . What is this IP?

I didn't understood the question and its sens. I tried to search on this hostname on the network dump. One of the intended answer was to use Virus Total and retrieve one of the previous involved IP.

## Q14 The attacker performed some lateral movements and accessed another system in the environment via RDP. What is the hostname of that system?

We have two disk and RAM dumps, we can assume that the second system is `Desktop-SDN1RPT`, but let's prove it. Thanks to Wireshark, we can get all the outgoing RDP connexion from the DC: `rdp && ip.src == 10.42.85.10`. You can see packets from the DC to the attacker's host but also to `10.42.85.115`, thanks to Network Miner, you can find that this IP is related to `Desktop-SDN1RPT`. If you don't have Network Miner, you can resolve the hostname by searching `LLMNR` packets in Wireshark.

## Q15 Other than the administrator, which user has logged into the Desktop machine? (two words)

You can browse logs but when a user login for the first time, a new user is created on the host machine. Go to `C:\Users` to see all of them. I don't know which one is the intended one but it's Rick Sanchez.

## Q16 What was the password for "jerrysmith" account?

Quite tricky question, and not a forensic one. You can resolve it by extract hashes from the DC disk dump and crack them. To extract it, use `secretsdump.py` or `ntdsutils.exe` (available in a DC), the first method is the easiest one. First, you need to extract files which contains hashes:

- `C:\Windows\System32\config\SYSTEM`

- `C:\Windows\NTDS\ntds.dit`

Then, use the Python tool to extract hashes and finally crack them with John, Hashcat on an online tool (weak password). To extract hashes with `secretsdump.py`:

`python secretsdump.py -system <SYSTEM hive path> -ntds <ntds.dit path> -outputfile <path>`

## Q17 What was the original filename for Beth’s secrets?

We can suppose that this file has a name like `.*secret.*` or `.*beth.*`, we can search a file by its filename with Autopsy (Tools > File Search by Attributes), we will find several files related to `Beth_Secret.txt`. Plus, Autopsy can read it and give its content.

## Q18 What was the content of Beth’s secret file? ( six words, spaces in between)

Unfortunately, this isn't the correct answer. We can see that the file was deleted (red cross), maybe the file is still in the bin? Go to `$Recycle.Bin`, there is a folder named like `S-1-...-500`, 500 is the Administrator's ID, so this folder contains Administrator's deleted files. Inside it, filenames are broken (if you know how filesystem works you already know that is intended), just open each of them, you will find the answer. The very first one contains printable caracters: `FileShare\Secrets\SECRET_beth.txt` and the second one contains its content.

## Q19 The malware tried to obtain persistence in a similar way to how Carbanak malware obtains persistence. What is the corresponding MITRE technique ID?

Tricky question, as you juste have to do some threat hunting; search informations on Cabarnak malware and its persitence method. You will find this website: [https://attack.mitre.org/groups/G0008/](https://attack.mitre.org/groups/G0008/), search for persistence and its Mitre ID.