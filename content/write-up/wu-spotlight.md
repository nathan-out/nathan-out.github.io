# Spotlight

The scenario:

> Spotlight is a MAC OS image forensics challenge where you can evaluate your DFIR skills against an OS you usually encounter in today's case investigations as a security blue team member.

I tried to extract files with FTKImager then perform analysis with Autopsy (you can find more detailed steps in my previous WU) but unfortunately it can't parse many artifacts. It's maybe possible to setup Autopsy to process macOS images but I wanted to learn a new tool: [mac_apt](https://github.com/ydkhatri/mac_apt).

![MacOSX Catalina - the version in this forensic case.](/img/write-up/spotlight/macos-catalina.jpg)

<figcaption>MacOSX Catalina - the version in this forensic case.</figcaption>

## Abstract

The article presents a MAC OS image forensics challenge known as "Spotlight." The challenge assesses digital forensics and incident response (DFIR) skills in dealing with macOS systems. I documents my exploration using various tools, including FTKImager, Autopsy, and mac_apt, while tackling a series of questions related to the challenge.

The questions cover a range of topics such as identifying the macOS version, extracting information from image files, examining Safari bookmarks, retrieving data from Notes, obtaining network information, investigating quarantined items, identifying app installations, tracking permission changes, uncovering steganography, accessing user history, and finding user UUIDs.

The article showcases my problem-solving approach, tool utilization, and detailed steps to answer each question, offering insights into macOS forensics challenges and methodologies.

## Q1. What version of macOS is running on this image?

I wanted to run all the `mac_apt` plugins using this command:

```bash
> ios_apt.exe -i APFS\\Vol4\\root -o ios_apt_output\\fsevents ALL

Output path was : ios_apt_output\fsevents
MAIN-INFO-Started iOS Artifact Parsing Tool, version 1.5.8.dev (20230617)
MAIN-INFO-Dates and times are in UTC unless the specific artifact being parsed saves it as local time!
MAIN-INFO-Python version = 3.10.7 (tags/v3.10.7:6cc6b13, Sep  5 2022, 14:08:36) [MSC v.1933 64 bit (AMD64)]
MAIN-INFO-Pytsk  version = 20221228
MAIN-INFO-Pyewf  version = 20230212
MAIN-INFO-Pyvmdk version = 20221124
MAIN-INFO-PyAFF4 version = 0.31
MAIN.HELPERS.MACINFO-INFO-iOS version detected is: Mac OS X (10.15) Build=19A583
...
```

The output provides lot of informations but at the top there is the Mac OS version.

## Q2. What "competitive advantage" did Hansel lie about in the file AnotherExample.jpg? (two words)

It's more like a CTF question than a Blue Team one.

If you are familiar with Unix/MacOs filesystem you will easily find `AnotherExample.jpg` (`APFS\Data\root\Users\Shared`). First, I thought that the advantage was 'Futur Phone', as it's written on the image. Instead, you had to extracts strings from the file and you will find the answer inside.

## Q3. How many bookmarks are registered in safari?

Autopsy can parse few things from the dump and bookmarks are part of it! Or, you can find this information here: `macOS Catalina - Data [volume_0]/root/Users/hansel.apricot/Library/Safari/Bookmarks.plist`.

## Q4. What's the content of the note titled "Passwords"?

I tried to recursively search for a filename named "Passwords" but it's located in a db file, as you will see bellow. Googling where notes are located in MacOs I found this path: `~/Library/Containers/com.apple.Notes/Data/Library/Notes`.

Open `NoteStore.sqlite` with your favorite tool and you will find in the `ZICCLOUDSYNINOBJECT` table, in the `ZTITLE1` column, a note named `Passwords`. The `ZSNIPPET` does not show anything, maybe because the note is empty.

## Q5. Provide the MAC address of the ethernet adapter for this machine.

I found this very useful website: [MAC FORENSICS PART 4 (MOUNTAIN LION 10.8 – SYSTEM FILE ARTIFACTS)](https://davidkoepi.wordpress.com/2013/07/06/macforensics4/), it provides the path to see MAC address: `/private/var/log/daily.out`. In this file we can see some reports about the system, included network informations:

```
Network interface status:
Name       Mtu   Network       Address            Ipkts Ierrs    Opkts Oerrs  Coll
lo0   16384 <Link#1>                          1072     0     1072     0     0
lo0   16384 127           localhost           1072     -     1072     -     -
lo0   16384 localhost   ::1                   1072     -     1072     -     -
lo0   16384 fe80::1%lo0 fe80:1::1             1072     -     1072     -     -
gif0* 1280  <Link#2>                             0     0        0     0     0
stf0* 1280  <Link#3>                             0     0        0     0     0
en0   1500  <Link#4>    00:0c:29:c4:65:77   372733     0    73025     0     0
```

## Q6. Name the data URL of the quarantined item.

By using Powhershell to recursively search for filename I found this location:

```powershell
>  Get-Childitem .\APFS\ *quarantine* -Recurse

\APFS\Data\root\Users\sneaky\Library\Preferences


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------        4/20/2020   5:58 AM          20480 com.apple.LaunchServices.QuarantineEventsV2
```

This is a SQLite database, open it, the column `LSQuarantineDataURLString` contains the URL where the quarantined file was downloaded: `https://futureboy.us/stegano/encode.pl`.

## Q7. What app did the user "sneaky" try to install via a .dmg file? (one word)

Once again, I used powershell to find all the dmg file (disk image file to store compressed software installer on Mac):

```powershell
>  Get-Childitem .\APFS\ *.dmg -Recurse

Directory: APFS\Data\root\Users\sneaky\.Trash


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------        4/20/2020   5:30 AM       26633714 silenteye-0.4.1b-snowleopard.dmg
```

Silenteye is a software used for steganography; hidding information into files like pictures or sound.

## Q8. What was the file 'Examplesteg.jpg' renamed to?

I had some troubles with `mac_apt`, for technical reasons I couldn't used source file (Python problems during requirements installations) and the exe file returns an error during my investigation (see bellow). I had to trick the software to go forward.

A good idea is to use the `FSEVENTS` plugin from `mac_apt`, according to the doc :

> Reads file system event logs (from .fseventsd)

As we have two partitions, I run the plugin on `Vol4`:

```bash
> ./ios_apt.exe -i APFS\Vol4\root -o ios_apt_output\fsevents FSEVENTS
...
```

In `ios_apt_output\fsevents` a file was created: `ios_apt.db`. It's a SQLite db:

![MacOS file system event logs view.](/img/write-up/spotlight/event_db_1.png)

<figcaption>MacOS file system event logs view.</figcaption>

`Filepath` is the key here, we can try a SQL query to find any information related to `Examplesteg.jpg`:

```SQL
select * from FsEvents where Filepath like '%Examplesteg.jpg%'

Execution finished without errors.
Result: 0 rows returned
```

There is no data related to our interesting file but let's investigate on the second partition logs:

```bash
> ./ios_apt.exe -i APFS\Data\root -o ios_apt_output\fsevents FSEVENTS

MAIN-ERROR-Could not find iOS system version!
:( Could not find an iOS installation on path provided. Make sure you provide the path to the root folder. This folder should contain folders 'bin', 'System', 'private', 'Library' and others.
```

This is where the trick begins, it looks like a bug for exe file, as source file don't returns this error. The program wants `bin`, `System`, ... let's provides them!

Just copy the missing files from `Vol4/root` to `Data/root` and the tool will not trigger errors! **Be carreful, in real world forensic case, this kind of manipulation can corrupt your evidences for judges or lawyers.**

On this second db we have datas related to our file:

![MacOS file system event logs view for Examplesteg.jpg.](/img/write-up/spotlight/event_db_2.png)

<figcaption>MacOS file system event logs view for Examplesteg.jpg.</figcaption>

Note the `File_ID` (12885043806), and search on it:

![Event logs for file id no 12885043806.](/img/write-up/spotlight/event_db_3.png)

<figcaption>Event logs for file id no 12885043806.</figcaption>

Order the `LogID` column to get the bigger one (i.e: the most recent) in the `Filepath` you have the new file name!

## Q9. How much time was spent on mail.zoho.com on 4/20/2020?

I didn't understand how the system count minutes on a given website. Using Autopsy and web history I could retrieved when user accessed to mail.zoho.com and when he leaved, but timestamp didn't matched with `RMAdminStore-Local.sqlite` db (see other WU). On this db, run the SCREENTIME plugin and you will find durations.

## Q10. What's hansel.apricot's password hint? (two words)

Thanks to this website: [MAC FORENSICS PART 4 (MOUNTAIN LION 10.8 – SYSTEM FILE ARTIFACTS)](https://davidkoepi.wordpress.com/2013/07/06/macforensics4/) you can find the path where this information is stored: `/private/var/db/dslocal/nodes/[user].plist`. Extract and open the pslist file and the hint is Family Opinion.

## Q11. The main file that stores Hansel's iMessages had a few permissions changes. How many times did the permissions change?

As we have the `fsevents` db from `Data` partition, we can use it to count the number of permission changed event. The iMessage db file is located at `/Users/Mac/Library/Messages/chat.db`, according to a quick web search.

I was unable to find this db but we don't need it to answer the question. We just have to search on `chat.db` and `permissionChanged` and count how many times the persmissions was changed.

![iMessages db logs.](/img/write-up/spotlight/event_db_4.png)

<figcaption>iMessages db logs.</figcaption>

## Q12. What's the UID of the user who is responsible for connecting mobile devices?

Kind of tricky question because you have to know several things:

- which kind of user is intended here? Regular one (human user), or "machine" user, i.e user attached to a particular task? Here it's the second user type; "responsible for connecting mobile devices" means that this is a task performed by a dedicated user, or something similar.

- where should I find this kind of information? Maybe the least difficult question to answer here, as we already browsed user informations on Q10 (`private/db/dslocal/nodes/Default/users/`).

- which user should I have to investigate on? Most difficult one to me, on the directory there is tons of users. I searched MacOS mobile devices related user but I found nothing interesting. There is a process named `usbmuxd` which looks interesting because of the "USB":

> USBMUXD is a Mac OS X process that performs USB multiplexing when synchronizing iTunes music libraries with USB music devices like iPods & iPhones.

It sounds good, by opening its plist file (readable in Autopsy) we can see that its UID is 213.

## Q13. Find the flag in the GoodExample.jpg image. It's hidden with better tools.

I followed the first tip, which is to investigate on `FSEVENTS` db (see previous questions), we can find that there is two location for `GoodExample.jpg` (second column is `File_ID`):

```
Users/Shared/GoodExample.jpg    12885043806
Users/sneaky/Downloads/GoodExample.jpg  12885043806
```

Then, I searched on `File_ID`:

```
Users/Shared/GoodExample.jpg
Users/sneaky/Downloads/Example.jpg
Users/sneaky/Downloads/Examplesteg.jpg
Users/sneaky/Downloads/Examplesteg.jpg.download/Examplesteg.jpg
Users/sneaky/Downloads/GoodExample.jpg
```

The interesting thing to me is that most of the file are located in `Downloads` directory and have the same `SourceModDate` timestamp (2020-04-20 03:19:45). Maybe we should take a look at the users web history (Autopsy).

Few hours before we can see several Google search:

- hide information in jpg

- How to hide data within an image

And, two websites related to steganography:

- https://null-byte.wonderhowto.com/how-to/steganography-hide-secret-data-inside-image-audio-file-seconds-0180936/

- https://www.computerhope.com/issues/ch000861.htm

Obviously, the user is not familiar with steganography and he used a tool like steghide. Searching for a reverse steghide online tool, I found this website: [Steganographic Decoder](https://futureboy.us/stegano/decinput.html). Upload the file and you will find the answer!

Conclusion: if you want to hide information, use very unusual algorithms!

## Q14. What was exactly typed in the Spotlight search bar on 4/20/2020 02:09:48

Most of information found on Spotlight forensic was to find the Spotlight's db located at `~/.Spotlight-V100/` but this file is not in this dump. I used a recursive search to locate all the spotlight related filename:

```powershell
> Get-Childitem .\APFS\ *.spotlight* -Recurse

Directory:
    APFS\Data\root\Users\sneaky\Library\Application
    Support\com.apple.spotlight


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
------        4/20/2020   5:44 AM            687 com.apple.spotlight.Shortcuts
```

There is one of my results, open this file and you will get the answer in the `key` tag:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>silent</key>
	<dict>
		<key>DISPLAY_NAME</key>
		<string>silenteye-0.4.1b-snowleopard_installer</string>
		<key>LAST_USED</key>
		<date>2020-04-20T02:44:27Z</date>
		<key>URL</key>
		<string>/Applications/silenteye-0.4.1b-snowleopard_installer.app</string>
	</dict>
	<key>term</key>
	<dict>
		<key>DISPLAY_NAME</key>
		<string>Terminal</string>
		<key>LAST_USED</key>
		<date>2020-04-20T02:09:48Z</date>
		<key>URL</key>
		<string>/System/Applications/Utilities/Terminal.app</string>
	</dict>
</dict>
</plist>
```

An another way is to recursively search on the given date and open the shortcut file:

```powershell
>  findstr /spin /c:"4-20" .\APFS\*

...
.\APFS\Data\root\Users\sneaky\Library\Application Support\com.apple.spotlight\com.apple.spotlight.Shortcuts:19:                 <date>2020-04-20T02:09:48Z</date>
...
```

## Q15. What is hansel.apricot's Open Directory user UUID?

Apple Open Directory is the LDAP directory service model implementation from Apple according to Wikipedia. My intuition was to use the `generateduid` value located in `hansel.apricot.plist` because it match the pattern and it's an UID, by chance, it was the answer. Howerver, I found this topic: [How is the UUID for a user used by macOS? - Ask Different](https://apple.stackexchange.com/questions/318388/how-is-the-uuid-for-a-user-used-by-macos):

> The UUID (aka GeneratedUID, so Google that one instead) is used by Open Directory Services on macOS

`5BB00259-4F58-4FDE-BC67-C2659BA0A5A4`