---
layout: post
series: msbuild
title: "How to extend MSBuild publish pipeline to copy content files from all Helix modules to the output"
description: "It's possible to drastically speed up the publish process by extending MSBuild to publish all projects in a single pass instead of using gulp."
date: "2018-10-18 +0100"
tags: [Sitecore, Helix, MSBuild]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
In Habitat's gulp script, all projects are published one by one independently. For each project, a separate instance of MSBuild is started. It has to read, interpret and execute all core .targets every time. Because of this, the whole publishing process may take a long time.

It's possible to drastically speed up this process by extending MSBuild to publish all projects in a single pass.

### WebRoot project

We need to pick one project that we want to extend with our MSBuild customisations. This can be a project from your Project layer. However, I think that the best option is to create a new project outside of your layer folders and name it **WebRoot**. It's only responsibility is to build and publish the solution.

So, let's create a new Web Application project under src folder. By doing this you should get four folders under src folder: Foundation, Feature, Project and WebRoot. Then in the WebRoot directory, create a **Helix.targets** file, open WebRoot.csproj file and add the following Import:

``` xml
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  ..
 <Import Project="$(MSBuildBinPath)\Microsoft.CSharp.targets" />
 <Import Project="$(VSToolsPath)\WebApplications\Microsoft.WebApplication.targets" Condition="'$(VSToolsPath)' != ''" />
 <Import Project="$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v10.0\WebApplications\Microsoft.WebApplication.targets" Condition="false" />
 <Import Project="Helix.targets" />
  ..
</Project>
```

**Helix.targets** is the file where we are going to add our customisations.

### Collect modules to publish

First, we need to collect a list of modules that we want to publish. We need paths to .csproj files. Gulp script searches for .csproj in solution directory using a query like this: "/**/code/*.csproj". We can use a similar technique to get that list:

``` xml
<ItemGroup>
  <HelixProjectsPaths Include="$(SolutionDir)\**\code\*.csproj"></HelixProjectsPaths>
</ItemGroup>
```

`$(SolutionDir)` is the path to the directory that holds your .sln file.

Alternatively, we can add modules as references to our WebRoot project. When we do this, all project references will be added as `ProjectReference` elements inside **WebRoot.csproj**:

``` xml
<ItemGroup>
  <ProjectReference Include="..\feature\Accounts\code\Sitecore.Feature.Accounts.csproj">
    <Project>{3f61c57d-50d9-4194-8196-ef5f7aa4cce5}</Project>
    <Name>Sitecore.Feature.Accounts</Name>
  </ProjectReference>
  ...
</ItemGroup>
```

We can then use `ProjectReference` item directly or populate `HelixProjectPaths` like this:

``` xml
<ItemGroup>
  <HelixProjectsPaths Include="%(ProjectReference.FullPath)"></HelixProjectsPaths>
</ItemGroup>
```

The advantage of this approach is that we can easily include or exclude projects and publish only the modules we want and also that MSBuild will build referenced projects if necessary and publish compiled DLLs.

### Publish content files from modules

In **Microsoft.Web.Publishing.targets** there is a target named `PipelineCollectFilesPhase`. It collects files to publish. Its declaration looks like this:

``` xml
<Target Name="PipelineCollectFilesPhase" DependsOnTargets="$(PipelineCollectFilesPhaseDependsOn)">
```

It depends on targets defined in `PipelineCollectFilesPhaseDependsOn` property. Let's add our own target to that list:

``` xml
<PropertyGroup>
  <PipelineCollectFilesPhaseDependsOn>
    $(PipelineCollectFilesPhaseDependsOn);
    CollectContentFilesFromHelixModules;
  </PipelineCollectFilesPhaseDependsOn>
</PropertyGroup>
```

Now, our custom target `CollectContentFilesFromHelixModules` will be executed as a last one in the list but before the `PipelineCollectFilesPhase` target.

