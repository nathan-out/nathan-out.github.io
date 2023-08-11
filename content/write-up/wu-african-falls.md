# African falls

The scenario:

> John Doe was accused of doing illegal activities. A disk image of his laptop was taken. Your task as a soc analyst is to analyze the image and understand what happened under the hood.

![African falls illustration made by Midjourney.](/img/write-up/african-falls-illustration.png)

<figcaption>African falls illustration made by Midjourney.</figcaption>

## Abstract

The article describes a forensic challenge involving the analysis of a disk image from a laptop belonging to a suspect named John Doe, who is accused of illegal activities. The task for a SOC analyst is to uncover various pieces of information from the disk image, using a combination of tools including FTK Imager, Autopsy, command line utilities, and specialized tools like Metadata2go. The challenge consists of multiple questions aimed at extracting specific details, such as the MD5 hash value of the suspect's disk, phrases searched, connected FTP server's IPv4 address, deleted files, Tor Browser usage, email address, network port scan details, photo metadata, and more. 

The article provides detailed instructions on how to approach each question, highlighting the use of relevant tools and demonstrating the steps to extract the required information from the disk image. It involves analyzing a variety of artifacts, including web history, installed programs, file metadata, PowerShell history, and system registry data. The challenge concludes with insights into retrieving password hashes and cracking them to determine the user's Windows login password.

---

First, mount the image using [FTK Imager](https://www.exterro.com/ftk-imager) (Windows) or command lines tools (Linux), follow this very good tutorial: [Guide: mounting challenge disk image on Linux.](https://bwiggs.com/posts/2021-07-25-cyberdefenders-hacked/).

**Important**: AD1 files are painfull to use with Autopsy, If you want to analyse AD1 file with this software, you will have troubles. As Autopsy is very powerfull and embed various parsing tools, it's convenient to use it. My method is to:

1. Mount AD1 file with FTK Imager

2. Export files using FTK Imager (right-click > Export files)

3. In Autopsy, Add Data Source > Logical Files > Add (select the previously exported files)

4. Data ingestion should now works :)

## Q1. What is the MD5 hash value of the suspect disk?

Image file creation log provides this information, check the `DiskDrigger.txt` you will find various information such as the MD5 hash.

## Q2. What phrase did the suspect search for on 2021-04-29 18:17:38 UTC? (three words, two spaces in between)

Under `Data Artifacts > Web History`, you get all the information, sort by time and you  will find two URLs at the given time: `password cracking lists - Google Search`.

Or, you can manually browe the history file here: `Users\John Doe\AppData\Local\Google\Chrome\User Data\Default\History`.

## Q3. What is the IPv4 address of the FTP server the suspect connected to?

