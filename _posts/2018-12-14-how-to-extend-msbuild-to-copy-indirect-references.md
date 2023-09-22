---
layout: post
series: msbuild
title: "How to extend MSBuild to copy indirect references"
description: "In this article I will show simple msbuild target that allows to copy indirect references."
date: "2018-12-14 +0100"
tags: [Sitecore, Helix, MSBuild, Habitat, Gulp]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
The compiler is smart enough to detect if a referenced assembly is used or not. So if in your root project, you add a Project Reference to other project and in that other project you have a NuGet package that is not used in the code, the compiler will not copy it to the output of the root project.

Usually it is a good thing, however, when developing a Sitecore website using Helix principles, it can be problematic. For example, you have **Foundation.Serialization** project with Unicorn package and you added it as a project reference in your **WebRoot** project. If you do a publish of WebRoot, the Unicorn.dll will not be published, because it's not used anywhere in the code. It's only mentioned in the config patch, but the compiler does not understand that.

This only happens with **packages.config**. If you use `PackagesReferences` it works fine without any fix. Fortunately, the fix is quite easy to implement and here is the code that we can add to the Helix.targets to fix it: 

``` xml
<PropertyGroup>
  <ResolveAssemblyReferencesDependsOn>
    $(ResolveAssemblyReferencesDependsOn);
    CollectReferencesFromHelixModules;
  </ResolveAssemblyReferencesDependsOn>
</PropertyGroup>

<Target Name="CollectReferencesFromHelixModules">
  <MSBuild Projects="@(ProjectReference)" Targets="ResolveAssemblyReferences" BuildInParallel="$(BuildInParallel)">
    <Output TaskParameter="TargetOutputs" ItemName="ReferencesFromHelixModules" />
  </MSBuild>
  <ItemGroup>
    <Reference Include="%(ReferencesFromHelixModules.FusionName)" Condition="'%(ReferencesFromHelixModules.CopyLocal)'=='true'">
      <ResolvedFrom>%(ReferencesFromHelixModules.ResolvedFrom)%(ReferencesFromHelixModules.FullPath)</HintPath>
      <Redist>%(ReferencesFromHelixModules.Redist)</Redist>
      <CopyLocal>true</CopyLocal>
    </Reference>
  </ItemGroup>
</Target>
```

### What next?

In my [next article]({{ site.baseurl }}{% post_url 2023-09-22-how-to-optimize-build-times-for-mvc-views %}) in the **MSBuild** series I demonstrated how to build mvc views incrementally.