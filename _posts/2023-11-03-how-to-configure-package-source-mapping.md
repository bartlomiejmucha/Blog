---
layout: post
title: "How to configure package source mapping"
description: "In Sitecore projects, we usually have more than one feed soure for packages and during restore nuget will do a request to each of them to find a package. That means many unnecessary network requests. Fortunatelly, there is a way to map packages to a source."
date: "2023-11-03 +0100"
tags: [Sitecore, Nuget, DevOps]
---
### Multiple feed sources
 
Most sitecore projects is usually having more than one feed soure for packages and during restore nuget will do a request to each of them to find a package. That means many unnecessary network requests. 

The `nuget.config` file might looks like that:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="Nuget" value="https://api.nuget.org/v3/index.json" />
    <add key="Sitecore" value="https://nuget.sitecore.com/resources/v3/index.json" />
  </packageSources>
  <activePackageSource>
    <add key="All" value="(Aggregate source)" />
  </activePackageSource>
</configuration>
```

With that config, for each package that has to be restored, nuget will do two requests (because we have two sources) to find a package.
Our example is very simple, but if you have more sources in the config (maybe multiple private ones), number of unnecessary requests might be huge.

### Adding package source mapping

Fortunatelly nuget allow us to configure [package source mapping](https://learn.microsoft.com/en-us/nuget/consume-packages/package-source-mapping). We can use pattersn, here is an example:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="Nuget" value="https://api.nuget.org/v3/index.json" />
    <add key="Sitecore" value="https://nuget.sitecore.com/resources/v3/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="Nuget">
      <package pattern="*" />
    </packageSource>
    <packageSource key="Sitecore">
      <package pattern="Sitecore.*" />
    </packageSource>
  </packageSourceMapping>
  <activePackageSource>
    <add key="All" value="(Aggregate source)" />
  </activePackageSource>
</configuration>
```

With that setup requests for packages with name that starts with `Sitecore.` will be done to **Sitecore** feed source and all other packages will be requested only from the **Nuget** feed. Additionally in the logs, when doing restore, you will see lines like that: 

> Package source mapping matches found for package ID 'Microsoft.CodeAnalysis.NetAnalyzers' are: 'Nuget'.   
> Package source mapping matches found for package ID 'Sitecore.ContentSearch.SolrProvider' are: 'Sitecore'.