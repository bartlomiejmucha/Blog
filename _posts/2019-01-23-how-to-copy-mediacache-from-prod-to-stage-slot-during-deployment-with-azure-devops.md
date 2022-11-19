---
layout: post
title: "How to copy MediaCache from prod to stage slot during deployment with Azure DevOps"
description: "Generating media cache every time you do deployment is a waste of CPU. In this article I show you how to copy MediaCache folder from prod slot to stage slot during deployment."
date: "2019-01-23 +0100"
tags: [Sitecore, PowerShell, Azure, DevOps, Media Library]
image: /assets/images/posts/024/sitecore-power-shell-small.png
---
**!!! Read the [updated version of this article]({{ site.baseurl }}{% post_url 2019-10-28-how-powershell-slowed-down-the-script-to-copy-mediacache-from-prod-to-stage-slot %}) !!!.**

When you do a clean deployment to a stage slot on AppService, the MediaCache folder is empty and it will be regenerated. It can lead to poor performance and worse user experience after deployment because every image has to be downloaded from the database, scaled to the requested size and then stored in the MediaCache folder. If you additionally do the image compression with Dianoga, and your AppService plan is rather slow (like S2), compression (especially the lossy one like jpegoptim) can use 100% of your CPU for a long time (like minutes), it can lead to Redis timeouts and even AppService restart (by auto heal). In short words, generating media cache every time you do deployment is just a waste of CPU.

### What if we could download MediaCache folder from prod slot and upload it to the staging slot during deployment?

It's quite easy. We can use Kudu API for this. On [this blog](https://blog.kloud.com.au/2016/08/30/interacting-with-azure-web-apps-virtual-file-system-using-powershell-and-the-kudu-api/) the author shared PowerShell scripts that we can use to authenticate and work with Kudu. Read it first. I copied the required code from that blog and created the script below:

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

    Invoke-RestMethod -Uri $kuduApiUrl `
                        -Headers @{"Authorization"=$kuduApiAuthorisationToken;"If-Match"="*"} `
                        -Method GET `
                        -OutFile $localPath `
                        -ContentType "multipart/form-data"
}

$resourceGroupName = "$(ResourceGroupName)"
$webAppName = "$(WebAppName_CD)"
$localPath  = "$(Agent.ReleaseDirectory)\\MediaCache.zip"

Download-ZippedMediaCacheFolderFromWebApp $resourceGroupName $webAppName $localPath
```

The script requires three parameters: `$resourceGroupName`, `$webAppName` and `$localPath`. You can put them directly into the script or use [Pipeline Variables](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=vsts&tabs=yaml%2Cbatch). To execute the script, use [Azure PowerShell task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-powershell?view=azure-devops&viewFallbackFrom=vsts). At the end you will get MediaCache.zip file. Next step is to extract it to your deployment package (under App_Data) and then deploy it together.