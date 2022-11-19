---
layout: post
title: "How to populate sitecore with test data using powershell"
description: "If you want to populate sitecore with some test data you can use powershell. I put here sample script and a few informations about its performance."
date: "2014-07-17 +0100"
tags: [Sitecore, Sitecore PowerShell, Test Data]
image: /assets/images/posts/002.003/sitecore-powershell-console-small.png
---
Sometimes you need to generate test data, for instance, to check the performance of a solution. In Sitecore you can do that in different ways: you can create items by hand, which is obviously the worst option, you can write a piece of code that will create items for you, and you can also use the Sitecore PowerShell.

The following script creates 10,000 test items in a loop, sets the name, price and category for each of them. Also at the end, it displays the total run time.

``` powershell
cd master:\\content\home\products
$categories = @{$true="{050E9CAE-F55F-4DF1-8001-07CD4D70FC09}";$false="{8C11E41B-0CE5-4602-BB6D-5DE5B4356D25}"}

$stopwatch = new-object -type 'System.Diagnostics.Stopwatch'
$stopwatch.Start()

for($i = 1; $i -le 10000; $i++)
{
    $newProduct = new-item "product_$i" -type 'sample\product'
    
    set-itemproperty $newProduct.Paths.Path -name "Name" -value "Product $i"
    set-itemproperty $newProduct.Paths.Path -name "Price" -value $i
    set-itemproperty $newProduct.Paths.Path -name "Category" -value $categories[$i % 2 -eq 1]
}

$stopwatch.Stop()
write-host $stopwatch.Elapsed
```

### Performance

On my test virtual machine, for 10,000 of items, the script executes from about an hour to one and a half hour. It is quite a lot of time if you want to insert a million of products. That is why I introduced two optimisations. Here they are:

``` powershell
cd master:\\content\home\products

$categories = @{$true="{050E9CAE-F55F-4DF1-8001-07CD4D70FC09}";$false="{8C11E41B-0CE5-4602-BB6D-5DE5B4356D25}"}

$stopwatch = new-object -type 'System.Diagnostics.Stopwatch'
$stopwatch.Start()

$bulkupdate = new-object -type "Sitecore.Data.BulkUpdateContext"

for($i = 1; $i -le 10000; $i++)
{
    $newProduct = new-item "product_$i" -type 'sample\product'
    
    $newProduct.Editing.BeginEdit();
    $newProduct["Name"] = "Product $i";
    $newProduct["Price"] = $i;
    $newProduct["Category"] = $categories[$i % 2 -eq 1] ;
    $newProduct.Editing.EndEdit() | out-null
}

if($bulkupdate -ne $null)
{
    $bulkupdate.Dispose();
}

$stopwatch.Stop()
write-host $stopwatch.Elapsed
```

The use of the `BulkUpdateContext` forces Sitecore to not update index immediately after every save, but to do it later.

By avoiding the use of cmdlet `Set-Property` for the direct Sitecore API calls, we managed to reduce the number of `BeginEdit()` and `EndEdit()` method calls three times, because `Set-Property` internally calls this methods every time.

Thanks to these optimisations I was able to reduce the time of inserting 10,000 items to about 25 minutes.

If you are using PowerShell that is included in Sitecore Rocks, these optimisations will not work. This PowerShell is not running in an application context, so you need to use cmdlets.