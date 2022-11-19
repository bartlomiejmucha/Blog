---
layout: post
series: msbuild
title: "How to extend MSBuild to execute Unicorn Sync action"
description: "After Build and Publish it's time to sync items with Unicorn. MSBuild can do that as well and extension is quite simple."
date: "2018-10-25 +0100"
tags: [Sitecore, Helix, MSBuild]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
After Build and Publish it's time to sync items with Unicorn. MSBuild can do that as well and extension is quite simple. Gulps script in Habitat repository, under the hood, executes the power shell script to sync items. The powershell contains a .NET code. We can write a custom `Task` that executes the same .NET code and new targets file that tells MSBuild how to use it.

I put that `Task` and targets file into the [Unicorn.MSBuild](https://www.nuget.org/packages/Unicorn.MSBuild/) nuget package. When we install it in our WebRoot project, MSBuild automatically imports targets file from a package (similar like with the SlowCheetah in the [previous article]({% post_url 2018-10-24-how-to-extend-msbuild-publish-pipeline-to-apply-transform-files %})).

I also wrote the [Visual Studio plugin](https://marketplace.visualstudio.com/items?itemName=BartomiejMucha.SyncUnicorn) that adds Sync Unicorn button to the Build menu in Visual Studio. Thanks to this you can execute Sync with just two clicks.

Source code for both: the NuGet package and the VS plugin is on my GitHub repositories: [Unicorn.MSBuild](https://github.com/bartlomiejmucha/Unicorn.MSBuild) and [SyncUnicorn](https://github.com/bartlomiejmucha/SyncUnicorn-for-VisualStudio).

### The implementation

Let's create a new ClassLibrary project and then create a new `SyncUnicorn` class that inherits from `Task` class (or `ToolTask` class if you prefer, which is more advanced):  

``` cs
public class SyncUnicorn : Task
{
   [Required]
   public string ControlPanelUrl { get; set; }

   [Required]
   public string SharedSecret { get; set; }

   public override bool Execute()
   {
       ...
   }
}
```

The class has two properties with the `Required` attribute: `ControlPanelUrl` and `SharedSecret`. Our task needs these two properties in order to execute Sync action. The `Required` attribute makes sure that if we do not provide it, the MSBuild will throw an error. You can see the full source code of `SyncUnicorn` class here.

Build a project to generate an **Unicorn.MSBuild.dll**, then create a new **Unicorn.MSBuild.targets** file and paste the following code:

``` xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <UnicornMSBuildTaskPath>$(MSBuildThisFileDirectory)..\tools\</UnicornMSBuildTaskPath>
  </PropertyGroup>

  <!--Main sync task-->
  <UsingTask TaskName="SyncUnicorn" AssemblyFile="$(UnicornMSBuildTaskPath)Unicorn.MSBuild.dll"/>
  <Target Name="SyncUnicorn">
    <SyncUnicorn ControlPanelUrl="$(UnicornControlPanelUrl)" SharedSecret="$(UnicornSharedSecret)" />
  </Target>
</Project>
```

It's copied from a NuGet package, that's why the `UnicornMSBuildTaskPath` property points to a tools folder, inside the package, that holds the **Unicorn.MSBuild.dll**. 

Next, there is a `UsingTask` statement which tells MSBuild what is the class name of the task and what is the path to the assembly. 

The last thing is the definition of a new target named `SyncUnicorn`. Inside we call our custom task (that has the same name). The values for `ControlPanelUrl` and `SharedSecret` are provided from `UnicornControlPanelUrl` and `UnicornSharedSecret` properties. We can define values for these properties somewhere in our project, for example in publishing profile, but the most convenient place I think is the **WebRoot.wpp.targets** file, because it's loaded for each publishing profiles.

``` xml
<Project>
  <!-- Unicorn Settings -->
  <PropertyGroup>
    <UnicornControlPanelUrl>https://habitat.sc/unicorn.aspx</UnicornControlPanelUrl>
    <UnicornSharedSecret>zUcdjtAKn21fEXIqFnrSzUcdjtAKn21fEXIqFnrSzUcdjtAKn21fEXIqFnrS</UnicornSharedSecret>
  </PropertyGroup>
</Project>
```

### How to use it?

If you installed the Visual Studio plugin, then just right click on the WebRoot project and click Sync Unicorn. You can also execute it from a command line like this:

```
msbuild WebRoot.csproj /t:SyncUnicorn /p:UnicornControlPanelUrl=https://habitat.sc/unicorn.aspx /p:UnicornSharedSecret=zUcdjtAKn21fEXIqFnrSzUcdjtAKn21fEXIqFnrSzUcdjtAKn21fEXIqFnrS
```

### What next?

If you want to see that in action, check out my [next article]({{ site.baseurl }}{% post_url 2018-11-12-video-speed-comparison-of-gulp-vs-msbuild-build-publish %}) in this series.