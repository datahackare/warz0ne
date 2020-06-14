---
title: Writeup - HackTheBox - Monteverde
published: true
---

This machine from HackTheBox is categorized as an “medium” Windows
machine.
The machine is a Domain Controller. At first i had some problems with with my nmap scans,
maybe because of a firewall, so I had to modify my nmap scan to get any results. With
rpcclient I was able to get some usernames that I tried against
SMB. One account had an easy-to-guess password and I was able to
get into an SMB share. There was a file, azure.xml. This file
contained the password for the user mhope. This user was able to
log in on the domain with winrm. When logged in, mhope was able to
dump the azure credentials for the administrator.

## [](#header-2)Enumeration

### [](#header-3)Nmap
I started with an nmap scan to get a list of running services on
the machine.

```
sudo nmap -sS -v -v -Pn 10.10.10.172
```

![](Pictures/Monteverde/nmap1.png)

I got results but I wanted some more information from ldap so I did another nmap scan:

```
nmap -p 389 --script ldap-rootdse -Pn 10.10.10.172
```

![](Pictures/Monteverde/nmap2.png)
I now got the name of the Domain and hostname.

### [](#header-3)Rpcclient
I wanted to see if I could login with rpc to get some user account
names. I tried to log in without a username.

```
rpcclient -U ‘’ 10.10.10.172
```

![](Pictures/Monteverde/rpcclient.png)
I got some usernames and I put them in a file, user.txt.

## [](#header-2)Exploitation

### [](#header-3)Metasploit – smb_login and smbmap
I used Metasploit - smb_login to see if any of the accounts had a
weak password. I tried to get access with the user, if username
and password was the same (if that make’s sense).

![](Pictures/Monteverde/meta-smb_login.png)

It turned out that the account SABatchJobs used SABatchJobs as
password.
I wanted to map all shares and used smbmap like this:

```
smbmap -u SABatchJobs -p SABatchJobs -H 10.10.10.172
```

![](Pictures/Monteverde/smbmap.png)
Here I got some shares. I tried to login to everyone of them that
I had access to read.

![](Pictures/Monteverde/smbclient1.png)

![](Pictures/Monteverde/smbclient2.png)

I had read access to the SYSVOL and users$ share.
Inside users$\mhope\ I found the file azure.xml and downloaded
that.

```
get azure.xml
```

![](Pictures/Monteverde/azure-xml.png)

The file contains a password. A wild guessing was that it might
belong to the user mhope.

### [](#header-3)Evil-WinRM - user.txt
I tried to log in with mhope with winrm, just for fun.

```
evil-winrm -u mhope -p 4n0therD4y@n0th3r$ -i 10.10.10.172
```

![](Pictures/Monteverde/evil1.png)

It worked! Now I got the user flag.
### [](#header-3)Enumeration of mhope’s home folder
I did some enumeration inside the user mhope’s home folder.

![](Pictures/Monteverde/enum-mhope.png)

I saw something about azure. This is the second hit that this
domain might be using azure.
So I Googled a little... I Googled a little more... and then I did
some more Googling. After some hours, I found something useful
that actually turned out to work in the end.

### [](#header-3)Azure AD Connect Database Exploit (Priv Esc)
What I found was a program written by vbscrub that might be the
solution. Link: https://vbscrub.com/2020/01/14/azure-ad-connect-database-exploit-priv-esc/

![](Pictures/Monteverde/azure-AD.png)

A bit down on this blogpost is the shellcode for the program and
also a compiled 3 binary (AdDecrypt). I downloaded the binary.
Download link: https://github.com/VbScrub/AdSyncDecrypt/releases

I uploaded the files AdDecrypt.exe and mcrypt.dll to C:\Users\
mhope\Documents\.
In the blogpost vbscrub writes that the binary has to be run
inside of the C:\Program Files\Microsoft Azure AD Sync\Bin folder.
The problem was that I could not write files there.
My solution to this problem was that I changed the path’s for the
user hmope. Just like in Linux you can set env paths. In Windows
it can look like this:

```
$env:Path += ";C:\Users\mhope\Documents\AdDecrypt\"
```

After this I got my self inside the Azure Bin folder and ran his
program like the following:

```
cd ‘C:\Program Files\Microsoft Azure AD Sync\Bin’
```
```
AdDecrypt.exe -FullSQL
```

![](Pictures/Monteverde/admin-creds.png)

Here I got the Administrators password. My next step was to try
logging in with the admin account with Evil-WinRM.

### [](#header-3)Evil-WinRM – root.txt
I used the same method as before:

```
evil-winrm -u administrator -p d0m@in4dminyeah! -i 10.10.10.172
```

![](Pictures/Monteverde/root-flag.png)

Here I got the root.txt flag.


I recorded a video to show this.



<iframe width="1280" height="720" src="https://www.youtube.com/embed/JrVURlg7Ngs" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
