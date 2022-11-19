---
layout: post
title: "AutoHotkey - brilliant tool that can make your life easier"
description: "A few weeks ago I decided to install a brilliant tool which is AutoHotkey. Since then I have used it every day. See how it can help you."
date: "2014-05-17 +0100"
tags: productivity AutoHotKey
image: /assets/images/posts/001/autohotkey-small.png
---
About a year ago after reading a blog post by Maciej Aniserowicz [Nie będziesz brał myszki, wroga swego, nadaremno][devstyle-post] I decided to install and try a variety of tools that he describes there. I was playing with [VistaSwitcher][vista-switcher], [Launchy][launchy] and [AutoHotKey][ahk].

After I installed AHK, I decided to use it for some simple tasks I repeat every day that are easy to script.

I searched a little in the depths of the internet and started to create a new script. I put shortcuts there to most frequently run applications, shortcuts to files that I often edit and commands that I run from time to time in CMD.

I created something like this:

```
#z:: ; WIN+Z
Input, userKey, I, {Enter}

; RUN
if userKey = rvs1 ; RUN - VISUAL STUDIO 2012 - PROJECT 1
	Run C:\Program Files (x86)\Microsoft Visual Studio 11.0\Common7\IDE\devenv.exe C:\pathtoprojects\Proj1.sln
else if userKey = rvs2 ; RUN - VISUAL STUDIO 2012 - PROJECT 2
	Run C:\Program Files (x86)\Microsoft Visual Studio 11.0\Common7\IDE\devenv.exe C:\pathtoprojects\Proj2.sln
else if userKey = rdp ; RUN - DOT PEAK
	Run C:\Program Files (x86)\JetBrains\dotPeek\v1.1\Bin\dotPeek64.exe
else if userKey = rssms ; RUN - SQL SERVER MANAGEMENT STUDIO
	Run ssms	
else if userKey = rnpp ; RUN - NOTEPAD ++
	Run C:\Program Files (x86)\Notepad++\notepad++.exe

; EDIT FILE
else if userKey = efhosts ; EDIT FILE - HOSTS
	Run C:\Program Files (x86)\Notepad++\notepad++.exe C:\Windows\System32\drivers\etc\hosts
else if userKey = efahks ; EDIT FILE - AUTO HOT KEY SCRIPT
	Run C:\Program Files (x86)\Notepad++\notepad++.exe C:\pathtoyourscript.ahk

; COMMAND	
else If userKey = crap ; COMMAND - RECYCLE APP POOLS
	Run %systemroot%\system32\inetsrv\appcmd.exe recycle apppool /apppool.name:".NET v4.5"
else if userKey = crahks ; COMMAND - RELOAD AUTO HOT KEY SCRIPT
	Reload
else if userKey = rcmd
	Run cmd
return
```

I push `WIN+Z` and then type `rvs1` and hit `enter` then Visual Studio starts with an open project on which I work for a long time.

The cool thing is that I set the script to run with elevated privileges, so all commands in it run with administrator rights. Thanks to this Visual Studio or hosts.etc file opens correctly.

I encourage you to play with the AHK and read materials on the wiki because this tool is much more powerful than I show you here.

[devstyle-post]: http://devstyle.pl/2013/02/04/nie-bedziesz-bral-myszki-wroga-swego-nadaremno/
[vista-switcher]: https://www.ntwind.com/software/alttabter.html
[launchy]: http://www.launchy.net/
[ahk]: https://www.autohotkey.com/