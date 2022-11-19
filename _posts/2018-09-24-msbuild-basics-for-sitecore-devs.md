---
layout: post
series: msbuild
title: "MSBuild basics for Sitecore devs"
description: "MSBuild can be your friend. Especially in Helix projects. We can extend MSBuild to build Helix projects the same way as gulp scripts do, but much, much faster. It's a first article in a msbuild series."
date: "2018-09-24 +0100"
tags: [Sitecore, Helix, MSBuild]
image: /assets/images/posts/msbuild-series-small-logo.png
categories: msbuild
---
**MSBuild** can be your friend. At first, it is a difficult friendship though, because it speaks a different language. However, if you talk to it patiently, it can do a lot of useful things for you. For example, it can build your Helix projects in the same way as gulp scripts do, but much, much faster. But this is a story that I will tell you next time. Today let's start with some basics.

### Target

Consider the following basic example:

``` xml
<Project>
  <Target Name="Hello">
    <Message Text="MSBuild says hello!" />
  </Target>
</Project>
```

Copy and save it as a **Hello.csproj** file, then open **Developer Command Prompt for Visual Studio 2017** from Start menu or go to the MSBuild directory **C:\Program Files (x86)\Microsoft Visual Studio\2017\BuildTools\MSBuild** and open PowerShell there. Then execute the following command:

`MSBuild.exe Hello.csproj /t:"Hello"`

You should see the **MSBuild** says hello! message. The `/t:` parameter is the name of the target you want to run.

You can think about a `Target` as a method or function. Target is a list of `Tasks` that are executed one by one in order. In the example above we created one target named Hello. It contains one `Task` named `Message`. The `Message` is a predefined task. There is plenty of other predefined tasks that you can use. You can also implement custom tasks using C#.

### Property

Think of a `Property` as a string variable. Each property has to be inside `PropertyGroup`. You can have one or more `PropertyGroup` and one or more `Property` inside single group. The `PropertyGroup` is just a separator. Let's add one to our **Hello.csproj** file:

``` xml
<Project>
  <PropertyGroup>
    <HelloMessage>MSBuild says hello from property</HelloMessage>
  </PropertyGroup>
  <Target Name="Hello">
    <Message Text="$(HelloMessage)" />
  </Target>
</Project>
```

Now our `Hello` target displays message that is stored in `HelloMessage` property. We access that property using `$(<PropertyName>)`. There is also the additional advantage of using properties. You can set it's value in command line like this:

`MSBuild.exe Hello.csproj /t:"Hello" /p:HelloMessage="New Message"`

The value from command line will be used instead of the one from **Hello.csproj** file.

### Item

Think of an `Item` as a list of objects. Each object can have `Metadata` and has to be inside `ItemGroup`. Similar like for `PropertyGroup` the `ItemGroup` is just a container. Thanks to this, you can have property and item with the same name. Let's update our *Hello.csproj*:

``` xml
<Project>
  <PropertyGroup>
    <HelloMessage>MSBuild says hello from property</HelloMessage>
    <FilesToList>C:\\*.*</FilesToList>
  </PropertyGroup>
  <ItemGroup>
    <FilesToList Include="$(FilesToList)" />
  </ItemGroup>
  <Target Name="Hello">
    <Message Text="$(HelloMessage)" />
    <Message Text="@(FilesToList)" />
    <Message Text="%(FilesToList.FullPath)" />
  </Target>
</Project>
```

We added `FilesToList` property and we added `FilesToList` item. The name is the same however those are different things. We can access property with `$` and item with `@` or `%` if we want to get metadata. We also updated our `Hello` target. Now it displays two additional messages. The first one is the `FilesToList` item accessed with `@` and it displays a list of filenames joined by `;`. The second message is the same `FilesToList` item accessed with `%`. This time we get a list of `FullPath` of each file separated by new line.

You can find a list of Well-known Item Metadata [here](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-well-known-item-metadata?view=vs-2022). You can also add your own `Metadata`.

### DependsOnTargets

Let's add the second target to our example:

``` xml
<Project>
  <PropertyGroup>
    <HelloMessage>MSBuild says hello from property</HelloMessage>
    <FilesToList>C:\\*.*</FilesToList>
  </PropertyGroup>
  <Target Name="Hello" DependsOnTargets="PrepareFilesToList">
    <Message Text="$(HelloMessage)" />
    <Message Text="@(FilesToList)" />
    <Message Text="%(FilesToList.FullPath)" />
  </Target>
  <Target Name="PrepareFilesToList">
    <ItemGroup>
      <FilesToList Include="$(FilesToList)" />
    </ItemGroup>
  </Target>
</Project>
```

The `DependsOnTargets` attribute in our `Hello` target, tells MSBuild that `Hello` target depends on `PrepareFilesToList` target. MSBuild runs the `PrepareFilesToList` before `Hello` target. Inside the `PrepareFilesToList` we create `FilesToList` item and then it can be used by `Hello` target.

We can depend on more than one target if we want:

``` xml
<Project>
  <PropertyGroup>
    <HelloMessage>MSBuild says hello from property</HelloMessage>
    <FilesToList>C:\\*.*</FilesToList>
  </PropertyGroup>
  <Target Name="Hello" DependsOnTargets="SayHello;PrepareFilesToList">
    <Message Text="$(HelloMessage)" />
    <Message Text="@(FilesToList)" />
    <Message Text="%(FilesToList.FullPath)" />
  </Target>
  <Target Name="PrepareFilesToList">
    <ItemGroup>
      <FilesToList Include="$(FilesToList)" />
    </ItemGroup>
  </Target>
  <Target Name="SayHello">
    <Message Text="$(HelloMessage)" />
  </Target>
</Project>
```

To separate list of targets use `;`. 

