---
layout: post
series: msbuild
title: "How to extend MSBuild publish pipeline to apply transform files"
description: "MSBuild can apply our transform files. It actually does this for Web.config out of the box. We can use it to apply transform files to other config files in Helix solution."
date: "2018-10-24 +0100"
tags: [Sitecore, Helix, MSBuild]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
MSBuild can apply our transform files. It actually does this for Web.config out of the box. In Helix projects there are a few additional scenarios to support:

- File.config and transform file for that config are located in a single project. No other transform files in other projects.
- File.config located in one of the projects in the solution and there are one or more transform files for that config file in solution.
- File.config not in the solution, but there are transform files that we want to apply on File.config located in publish directory,
    - We publish directly to the publish directory and we want to apply transform files,
    - We generate a deployment package and we want to include transform files in the package, so we can apply during deployment.

By File.config I mean any XML or JSON file that we want to transform.

### Scenario #1

This works out of the box for Web.config. MSBuild automatically applies Web.Debug.config or Web.Release.config. For other files, we can use [SlowCheetah](https://github.com/Microsoft/slow-cheetah).

SlowCheetah is an MSBuild extension. It is a bit similar to the extension described in this article. It also includes Visual Studio plugin for previewing transform files.

### Scenario #2 and Scenario #3A

Scenario #2 and #3A are similar. We have File.config that we want to transform. The only difference is the location of the file. It's either located in our solution or in the publish directory. This, unfortunately, does not work out of the box and we need to write an extension for MSBuild to support it.

I decided that the best option is to extend [SlowCheetah](https://github.com/Microsoft/slow-cheetah) to support this scenario. The advantage of the SlowCheetah is that it's maintained by Microsoft, it supports different versions of Visual Studio and MSBuild and it supports transformations of XML and JSON. Additionally, it's easier to extend SlowCheetah than write all from the scratch.

### Scenario #3B

This scenario is useful when you want to build a deployment package and then use that package to deploy your application on all your environments. It's usually done in an automated way in some CI\CD system like VSTS. I will get back to this scenario in the follow-up article about VSTS.

For now, let's focus on scenario #2 and #3A. This is what we need for local development.

### Implementation of #2 and #3A

First, we have to collect all transform files from helix modules. We can search for all files with **.xdt** or **.jdt** extension, however, I think the better idea, is to mark transform files with our custom metadata inside ***.csproj**. Let's name our custom metadata as `ApplyTransformOnPublish` and set it to true like this:

``` xml
<None Include="App_Config\Security\Domains.config.xdt">
  <ApplyTransformOnPublish>true</ApplyTransformOnPublish>
</None>
```

In the future we can write Visual Studio plugin to do this or even better, we can extend SlowCheetah plugin, but for now, we have to open each *.csrpoj that contain transform file and modify it manually. 

Next, we have to write a new target that will return files with `ApplyTransformOnPublish` set to `true`. The code can look like this:

``` xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <!--Default values -->
  <ItemDefinitionGroup>
    <None>
      <ApplyTransformOnPublish>false</ApplyTransformOnPublish>
    </None>
  </ItemDefinitionGroup>

  <Target
    Name="GetTransformFilesToApplyOnPublish"
    DependsOnTargets="PrepareForBuild;AssignTargetPaths"
    Returns="@(_TransformFilesToApplyOnPublish)">
    <ItemGroup>
      <_TransformFilesToApplyOnPublish Include="@(_NoneWithTargetPath->'%(FullPath)')" Condition="'%(ApplyTransformOnPublish)' == 'true'"/>
    </ItemGroup>
  </Target>
</Project>
```

In the `ItemDefinitionGroup` we set default value for all `<None>` items. I assumed that for transform files, the Build Action should be always set to `None`, because we don't want to treat them as content files and deploy, but this can be easily changed if needed.

The `GetTransformFilesToApplyOnPublish` just gets all items from `_NoneWithTargetPath` list and include them into `_TransformFilesToApplyOnPublish` list if the `ApplyTransformOnPublish` is set to `true`. Then `_TransformFilesToApplyOnPublish` is returned by target. The target depends on `PrepareForBuild` and `AssignTargetPaths` because we need the `_NonewithTargetPath` to be populated before we use it.

The above code will extend our module projects, not the WebRoot project because we want to evaluate `_NoneWithTargetPaths` list that is collected for a module and not for a WebRoot. So, let's create a new **Helix.Module.targets** file and put above code into it. 

In the **Helix.targets** file, we have to extend `PipelineCollectFilesPhaseDependsOn`. We did the same in the [previous article]({% post_url 2018-10-18-how-to-extend-msbuild-publish-pipeline-to-copy-content-files-from-all-helix-modules-to-the-output %})  to collect content files from helix modules.

``` xml
<PropertyGroup>
  <PipelineCollectFilesPhaseDependsOn>
    $(PipelineCollectFilesPhaseDependsOn);
    CollectTransformFilesToApplyOnPublish;
  </PipelineCollectFilesPhaseDependsOn>
</PropertyGroup>
<Target Name="CollectTransformFilesToApplyOnPublish">
...
</Target>
```

Inside `CollectTransformFilesToApplyOnPublish` add following code:

``` xml
<MSBuild Projects="@(ProjectReference)" Targets="GetTransformFilesToApplyOnPublish" BuildInParallel="$(BuildInParallel)" 
         Properties="CustomBeforeMicrosoftCSharpTargets=$(MSBuildThisFileDirectory)Helix.Module.targets">
  <Output TaskParameter="TargetOutputs" ItemName="TransformFilesToApplyOnPublish" />
</MSBuild>
```

This code executes `GetTransformFilesToApplyOnPublish` target, that we created earlier, against all referenced projects. It aggregates results from all projects into a single item `TransformFilesToApplyOnPublish`

You probably noticed `Properties` attribute on the MSBuild task. Inside we set `CustomBeforeMicrosoftCSharpTargets` property. It collects paths to all additional files that we want to import into a project before it's executed. Inside, we provided the path to **Helix.Module.targets** file we created earlier. This way, MSBuild is able to execute our custom target from inside module's project.

Now, we have a list of transform files, so now we can update that list with some additional metadata we will use later:

``` xml
<ItemGroup>
  <TransformFilesToApplyOnPublish>
    <TransformFile>%(FullPath)</TransformFile>
    <TargetPathToFileToTransform>$([System.IO.Path]::Combine($([System.IO.Path]::GetDirectoryName(%(TargetPath))),%(Filename)))</TargetPathToFileToTransform>
  </TransformFilesToApplyOnPublish>

  <TransformFilesToApplyOnPublish>
    <DestinationFileInPackageDir>$(_PackageTempDir)\%(TargetPathToFileToTransform)</DestinationFileInPackageDir>
    <DestinationFileInPublishDir>$(PublishUrl)\%(TargetPathToFileToTransform)</DestinationFileInPublishDir>
  </TransformFilesToApplyOnPublish>
</ItemGroup>
```

As you can see the list is transformed twice. In the first pass, two metadata are added, and in the second pass, one of the metadata is used to add additional two. The `TransformFile` is the full path to the transform file. The `TargetPathToFileToTransform` is a relative path to the file that is going to be transformed by the transform file and it can look like this **AppConfig\Security\Domains.config**. Next, the `DestinationFileInPackageDir` and `DestinationFileInPublishDir` are the paths to the file that is going to be transformed, but the first path is inside temp directory and the second one is inside the published directory.

The final part of the `CollectTransformFilesToApplyOnPublish` is to calculate files that should be copied from a publish directory to a package directory. This is scenario #3A. We have transform files, but we don't have a file we want to transform. The file is present in a publish directory, so we can just copy it to a package directory and apply transforms before we publish it again.

``` xml
<ItemGroup>
  <FilesToCopyFromPublishDirToPackageDir Include="@(TransformFilesToApplyOnPublish -> '%(DestinationFileInPublishDir)')" Condition="Exists('%(DestinationFileInPublishDir)')">
    <DestinationRelativePath>%(TargetPathToFileToTransform)</DestinationRelativePath>
  </FilesToCopyFromPublishDirToPackageDir>
  <FilesToCopyFromPublishDirToPackageDir Remove="@(FilesToCopyFromPublishDirToPackageDir)" Condition="@(FilesToCopyFromPublishDirToPackageDir) != '' and @(FilesForPackagingFromProject) != '' and %(DestinationRelativePath) != ''"/>

  <FilesForPackagingFromProject Include="@(FilesToCopyFromPublishDirToPackageDir)" Condition=""/>
</ItemGroup>
```

The code first includes all files that exist in publish directory but then removes a file from that list, if there is already a file in `FilesForPackaginFromProject` with the same `DestinationRelativePath`. If there is, that means we have that file in solution and it should not be copied form publish directory. At the end, the final list is included in FilesForPackagingFromProject. It is the same what we did in a [previous article]({% post_url 2018-10-18-how-to-extend-msbuild-publish-pipeline-to-copy-content-files-from-all-helix-modules-to-the-output %}) where we added all content files form helix modules to the `FilesForPackagingFromProject` list. Those files are then copied automatically to the package temporary directory and then published.

Now, we have a list of transform files and all files we want to transform are copied to package temp directory. The last part is to actually apply transformations. This is done by the following target:

``` xml
<Target Name="ApplyTransformsOnPublish" AfterTargets="ScApplyWebTransforms" DependsOnTargets="CollectTransformFilesToApplyOnPublish">
  <SlowCheetah.TransformTask Source="%(TransformFilesToApplyOnPublish.DestinationFileInPackageDir)"
    Transform="%(TransformFile)"
    Destination="%(DestinationFileInPackageDir)"
    Condition="Exists('%(TransformFile)') and Exists('%(DestinationFileInPackageDir)')" />
</Target>
```

As you can see, the target is set to run after `ScApplyWebTransforms` target. So we actually extending SlowCheetah here. Our target also depends on `CollectTransformFilesToApplyOnPublish` because we need a list of transform files first. Inside target, we execute only one task: `SlowCheetah.TransformTask`. The task is defined in SlowCheetah nuget package, so you have to install it into the WebRoot project. The `SlowCheetah.TransformTask` as a source and destination gets paths to the files that are in package temp directory. After that, the package is published to the publish directory.

### How to test it?

In your own copy of Habitat, set `ApplyTransformOnPublish` metadata, install SlowCheetah in your WebRoot project, create Helix.Module.targets file and upate Helix.targets and then do the publish.

I committed all above changes into [this commit](https://github.com/bartlomiejmucha/Habitat/commit/a07d022937de29b15fcfe74f7d08ca4e393ce629) in my Habitat fork.

### I want more

If you like to know how to extend msbuild to execute unicorn sync action, check out my [next article]({{ site.baseurl }}{% post_url 2018-10-25-how-to-extend-msbuild-to-execute-unicorn-sync-action %}) in this series.