# HireMe

The scenario:

> Karen is a security professional looking for a new job. A company called "TAAUSAI"  offered her a position and asked her to complete a couple of tasks to prove her technical competency. As a soc analyst Analyze the provided disk image and answer the questions based on your understanding of the cases she was assigned to investigate.

## Abstract

In the field of digital forensics, Autopsy shines as a formidable tool that plays a crucial role in analyzing complex scenarios. Autopsy's versatility becomes evident right from the start. It's harnessed to explore every part of the image, and parse lot of file format. The capability to handle AD1 files, known for their complexity, is highlighted, with a recommended approach involving mounting AD1 files with FTK Imager and then leveraging Autopsy's innate parsing tools.

Throughout the investigation, Autopsy serves as a central hub for extracting and processing crucial data artifacts. The administrator's username, OS's build number, hostname, and other key information are effortlessly. The power of Autopsy to expose web form autofill data, leading to insights like postal codes, or retrieve mail conversations showcases its utility in deriving meaningful information from user activities. The tool's integration with external tools like Eric Zimmerman's tools provides additional pathways for data extraction, enhancing the investigation's depth.

In sum, Autopsy emerges as an indispensable digital forensic tool. Its comprehensive features, and inherent ability to handle various data artifacts make it an asset for analysts and investigators.

![Autopsy software interface.](img/write-up/hireme/autopsy-interface.png)

<figcaption>Autopsy software interface.</figcaption>

---

First, mount the image using [FTK Imager](https://www.exterro.com/ftk-imager) (Windows) or command lines tools (Linux), follow this very good tutorial: [Guide: mounting challenge disk image on Linux.](https://bwiggs.com/posts/2021-07-25-cyberdefenders-hacked/).

**Important**: AD1 files are painfull to use with Autopsy, If you want to analyse AD1 file with this software, you will have troubles. As Autopsy is very powerfull and embed various parsing tools, it's convenient to use it. My method is to:

1. Mount AD1 file with FTK Imager

2. Export files using FTK Imager (right-click > Export files)

3. In Autopsy, Add Data Source > Logical Files > Add (select the previously exported files)

4. Data ingestion should now works :)

## Abstract

Autopsy <3

## Q1. What is the administrator's username?

To see users just open `Users` directory you will see all the user. Hopefully there is only one user, Karen. To confirm, I have two methods:

