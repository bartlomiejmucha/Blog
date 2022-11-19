---
layout: post
series: msbuild
title: "MSBuild Build and Publish pipelines"
description: "The Microsoft.Common.targets file defines two main pipelines: Build and Publish. Here is a short summary of both."
date: "2018-10-13 +0100"
tags: [Sitecore, Helix, MSBuild]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
### The Build pipeline

The newly created .csproj file contains the following import near the end of the file:

``` xml
<Import Project="$(MSBuildBinPath)\Microsoft.CSharp.targets" />
```

This file then imports **Microsoft.CSharp.CurrentVersion.targets** and it then imports **Microsoft.Common.targets** and it then imports **Microsoft.Common.CurrentVersion.targets**. On my local PC all files are located here: C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\MSBuild\15.0\Bin. If you use a different version of VisualStudio, BuildTools or project type than me, it can load different files.

The **Microsoft.Common.CurrentVersion.targets** contains a definition of all targets and properties related to the Build and Publish process. The most critical target is the `Build` target. Its definition looks like this:

``` xml
<Target Name="Build" DependsOnTargets="$(BuildDependsOn)">
```

The `DependsOnTargets` attribute says that `Build` target depends on targets defined by `BuildDependsOn` property. The `BuildDependsOn` looks like this:

``` xml
<PropertyGroup>
  <BuildDependsOn>
      BeforeBuild;
      CoreBuild;
      AfterBuild
  </BuildDependsOn>
</PropertyGroup>
```

So it's a list of three targets: `BeforeBuild`, `CoreBuild` and `AfterBuild`. The `BeforeBuild` and `AfterBuild` are empty. They are here for us, so we can redefine them (overwrite) in our project files. In other words, we can create a new .targets file and import it to our .csproj file, then inside this new .targets file we can override these targets like this:

``` xml
<Project>
  <Target Name="BeforeBuild">
    <Message Text="Message before build." />
  </Target>
  <Target Name="AfterBuild">
    <Message Text="Message after build." />
  </Target>
</Project>
```

Alternatively, we can update the value of the `BuildDependsOn` property in this way:

``` xml
<PropertyGroup>
  <BuildDependsOn>
      OurCustomTargetBefore;
      $(BuildDependsOn);
      OurCustomTargetAfter
  </BuildDependsOn>
</PropertyGroup>
```

The `BuildDependsOn` property now contains a list of 5 targets where the first one and the last one are our custom targets, and the other three are defined in original value of the `BuildDependsOn` property. So this is a way to extend a list of targets.

The `CoreBuild` target is defined like this:

``` xml
<PropertyGroup>
  <CoreBuildDependsOn>
      BuildOnlySettings;
      PrepareForBuild;
      PreBuildEvent;
      ResolveReferences;
      PrepareResources;
      ResolveKeySource;
      Compile;
      ExportWindowsMDFile;
      UnmanagedUnregistration;
      GenerateSerializationAssemblies;
      CreateSatelliteAssemblies;
      GenerateManifests;
      GetTargetPath;
      PrepareForRun;
      UnmanagedRegistration;
      IncrementalClean;
      PostBuildEvent
  </CoreBuildDependsOn>
</PropertyGroup>
<Target Name="CoreBuild" DependsOnTargets="$(CoreBuildDependsOn)">
```

As you can see it's a long list of targets. There are some preparation steps first, then the references are resolved, and then the code is compiled and so on.

Describing those targets it's outside the scope of this article. It's just a starter. If you want to extend the Build process, you have to dig through it and understand how it works.

Similar for `Rebuild` target:

``` xml
<PropertyGroup>
  <RebuildDependsOn>
      BeforeRebuild;
      Clean;
      Build;
      AfterRebuild;
  </RebuildDependsOn>
</PropertyGroup>
<Target Name="Rebuild" DependsOnTargets="$(RebuildDependsOn)">
```

The `BeforeRebuild` and `AfterRebuild` are empty, and we can override them.

And again for Clean target:

``` xml
<PropertyGroup>
  <CleanDependsOn>
      BeforeClean;
      UnmanagedUnregistration;
      CoreClean;
      CleanReferencedProjects;
      CleanPublishFolder;
      AfterClean
  </CleanDependsOn>
</PropertyGroup>
<Target Name="Clean" DependsOnTargets="$(CleanDependsOn)">
```

The `BeforeClean` and `AfterClean` are empty, and we can override them.

### The Publish pipeline

The **Microsoft.Common.CurrentVersion.targets** also contains `Publish` and `PublishOnly` (used by OneClick toolbar) targets. Here are the definitions:

``` xml
<PropertyGroup>
  <PublishDependsOn>
      SetGenerateManifests;
      Build;
      PublishOnly
  </PublishDependsOn>
</PropertyGroup>
<Target Name="Publish" DependsOnTargets="$(PublishDependsOn)">
<PropertyGroup>
  <PublishOnlyDependsOn>
      SetGenerateManifests;
      PublishBuild;
      BeforePublish;
      GenerateManifests;
      CopyFilesToOutputDirectory;
      _CopyFilesToPublishFolder;
      _DeploymentGenerateBootstrapper;
      ResolveKeySource;
      _DeploymentSignClickOnceDeployment;
      AfterPublish
  </PublishOnlyDependsOn>
</PropertyGroup>
<Target Name="PublishOnly" DependsOnTargets="$(PublishOnlyDependsOn)">
```

The `Publish` target is executed when you run a Publish in VisualStudio. It first generates manifests, then does a build and then executes `PublishOnly` target. When you click Publish button on your OneClick Publish toolbar in VisualStudio, the `PublishOnly` target is executed.

Again, the `BeforePublish` and `AfterPublish` targets are empty, and we can override them.

In a web application project, an additional import is added to the project file:

``` xml
<Import Project="$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v10.0\WebApplications\Microsoft.WebApplication.targets" Condition="false" />
```

The **Microsoft.WebApplication.targets** then will import **Microsoft.Web.Publishing.targets**. And this is where some customisation to default Publish pipeline, that is required for web applications, is defined.

### I want more

To learn more, read my [next article]({{ site.baseurl }}{% post_url 2018-10-18-how-to-extend-msbuild-publish-pipeline-to-copy-content-files-from-all-helix-modules-to-the-output %}) in the series, where I describe a real example of extending Publish pipeline.