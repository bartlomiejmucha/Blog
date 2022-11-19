---
layout: post
title: "How to query for unused assets in media library in sitecore using powershell"
description: "If assets in media library in sitecore are stored in the database after some time your db can grow to considerable size. I created the powershell script, which can query for unused resources larger than 1MB."
date: "2014-07-09 +0100"
tags: [Media Library, Sitecore, Sitecore PowerShell]
image: /assets/images/posts/002.003/sitecore-powershell-console-small.png
---
Too big database generates a lot of problems. It is increasingly difficult to backup the database because the backup process, compression an upload to - for example - FTP may take several hours. If you have additional specific requirements like to store backups from the last 7 days, last 6 Fridays, 4 last first Fridays of the month, then you need a lot of storage space.

Below is a PowerShell script that you can use to search media library for unused items larger than 1MB. You can then ask the client to delete them.

``` powershell
cd "media library"

get-childitem -recurse 
    | where-object { $_.Size -ne $null -and [System.Int32]::Parse($_.Size) -ge [System.Int32]::Parse(1048576) -and [Sitecore.Globals]::LinkDatabase.GetReferers($_).Length -eq 0} 
    | sort-object { [System.Int32]::Parse($_.Size) } 
    | format-table {$_.Paths.Path, $_.Size, [Sitecore.Globals]::LinkDatabase.GetReferers($_).Length}
```

This script will only work if you are using PowerShell Extensions (inside Sitecore desktop). If you are using PowerShell from inside Sitecore Rocks, you need to use cmdlets. As far as I know, cmdlet for getting related items will be available in version 2.7.