---
layout: post
title: "How to find items with duplicated names using sitecore powershell"
description: "Sitecore 10 by default does not allow having two items with the same path. If you have items like that, here is a script that you can use to find and fix such items."
date: "2023-12-16 +0100"
tags: [Sitecore, Sitecore PowerShell]
---
### Sitecore 10
Sitecore 10 introduced a new setting, that is set to `false` by default:
``` xml
<!--
  ALLOW DUPLICATE ITEM NAMES ON THE SAME LEVEL
  Specifies whether the duplicate item names are allowed on the same level in the content tree.
  Default value: false
    
-->
<setting name="AllowDuplicateItemNamesOnSameLevel" value="false"/>
```

If you migrated from Sitecore 8, or you had that setting set to `true`, you might have items with duplicated item names. If you like to find them and fix the names, here is a powershell script for you:

``` powershell 
$rootItem = Get-Item -Path 'master:/sitecore/content/home'

function CheckDupes($parentItem) {
    $childItems = $parentItem.Axes.GetDescendants()
    
    $childItemsGrouped = $childItems | Initialize-Item | 
        Group-Object { $_.ItemPath} | 
        Where-Object { $_.Count -gt 1 }
    
    foreach($group in $childItemsGrouped)
    {
        'Group Name: ' + $group.Name + ' Count: ' + $group.Count
        
        $index = 1
        foreach ($groupItem in $group.Group) {
            $newName = $groupItem.Name + ' ' + $index++
            
            'New name: ' + $newName
            
            $groupItem.Editing.BeginEdit()
            $groupItem.Name = $newName
            $groupItem.Editing.EndEdit()
        }
    
    }
}

CheckDupes($rootItem)
```

The script groups items with the same path and then for each item in the group, updates the names by adding an index to it, for instance if you have two items with name `Some Item`, the script updates their names to `Some Item 1` and `Some Item 2`.