- analyse Windows Registry with [Eric Zimmerman's tools](https://ericzimmerman.github.io/) or [Autopsy](https://www.autopsy.com/) to parse `SAM\Domains\Account\Users\Names` hive. With Autopsy:

![Parsing SAM hive using Autopsy.](img/write-up/hireme/sam-hive-autopsy.png)

<figcaption>Parsing SAM hive using Autopsy.</figcaption>

- go to OS Account in Autopsy, once ingestion is finished:

![Explore OS Account using Autopsy.](img/write-up/hireme/os-account-autopsy.png)

<figcaption>Explore OS Account using Autopsy.</figcaption>

Karen is the only non-disabled account.

## Q2. What is the OS's build number?

Searching on internet I found this article: [How to find the Windows 10 build number you are running](https://winaero.com/how-to-find-the-windows-10-build-number-you-are-running/). It provides the reg key to fint build number: `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
`. Using Autopsy you can easily retrive it:

![Build number using Autopsy.](img/write-up/hireme/build-number.png)

<figcaption>Build number using Autopsy.</figcaption>

## Q3. What is the hostname of the computer?

Using Autopsy, the value is located at `Data Artifacts > Operating System Information` in the Name column.

## Q4. A messaging application was used to communicate with a fellow Alpaca enthusiest. What is the name of the software?

The simpler is to use Autopsy, search under `Data Artifacts > Installed Programs`. You will find Skype.

Or, you can searce in: `Horcrux.E01_Partition 3 [3122MB]_PacaLady [NTFS]/[root]/` and you will find `Skype-8.41.0.54.exe`.

Third method, searching in the hive: `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths` you will see Skype too.

Note that first and third methods are equivalent, as Autopsy parse hive and key to provides theses informations.

## Q5. What is the zip code of the administrator's post?

The free hint is very usefull:

> Check browser data.

Under `Data Artifacts > Web Form Autofill` there is a saved value: `postal: 19709`. 

The database read by autopsy contains the values ​​which are automatically proposed to you when you wish to fill out a form on your browser. You can also manually browse this database, it's located at: `/LogicalFileSet1/Horcrux.E01_Partition 2 [32216MB]_NONAME [NTFS]/[root]/Users/Karen/AppData/Local/Google/Chrome/User Data/Default/Web Data`.

## Q6. What are the initials of the person who contacted the admin user from TAAUSAI?

Under `Data Artifacts > E-Mail Messages > Default ([Default]) > Default`, you will see emails data such as conversations and attached documents. There is lot of interesting informations between `Karen Alice` and `Alpaca Activists`. If you don't have Autopsy, you can use [OST Viewer](https://www.sysinfotools.com/recovery/ost-file-viewer.php) you have to find Outlook file and open them with the tool, you will be able to read the emails.

By opening the most recent mail, you will see this:

```
From: Karen Alice <klovespizza@outlook.com> 
Sent: 22 March 2019 23:37
To: Karen Alice <klovespizza@outlook.com>
Subject: RE: Interested in the job
 
MS
 
Things didn’t go as planned with Bob. I attached a copy of our chat history. The password is pacalove
```

`MS` looks like initials, reading a little bit more mails will inform you that the Alpaca Activists guy is named Michael, we were right, this is initals we looked for.

## Q7. How much money was TAAUSAI willing to pay upfront?

By reading the mail conversation, you will find this mail:

```
Hi Michael,
 
I’m so sorry for the delay. I meant to send you a message earlier, but I’ve been incredibly busy with my kids and was having issues with Outlook. I’ll be honest with you, I have computer knowledge (I know all about power buttons, how to clean keyboards, and am a pro on internet explorer (I found a way to have Bing and Yahoo as a search bar on my internet explorer web platform)) but don’t know enough to where I think I would be of use for you.
 
I am definitely interested in this opportunity, and want to know what it may require as $150,000 seems like a lot for someone who isn’t too skilled on computers.
 
-Karen
```

## Q8. What country is the admin user meeting the hacker group in?

By reading the conversation, you will find this mail:

```
Hey there!
 
So here's what we need you to do:
 
We have been conducting an investigation on Bob Redliubeht (the CEO of Alpacamybags Luxury Alpaca handbags) and we believe he's been mistreating some of his Alpacas. We have heard complaints that he refuses to provide Alpacas with scarfs and beanies during the winter! 
 
What we need you to do is gain his trust and then hack his machine. We will give you more information about this in person. Meet us here "27°22’50.10″N, 33°37’54.62″E"
```

Using Google maps, or just Google you will find the country: Egypt.

## Q9. What is the machine's timezone? (Use the three-letter abbreviation)

I used this article: [Manual Identification of Suspect Computer Timezone](https://www.digital-detective.net/time-zone-identification/).

First, note the `SYSTEM\Select\Current` key value. In this challenge it's `0x1`. After, go to `SYSTEM\ControlSet00<previous_key_value>\Control\TimeZoneInformation`. In our example, there is only one `ControlSet` hive but it's good to know how to deal with several.

![Current timezone.](img/write-up/hireme/timezone.png)

<figcaption>Current timezone.</figcaption>

## Q10. When was AlpacaCare.docx last accessed?

**Be careful** to be in UTC timezone (cf previous question) if you browse the file with your system explorer, instead of FTK Imager. Otherwise, the timestamp will be converted into your timezone and the answer will be wrong.

There is a good advantage to use FTK Imager as it does not convert automatically the timezone into your current one. But I was unable to retrieve the timestamp with Autopsy, some of them was set to 0, even if I reset the timezone.

![Using FTK Imager to retrieve the last accessed timestamp.](img/write-up/hireme/alpacacare-timestamp.png)

<figcaption>Using FTK Imager to retrieve the last accessed timestamp.</figcaption>

## Q11. There was a second partition on the drive. What is the letter assigned to it?

I found two methods:

- using Autopsy just go under `Data Artifacts > Recent Documents`, you will see that there is two partitions; `C`, the Windows native one, and `A`.

- browse `SYSTEM\MountedDevices` value and you will see three partitions: `C`, `D` and `A`. `D` it's quite different as it looks like a VMWare installation drive (see value).

## Q12. What is the job position offered to Karen? (3 words, 2 spaces in between)

Go through the emails again (see Q6) and read the conversation, you will find this mail:

```
On Sun, Mar 17, 2019 at 2:34 AM Karen Alice <klovespizza@outlook.com> wrote:
Hi Michael,
 
The answer is TheCardCriesNoMore
 
-Karen
 
 
From: Alpaca Activists <taausai@gmail.com> 
Sent: 16 March 2019 23:19
To: Karen Alice <klovespizza@outlook.com>
Subject: Re: Interested in the job
 
Hi Karen,
 
No worries, it happens! We're just happy to finally hear from you.
 
So I may have lied, my manager is saying that before we can offer you a job, we need to give you a quick test. Can you tell me what the answer to the thing at the bottom is?
 
VGhlQ2FyZENyaWVzTm9Nb3Jl
 
 
Good Luck!
Michael
```

## Q13. What is the job position offered to Karen? (3 words, 2 spaces in between)

Go through the emails again (see Q6) and read the conversation, you will find this mail:

```
Karen,
 
WOW! That was quick! I have confirmed with my manager that that answer is correct. We didn't expect you to know the answer, but were really testing you on your ability to quickly learn new things that may be a bit out of your comfort zone. 
 
The job position we think you'll be an awesome fit for is an entry level cyber security analysts. We want someone who's willing to learn and don't really care about what you know coming in. We'll be in touch with more information about what this job entails (and the set up involved with getting you payed), but wanted to give you some material to study in the mean time.
...
```

## Q14. When was the admin user password last changed?

User's informations are saved into `SAM` hive. Firstly, I browsed `SAM\Domains\Account\Users\Names\Karen` but I found nothing interesting. Using Autopsy, this hive looks bit empty, **it's because the `SAM` hive in encrypted**. To decrypt it, use [Reg Ripper 3.0](https://github.com/keydet89/RegRipper3.0). It's very easy to use and it will produce a report, here is the Karen's part:

```
Username        : Karen [1001]
SID             : S-1-5-21-1649836244-3544936428-1548601679-1001
Full Name       : 
User Comment    : 
Account Type    : 
Account Created : Sat Jan 26 19:40:22 2019 Z
Name            :  
Password Hint   : forensics is boring
Last Login Date : Fri Mar 22 23:22:01 2019 Z
Pwd Reset Date  : Thu Mar 21 19:13:09 2019 Z
Pwd Fail Date   : Thu Mar 21 19:14:49 2019 Z
Login Count     : 32
  --> Password does not expire
  --> Password not required
  --> Normal user account
```

I converted the `Pwd Reset Date` into the attended format: 03/21/2019 19:13:09.

## Q15. What version of Chrome is installed on the machine?

It's very easy with Autopsy, under `Data Artifacts > Installed Programs` you will find Google Chrome and all related datas such as its version: `Google Chrome v.72.0.3626.121`. Without Autopsy, it's a little bit more complicated. Obviously, the answer is in the `SOFTWARE` hive but not directly under `Google` key. In fact, it's quite hard to find the good value as it's located in an unintuitive location: `SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Google Chrome`.

## Q16. What is the HostUrl of Skype?

The question is quite strange, it's unclear what you should to looking for. **You need to find the URL where Skype was downloaded.**

First, I searched in the `SOFTWARE` hive using [Registry Explorer EZ](https://ericzimmerman.github.io/#!index.md) because this software allows you to search strings inside hive. Unfortunately I didn't found the answer. My second idea was to extract the Skype exe file and extract strings. Once again, I didn't found the answer.

In fact, I found two solutions:

- browse internet history located at `Users/Karen/AppData/Local/Google/Chrome/User Data/Default/History` in the `download_url_chains` table and you will see the URL.

- search for alternate data stream (ADS), more information [here](https://www.malwarebytes.com/blog/news/2015/07/introduction-to-alternate-data-streams).

I prefer the second method, as **ADS are used by malwares**, so it's more usefull in the DFIR learning journey. In the case of a downloaded file, an ADS file named `Zone.Identifier` could be created to store the host URL. To see it, I used FTK Imager. Click on the file and the `Zone.Identifier` will be displayed in the right panel.

![Alternate Data Stream (ADS) named Zone.Identifier, contains datas such as HostUrl.](img/write-up/hireme/ads.png)

<figcaption>Alternate Data Stream (ADS) named Zone.Identifier, contains datas such as HostUrl.</figcaption>

You can also use other tools and even command line tool to see ADS. I guess you have to mount properly the image and/or perform other steps before. If you are interested in, take a look here: [Forensic Analysis of the Zone.Identifier Stream](https://www.digital-detective.net/forensic-analysis-of-zone-identifier-stream/).

## Q17. What is the domain name of the website Karen browsed on Alpaca care that the file AlpacaCare.docx is based on?

Another vague question. Open the document and you will see several links inside, like `rcfalpaca.com` or `palominoalpacafarm.com`. The second one is not directly displayed, you will find it in the page footer. Using the answer pattern provided by Cyberdefenders you will find that the correct answer is the second one.

Note: you can also observe `palominoalpacafarm.com` in the web history.