In **Microsoft.Common.CurrentVersion.targets** there is `ContentFilesProjectOutputGroup` target. It returns a list of all files that has Build Action set to Content. We want to get exactly that list for each module to include these files in another list `FilesForPackagingFromProject` which stores all files that are going to be published.

To execute that target for each module and collect results, we can use code below:

``` xml
<MSBuild Projects="@(ProjectReference)" Targets="ContentFilesProjectOutputGroup" BuildInParallel="$(BuildInParallel)">
  <Output TaskParameter="TargetOutputs" ItemName="_ContentFilesFromHelixModules" />
</MSBuild>
```

You remember, that inside `ProjectReference` item we have a list of all referenced projects in our WebRoot project. We pass that list to Projects attribute. Then into Targets attribute we pass the name of the target we want to execute. Additionally, we set `BuildInParallel` attribute to the defaults defined in VisualStudio. You probably noticed the Output element as well. It says that we want to a store list returned by the MSBuild task into a new item named `_ContentFilesFromHelixModules`. The results from all projects will be merged to that item.

Now, as the last step, we need to add files from our `_ContentFilesFromHelixModules` item to the list of files for publishing. The name of that list is `FilesForPackagingFromProject`. We can do that with the following code:

``` xml
<ItemGroup>
  <ContentFilesFromHelixModules Include="@(_ContentFilesFromHelixModules)">
    <DestinationRelativePath>%(TargetPath)</DestinationRelativePath>
  </ContentFilesFromHelixModules>
  <FilesForPackagingFromProject Include="@(ContentFilesFromHelixModules)" /> 
</ItemGroup>
```

The `FilesForPackagingFromProject` requires additional metadata for each file: `DestinationRelativePath`. That's why we first extend our list with that metadata. The `%(TargetPath)` is just the relative path of the file inside the project, for example **/App_Config/Include/Foundation/Sitecore.Foundation.Account.config**. As the final step, we include our list into `FilesForPackagingFromProject` item.

This is how the Helix.targets file can look at the end:

``` xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <PipelineCollectFilesPhaseDependsOn>
      $(PipelineCollectFilesPhaseDependsOn);
      CollectContentFilesFromHelixModules;
    </PipelineCollectFilesPhaseDependsOn>
  </PropertyGroup>
  <Target Name="CollectContentFilesFromHelixModules">
    <MSBuild Projects="@(ProjectReference)" Targets="ContentFilesProjectOutputGroup" BuildInParallel="$(BuildInParallel)">
      <Output TaskParameter="TargetOutputs" ItemName="_ContentFilesFromHelixModules" />
    </MSBuild>
    <ItemGroup>
      <ContentFilesFromHelixModules Include="@(_ContentFilesFromHelixModules)">
        <DestinationRelativePath>%(TargetPath)</DestinationRelativePath>
      </ContentFilesFromHelixModules>
      <FilesForPackagingFromProject Include="@(ContentFilesFromHelixModules)" /> 
    </ItemGroup>
  </Target>
</Project>
```

### How to test it?

Create a **WebRoot** project and **Helix.targets** file, import Helix.targets inside WebRoot.csproj. Don't forget to add modules as project references in WebRoot project, set Build Actionfor Web.config to None, create (or copy) publishing profile and try to do a publish to a new location on your disk. You should see all content files from your all referenced modules as well as all required dlls in the bin folder.

There is one more thing. Convert your WebRoot project to use PackageReference instead of packages.config. If you don't do that indirect references from referenced projects will not be published, but this is a topic for another time.

I committed all above changes into [this commit](https://github.com/bartlomiejmucha/Habitat/commit/9f26f20f17fd2074ce0e76a5a73a8c5c8a2f1325) in my Habitat fork.

### I want more

If you like to know how to extend MSBuild to apply transform files, check out my [next article]({{ site.baseurl }}{% post_url 2018-10-24-how-to-extend-msbuild-publish-pipeline-to-apply-transform-files %}) in this series.