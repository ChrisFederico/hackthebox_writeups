# Path to User #
## Network scan ##
I scanned the network with nmap on target IP. This machine has very few open ports since it has port 22 and port 80.
![](/openadmin/img/nmap.PNG)

The web page has a login form but I didn't find anything useful, so I started enumerating directories.

## Directory enumeration ##
I ran dirbuster from the GUI looking for some interesting directory
![](/openadmin/img/dirb.PNG)

There were many directories but traveling through them on browser led nowhere, except for a single directory named "ona".
This is a home page for OpenNetAdmin service.
![](/openadmin/img/ona_page.PNG)

The most interesting part is that this version is **not** updated. This was proven to be my initial foothold, so I started navigating
the web in search of exploits.

## ONA exploitation ##
(this)[https://www.exploit-db.com/exploits/47691] was the exploit I was looking for, just running this script passing the target URL as argument did the trick, I got a shell on my terminal, but I couldn't traverse the directory. So I began listing every folder in order to find something. I spent a lot of time doing this because I couldn't use cd or other commands, however I was able to list two users: (jimmy and joanna) and cat file contents on the terminal. Since there were many configuration files I started analyzing each of them.

![](/openadmin/img/php_conf.PNG)

This file might have some credentials so I cat its content and this was what I found:

![](/openadmin/img/db_conf.PNG)

Not bad! A database password. I tried to use it somewhere but I couldn't find a database, so I tried to log from the shell using *su* command but I had no luck. I did manage to log in using the SSH service. The correct user was jimmy, from here I cat the user flag. Then was time to escalate to root.

# Path to Root #
## Privilege escalation ##
This part was quite tricky but not very hard. I basically continued my enumeration following [this guide](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/) (I suggest to always have the guide at hand especially if you are a beginner). 
I started listing all processes running on the machine with netstat: there is an unusual process running on port 52846

![](/openadmin/img/process.PNG)

I listed all files owned by jimmy using *find* command, there were **a lot** of files but, in the end, there was a non-default directory called *internal* 
with three .php files inside.

![](/openadmin/img/main_php.PNG)

__(on the background you can admire a stunning draft version of this write-up)__

By the way, this file is extremely useful because it prints the RSA key for the user joanna, still, I had to find a way to call the process, maybe it was running on the same port we saw before.

I tried *curl*ing the same machine on that port, bingo!
```
curl localhost:52486/main.php
```

This command will print the RSA key for joanna. Then I cracked the key using ssh2john, returning me a hash. 
![](/openadmin/img/john.PNG)

Then I used John the Ripper to crack the resulting hash and get a plain-text password. As you can see, using rockyou as wordlist gave me
the password "bloodninjas". 

The last part of this machine is to use this passphrase in order to login on SSH with the user joanna.

![](/openadmin/img/ssh_joanna.PNG)

Enumerating with joanna's account revealed that she can use the executable /bin/nano on /opt/priv using root privileges. Running nano
on that file and pressing CTRL+R and CTRL+X we can enter in command execution mode, now **we can run any command with root privileges**!
Just cat the content on the root.txt in /root directory and job is done.

![](/openadmin/img/root.PNG)

