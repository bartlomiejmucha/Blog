---
layout: post
title: "How Powershell slowed down the script to copy MediaCache from prod to stage slot"
description: "Here is the updated version of the script that copies MediaCache from prod to stage slot during deployment with Azure DevOps. It uses WebClient instead of Invoke-RestMethod."
date: "2019-10-28 +0100"
tags: [Sitecore, Azure, Azure Pipelines, PowerShell]
image:
---
**!!! Here is the link to the original article: [link]({{ site.baseurl }}{% post_url 2019-01-23-how-to-copy-mediacache-from-prod-to-stage-slot-during-deployment-with-azure-devops %}) !!!.**

The original script works fine, however, when the MediaCache folder grows, it takes longer and longer to execute and at some point it starts throwing timeouts every time. To me, it was weird because downloading MediaCache locally worked very fast every time, (it was 30 minutes vs 1 minute locally). 

My colleague wrote an article [How to expand archive with powershell](https://blog.senktas.net/2019/07/12/how-to-expand-archive-with-powershell/) where he compared performance between `[io.compression.zipfile]::ExtractToDirectory` and Powershell's `Expand-Archive` command. The conclusion was that `[io.compression.zipfile]::ExtractToDirectory` is much faster than the *Powershell* command.

I thought that maybe there is a similar situation with my script and it turned out that it is. Instead of using `Invoke-RestMethod` I decided to use `WebClient` and the results are amazing. The script finally works fast. 

So I changed this:

``` powershell
Invoke-RestMethod -Uri $kuduApiUrl `
                  -Headers @{"Authorization"=$kuduApiAuthorisationToken;"If-Match"="*"} `
                  -Method GET `
                  -OutFile $localPath `
                  -ContentType "multipart/form-data"
```

To this:

``` powershell
$webClient = New-Object System.Net.WebClient
$webClient.Headers.Add("Authorization", $kuduApiAuthorisationToken)
$webClient.Headers.Add("If-Match", "*")
$webClient.Headers.Add("Content-Type","multipart/form-data");

$webClient.DownloadFile($kuduApiUrl, $localPath)
```

And here is the full script:

``` powershell
function Get-AzureRmWebAppPublishingCredentials($resourceGroupName, $webAppName, $slotName = $null){
    if ([string]::IsNullOrWhiteSpace($slotName)){
        $resourceType = "Microsoft.Web/sites/config"
        $resourceName = "$webAppName/publishingcredentials"
    }
    else{
        $resourceType = "Microsoft.Web/sites/slots/config"
        $resourceName = "$webAppName/$slotName/publishingcredentials"
    }
    $publishingCredentials = Invoke-AzureRmResourceAction -ResourceGroupName $resourceGroupName -ResourceType $resourceType -ResourceName $resourceName -Action list -ApiVersion 2015-08-01 -Force
    return $publishingCredentials
}

function Get-KuduApiAuthorisationHeaderValue($resourceGroupName, $webAppName, $slotName = $null){
    $publishingCredentials = Get-AzureRmWebAppPublishingCredentials $resourceGroupName $webAppName $slotName
    return ("Basic {0}" -f [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(("{0}:{1}" -f $publishingCredentials.Properties.PublishingUserName, $publishingCredentials.Properties.PublishingPassword))))
}

function Get-ZippedMediaCacheFolderKuduApiUrl($webAppName, $slotName){
    if ($slotName -eq $null)
    {
        return "https://$webAppName.scm.azurewebsites.net/api/zip/site/wwwroot/App_Data/MediaCache/"
    }

    return "https://$webAppName-$slotName.scm.azurewebsites.net/api/zip/site/wwwroot/App_Data/MediaCache/"
}

function Download-ZippedMediaCacheFolderFromWebApp($resourceGroupName, $webAppName, $localPath, $slotName = $null){

    $kuduApiAuthorisationToken = Get-KuduApiAuthorisationHeaderValue $resourceGroupName $webAppName $slotName
    $kuduApiUrl = Get-ZippedMediaCacheFolderKuduApiUrl $webAppName $slotName
    Write-Host " Downloading MediaCache Folder. Source: '$kuduApiUrl'. Target: '$localPath'..." -ForegroundColor DarkGray

    $webClient = New-Object System.Net.WebClient
    $webClient.Headers.Add("Authorization", $kuduApiAuthorisationToken)
    $webClient.Headers.Add("If-Match", "*")
    $webClient.Headers.Add("Content-Type","multipart/form-data");

    $webClient.DownloadFile($kuduApiUrl, $localPath)
}

$resourceGroupName = "$(ResourceGroupName)"
$webAppName = "$(WebAppName_CD)"
$localPath  = "$(Agent.ReleaseDirectory)\MediaCache.zip"

Download-ZippedMediaCacheFolderFromWebApp $resourceGroupName $webAppName $localPath
```