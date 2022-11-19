---
layout: post
series: msbuild
title: "MSBuild extension points"
description: "MSBuild offers a few ways you can extend build and publish process with your custom code. It's not a complete list but just a summary of the most useful techniques."
date: "2018-10-12 +0100"
tags: [Sitecore, Helix, MSBuild]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
MSBuild offers a few ways you can extend build and publish process with your custom code. It's not a complete list but just a summary of the most useful techniques.

### Import

This is the most common technique. You probably saw it many times in your .csproj file. This is how the new Class Library project looks like:

``` xml
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" Condition="Exists('$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props')" />
  ...
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```

The project imports **Microsoft.Common.props** and **Microsoft.CSharp.targes**. There is a chain of nested imports, but at the end, this is where the `Clean`, `Build` and `Rebuild` targets are defined. You can add your own imports into .csproj file to load your custom code.

### Import from Nuget package

This is the same as above. You can create a NuGet package that contains .targets file, build it with a `-Tool` parameter, and when you install that package in your project, the `<Import>` will be added automatically to your project file. If you want to see the real example, take a look at the source code of my [Unicorn.MSBuild](https://github.com/bartlomiejmucha/Unicorn.MSBuild) package. If you install it in your project, you will automatically get import like this at the end of the project file:

``` xml
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  ..
  <Import Project="..\packages\Unicorn.MSBuild.1.0.0\build\Unicorn.MSBuild.targets" Condition="Exists('..\packages\Unicorn.MSBuild.1.0.0\build\Unicorn.MSBuild.targets')" />
</Project>
```

You can read how I implemented `Unicorn.MSBuild` package in this article in [this series]({% post_url 2018-10-25-how-to-extend-msbuild-to-execute-unicorn-sync-action %}).

### Directory.Build.props and Directory.Build.targets

The **Directory.Build.props** and **Directory.Build.targest** are user-defined files that provide customisation to projects under a directory. If you put that files in your project location, the files will be loaded automatically. If the file is not there, MSBuild would search the directory structure upward until it locates the files.

This is very useful if you want to set some properties or add some targets for your all projects, or projects under foundation folder for example.

### .wpp.targets file

If you create a new Web Application project at the end of the file you will get additional imports:

``` xml
<Import Project="$(VSToolsPath)\WebApplications\Microsoft.WebApplication.targets" Condition="'$(VSToolsPath)' != ''" />
<Import Project="$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v10.0\WebApplications\Microsoft.WebApplication.targets" Condition="false" />
```

One of these two lines will be executed, and it will import the **Microsoft.Web.Publishing.targets** file. This file then will try to import the `<projectname>.wpp.targets` file from the folder that holds your .csproj. The file settings and extensions defined there will be applied automatically to your all publishing profiles.

### I want more
The best way to learn more is to read the official documentation. Here are a few interesting pages:

- [How to extend VS build process](https://learn.microsoft.com/en-us/visualstudio/msbuild/how-to-extend-the-visual-studio-build-process?view=vs-2022)
- [MSBuild .targets files](https://learn.microsoft.com/pl-pl/visualstudio/msbuild/msbuild-dot-targets-files?view=vs-2022)
[Customize your build](https://learn.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2022)

Also, read my [next article]({{ site.baseurl }}{% post_url 2018-10-13-msbuild-build-and-publish-pipelines %}) in this series where I described two ways to extend Build and Publish pipeline.