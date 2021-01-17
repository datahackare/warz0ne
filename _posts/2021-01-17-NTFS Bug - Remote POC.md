---
title: Windows 10 NTFS bug remote POC
published: true
---
## [](#header-2)Disclaimer
This is bug that is still active in any Windows 10 version running in the wild. This blog-post and the recorded video is a POC that the bug will work in an easy remote setup. I  take no responsibility if you brake any system. This is just for demo purpose in a controlled environment.

## [](#header-2)The blog post
Bleeping computer published a <a href="https://www.bleepingcomputer.com/news/security/windows-10-bug-corrupts-your-hard-drive-on-seeing-this-files-icon/">blog post</a> the other day, talking about a bug in Windows 10 that will brake the NTFS partition of any partition that a special string of code was run on.

I recommend reading the full article to get more information about this bug.

Full cred to <a href="https://twitter.com/jonasLyk">Jonas L</a> for disclosing this.

## [](#header-2)Remote POC
If you want to try this out I strongly recommend doing this in a controlled environment inside a virtual machine. 

I read the article and wanted to try if I could trigger the bug by putting the string of code inside a simple img-tag in a html-file, running on a python3 local webserver.

### [](#header-3)HTML IMG tag

```
<html>
<head>
</head>
<body>
<img src="THE_STRING_OF_CODE"> 
<p>This is just a test</p>
</body>
</html>
```

### [](#header-3)The tested Windows system
The Windows 10 system i tried this on is not a fully updated one, i misses maybe 6 months of updates, including the 20H2 feature update. The machine was running in a Virtualbox environment. 

### [](#header-3)The crash
As described in the original blog post, the bug triggered and a notification in Windows showed that something is wrong and wanted med to do a reboot.

During the reboot, the filesystem repair mechanism was triggered, but in my test it didn't actually destroy the file system. Windows was able to fully boot.  

<iframe id="lbry-iframe" width="560" height="315" src="https://lbry.tv/$/embed/-Critically-underestimated--NTFS-vulnerability%2C-Windows-10---Remote-exploitable-POC-/d20b2ec73e54372e73b9c00df61c0430cceb04e5?r=U7yBQpkxKswNqd3DGqrYCHPfGaGg6YhW" allowfullscreen></iframe>