Little bit tricky as you must know the most famous FTP client: [FileZilla](https://wiki.filezilla-project.org/). To know if the software is installed go to `Data Artifacts > Installed Programs`, there is FileZilla, let's read its log file.

If you don't have Autopsy you can browse `SOFTWARE` hive, you will find FileZilla too.

So FileZilla is installed, what's next? Application datas are located at `Users/<Username>/AppData/Roaming|Local/AppData/` where `Roaming|Local` means that you have to browse theses two directories. The main difference is that roaming datas will "follow" a user throught another domains, and local will not. During my forensic learning journey, I can say that sometimes it's located at `Roaming`, sometimes at `Local`, you have to dig into both.

Digging into `Local/FileZilla` is useless, there is only icons and stuff like that, not user datas. `Roaming/FileZilla/` contains xml files and especially `recentservers.xml`. There is our answer!

## Q4. What date and time was a password list deleted in UTC? (YYYY-MM-DD HH:MM:SS UTC)

Using FTK Imager, I browsed `$Recycle.Bin`. When a user delete a file, this is the location where the file is put in Windows. To understand how does it works I found this article: [Recycle Bin Forensic.](https://medium.com/@akhi.pbm/recycle-bin-forensic-5e58e33e7f30).

> From windows Vista and later the recycler path was renamed as `C:\$Recycle.Bin\SID*\$I` and `C:\$Recyecle.Bin\SID*\$R` where `$I` contains meta data of the deleted file and `$R` contains actual deleted file.

We have this exact structure, let's get the `$R` date modified timestamp: 29/04/2021 18:19:55. Opening it we can see that it's obviously a password list. The answer has to be in UTC format, we can assume that UTC is the system timezone, but instead let's investigate to confirm. System timezone is located in `SYSTEM` hive: `SYSTEM\<CurrentControlSet>\Control\TimeZoneInformation\TimeZoneKeyName`, which is `UTC`.

**But**, this is not the answer, you have to provide the `$I` timestamp. I didn't searched why, to me it's more accurate to provide the `$R` one but maybe I misuderstood something. Two timestamps are close.

*Note: I couldn't export `$Recycle.Bin` with FTK Imager, I didn't searched why an error occured. I wanted to add these files into Autopsy to observe if the tool can retrieve deleted files.*

## Q5. How many times was Tor Browser ran on the suspect's computer? (number only)

I found this article: [Windows Forensics : Evidence of Execution](https://frsecure.com/blog/windows-forensics-execution/). To summarize, in our case, one interesting artifact is prefecth.

> Prefetch essentially grabs all of the files associated with an application from disk and writes them to memory so the user doesn’t have to wait for them to be loaded from disk.

In our case, it could be use to know how many times a given software was launched. Prefetch files are located at: `Windows\Prefetch` all files begin by the software name following by an ID. In our case we can find theses two files:

```
TORBROWSER-INSTALL-WIN64-10.0-F3C4DF19.pf
TORBROWSER-INSTALL-WIN64-10.0-F3C4DF19.pf.FileSlack
```

To read a pf file, I used [PECmd (Eric Zimmerman tool)](https://ericzimmerman.github.io/#!index.md), the output contains many informations, I only written the most usefull one for us:

```bash
PECmd.exe -f TORBROWSER-INSTALL-WIN64-10.0-F3C4DF19.pf

...
Executable name: TORBROWSER-INSTALL-WIN64-10.0
Hash: F3C4DF19
File size (bytes): 139.504
Version: Windows 10 or Windows 11

Run count: 1
Last run: 2021-04-29 18:22:32
...
```

An intuitive answer could to answer 1, and that's what I did. But the program we are inspecting is not Tor Brower, it's **Tor Installer**. As we don't have a pf file for Tor Browser, the user never launched this program, just installed it.

## Q6. What is the suspect's email address?

Autopsy provides an automatic email parser under `Analysis Result > Keywords Hits > Email addresses`, but there is many email addresses, we have to correlate with other datas. There is no emails sofware installed nor emails datas in the computer. The only way that the user uses email is throught a web browser, let's investigate into `Data Artifacts > Web History`:

```
https://protonmail.com/login
https://protonmail.com/inbox
```

Unfortunately, we have 6 proton emails addresses in the computer:

```
contact@protonmail.com
dreammaker82@protonmail.com
notify@protonmail.com
security@protonmail.com
support@protonmail.com
username2@protonmail.com
```

Using the `Data Artifacts > Web Form Autofill`, which are the web browser form saved values, we can see a known username `dreammaker82`. Theses datas can be accessed at: `Users/John Doe/AppData/Local/Google/Chrome/User Data/Default/Web Data`.

## Q7. What is the FQDN did the suspect port scan?

Which program can be used to make network scanning? Nmap of course, and it's installed on the computer `Data Artifacts > Installed Programs`.

So now the question is, how can I find the nmap history or equivalent? I was unable to find some nmap logs or something else. But as nmap is a CLI-tool, you can retrieve the command launched in the Power-Shell history.

The history is located at: `Users\John Doe\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`. Here, you can see that the user perform this network scanning:

```bash
ping dfir.science
nmap dfir.science
```

## Q8. What country was picture "20210429_152043.jpg" allegedly taken in?

My first idea was to made a reverse Google/Yandex image search. I found nothing interesting, so I decided to inspect metadatas. First, I extracted the file with Autopsy or FTK Imager. **Using the Windows property will not show all metadatas, you have to use a dedicated tool.** I found this one: [Metadata2go](https://www.metadata2go.com/). Juste upload the photo and the tool will process the image for you. There is many metadatas and some of them are really interesting:

```
gps_latitude:   16 deg 0' 0.00" S
gps_longitude:  23 deg 0' 0.00" E
gps_position:   16 deg 0' 0.00" S, 23 deg 0' 0.00" E
```

[Itilog.com](https://www.itilog.com/) will give you the location from the coordinates. Please be sure how the coordinate system works, otherwise the tool will give you a wrong country (Chad). As the latitude is with the letter 'S', you have to provide this value too. The country is Zambia.

## Q9. What is the parent folder name picture "20210429_151535.jpg" was in before the suspect copy it to "contact" folder on his desktop?

I was not able to answer this question, but it's quite easy, I quote this another writeup from [jamesgibbins.com](https://www.jamesgibbins.com/posts/cyberdefenders-africafalls/#11-what-is-the-user-john-does-windows-login-password)

> Looking at the file metadata in Autopsy, we can see it was taken by an LG Electronics LM-Q725K, which is a smartphone. If we look in USB Device Attached, we can see it there too: LG Electronics, Inc. LM-X420xxx/G2/G3 Android Phone (MTP/download mode). Background knowledge time, many cameras store photos in a DCIM, or a subfolder of this folder. If we search for DCIM, we get a Shell Bags Artifact relating to this photo: `My Computer\LG Q7\Internal storage\DCIM\Camera`

## Q10. A Windows password hashes for an account are below. What is the user's password? Anon:1001:aad3b435b51404eeaad3b435b51404ee:3DE1A36F6DDB8E036DFD75E8E20C4AF4:::

I don't know why this question pop, its not related at all to the challenge. Use an online tool, or John or Hashcat.

## Q11. What is the user "John Doe's" Windows login password?

Users informations, included NTLM password, hashes are stored in `SAM` hive. But Autopsy can't read them (maybe because it's protected?), we have to be more more "aggressive". [Mimikatz](https://github.com/gentilkiwi/mimikatz/) can be used to dump NTLM hashes.

Once you have deactivated securities (Mimikatz is a well-known offensive tool, flagged by every security solution), let's launch the tool. It was the first time I used this tool, apologies for mistakes. Before dumping hashes, you have to export `SAM` and `SYSTEM` hives.

```
  .#####.   mimikatz 2.2.0 (x86) #19041 Sep 19 2022 17:43:26
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # lsadump::sam /system:<SYSTEM file path> /sam:<SAM file path>

...
RID  : 000003e9 (1001)
User : John Doe
  Hash NTLM: ecf53750b76cc9a62057ca85ff4c850e

Supplemental Credentials:
* Primary:NTLM-Strong-NTOWF *
    Random Value : 7844054d945112afaa36825b3ffcedfc
...
```

The tool will display all user information, including sensitive datas such as password hashes. Now, you just have to launch a password cracker like John or Hashcat to brute-force the hash and retrieve the password. It's an easy password so you should not wait too long.