### BeforeTargets and AfterTargets

MSBuild 4.0 introduced two new attributes: `BeforeTargets` and `AfterTargets`. You can use them instead of `DependsOnTargets`:

``` xml
<Project>
  <PropertyGroup>
    <HelloMessage>MSBuild says hello from property</HelloMessage>
    <FilesToList>C:\\*.*</FilesToList>
  </PropertyGroup>
  <Target Name="Hello">
    <Message Text="$(HelloMessage)" />
    <Message Text="@(FilesToList)" />
    <Message Text="%(FilesToList.FullPath)" />
  </Target>
  <Target Name="PrepareFilesToList" BeforeTargets="Hello">
    <ItemGroup>
      <FilesToList Include="$(FilesToList)" />
    </ItemGroup>
  </Target>
  <Target Name="SayHello" BeforeTargets="Hello">
    <Message Text="$(HelloMessage)" />
  </Target>
</Project>
```

`Hello` does not depend directly on other targets, however, you directed MSBuild to execute `PrepareFilesToList` and `SayHello` targets before `Hello` target. Sometimes it's easier to use these two attributes to plug your code into an existing pipeline.

You can read more details about the order in which targets are run [here](https://learn.microsoft.com/en-us/visualstudio/msbuild/target-build-order?view=vs-2022).

### Conditions

You can use conditions to execute some parts of code only if something is true:

``` xml
<Project>
  <PropertyGroup>
    <HelloMessage Condition="$(HelloMessage) == ''">MSBuild says hello from property</HelloMessage>
    <FilesToList>C:\\*.*</FilesToList>
  </PropertyGroup>
  <Target Name="Hello">
    <Message Text="$(HelloMessage)" />
    <Message Text="@(FilesToList)" />
    <Message Text="%(FilesToList.FullPath)" />
  </Target>
  <Target Name="PrepareFilesToList" BeforeTargets="Hello">
    <ItemGroup>
      <FilesToList Include="$(FilesToList)" />
    </ItemGroup>
  </Target>
  <Target Name="SayHello" BeforeTargets="Hello">
    <Message Text="$(HelloMessage)" />
  </Target>
</Project>
```

When `HelloMessage` is empty then set it to our initial message. You can use conditions on properties, items and targets (and some others).

### Examine Class Library project

Below you can see the content of a new Class Library project that I created in VisualStudio 2017:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" Condition="Exists('$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props')" />
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    <ProjectGuid>edcbf0e5-7fe5-4ac2-af45-979f949d03db</ProjectGuid>
    <OutputType>Library</OutputType>
    <AppDesignerFolder>Properties</AppDesignerFolder>
    <RootNamespace>ClassLibrary1</RootNamespace>
    <AssemblyName>ClassLibrary1</AssemblyName>
    <TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>
    <FileAlignment>512</FileAlignment>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">
    <DebugSymbols>true</DebugSymbols>
    <DebugType>full</DebugType>
    <Optimize>false</Optimize>
    <OutputPath>bin\Debug\</OutputPath>
    <DefineConstants>DEBUG;TRACE</DefineConstants>
    <ErrorReport>prompt</ErrorReport>
    <WarningLevel>4</WarningLevel>
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|AnyCPU' ">
    <DebugType>pdbonly</DebugType>
    <Optimize>true</Optimize>
    <OutputPath>bin\Release\</OutputPath>
    <DefineConstants>TRACE</DefineConstants>
    <ErrorReport>prompt</ErrorReport>
    <WarningLevel>4</WarningLevel>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="System"/>
    
    <Reference Include="System.Core"/>
    <Reference Include="System.Xml.Linq"/>
    <Reference Include="System.Data.DataSetExtensions"/>
    
    
    <Reference Include="Microsoft.CSharp"/>
    
    <Reference Include="System.Data"/>
    
    <Reference Include="System.Net.Http"/>
    
    <Reference Include="System.Xml"/>
  </ItemGroup>
  <ItemGroup>
    <Compile Include="Class1.cs" />
    <Compile Include="Properties\AssemblyInfo.cs" />
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
 </Project>
 ```

 We have three `PropertyGroup` and two `ItemGroup`. In the first `PropertyGroup` there is `Configuration` property. It has Condition attribute that directs MSBuild to set it's value to Debug only if it's empty. You can set configuration in Visual Studio when you do a build and in that case, Visual Studio will pass correct value to the `Configuration` property. The other two `PropertyGroup` also has conditions. This time, however, whole `PropertyGroup` will be executed or not. Have you noticed how the condition is concatenated to use two parameters?

 ``` xml
 <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">
 ```

 It is similar to [interpolated string](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated) in C#

We also have two ItemGroup. We could, of course, put everything into single ItemGroup. It just separates two different lists. The first ItemGroup adds all project references into Reference list. The second one adds all files into Compile list. MSBuild will then use those properties and items to build your project.

### Imports

When you execute `Clean`, `Build` or `Rebuild` from Visual Studio you actually execute targets with the same name that are defined in **Microsoft.Common.targets** file. In the project above, there is `Import` line at the end. It loads **Microsoft.CSharp.targets** file that is located in the path stored in a `MSBuildToolsPath` property. This file contains bunch of other imports inside and **Microsoft.Common.targets** is between them.

Import is one of a few ways how you can extend MSBuild with your custom code. Read my next article in the series to find out more about this.

### Advanced topics

This article only describes the basics of MSBuild. The best way to learn more is to read [official documentation](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild?view=vs-2022), read **Microsoft.Common.targets** and other target files and read [this book](http://msbuildbook.com/).

### I want more

To learn more, read my [next article]({{ site.baseurl }}{% post_url 2018-10-12-msbuild-extension-points %}) in the series where I described msbuild extension points.